# Optimistic Concurrency Control (OCC) Loop

Optimistic Concurrency Control (OCC) is a strategy used to manage shared resources and prevent "lost updates" or race conditions when multiple users or processes attempt to modify the same resource simultaneously.

As an example, consider systems like Google Cloud IAM, where the shared resource is an **IAM Policy** applied to a resource (like a Project, Bucket, or Service). To implement OCC, systems typically use a version number or an `etag` (entity tag) field on the resource object.

## Introduction to OCC

Imagine two processes, A and B, try to update a shared resource at the same time:

1. Process **A** reads the current state of the resource.

2. Process **B** reads the *same* current state.

3. Process **A** modifies its copy and writes it back to the server.

4. Process **B** modifies its copy and writes it back to the server.

Because Process **B** overwrites the resource *without* knowing that Process **A** already changed it, Process **A**'s updates are **lost**.

OCC solves this by introducing a unique fingerprint which changes every time an entity is modified. In many systems (like IAM), this is done using an `etag`. The server checks this tag on every write:

1. When you read the resource, the server returns an `etag` (a unique fingerprint).

2. When you send the modified resource back, you must include the original `etag`.

3. If the server finds that the stored `etag` does **not** match the `etag` you sent (meaning someone else modified the resource since you read it), the write operation fails with an `Aborted` or `FailedPrecondition` error.

This failure forces the client to **retry** the entire process—re-read the *new* state, re-apply the changes, and try the write again with the new `etag`.

## Implementing the OCC Loop

The core of the OCC implementation is a `for` loop that handles the retry logic. You should set a reasonable maximum number of retries to prevent infinite loops in cases of high contention.

### Steps of the Loop:

| **Step** | **Action** | **Implementation Example** |
| --- | --- | --- |
| **Read** | Fetch the current resource state, including the `etag`. | `policy, err := client.GetIamPolicy(ctx, req)` |
| **Modify** | Apply the desired changes to the local object. | `policy.Bindings = append(policy.Bindings, newBinding)` |
| **Write/Check** | Attempt to save the modified resource using the old `etag`. This action must involve error checking. | `policy, err = client.SetIamPolicy(ctx, &iampb.SetIamPolicyRequest{ Resource: name, Policy: policy })` |
| **Success/Retry** | If the write succeeds, exit the loop. If it fails with a concurrency error, increment the retry counter and continue the loop (go back to the Read step). |  |

The following file provides a runnable example of how to implement the OCC loop using an IAM policy on a Project resource as the target.

*Note: This example uses the `cloud.google.com/go/resourcemanager/apiv3` package, but the same OCC pattern applies to any service or database that implements versioned updates.*

### Example

```go
import (
	"context"
	"fmt"
	"time"

	resourcemanager "cloud.google.com/go/resourcemanager/apiv3"
	"cloud.google.com/go/iam/apiv1/iampb"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

/**
 * UpdateIamPolicyWithOcc executes an Optimistic Concurrency Control (OCC) loop to safely update a resource.
 *
 * This function demonstrates the core Read-Modify-Write-Retry pattern.
 *
 * @param ctx The context for the API calls.
 * @param projectId The Google Cloud Project ID (e.g., 'my-project-123').
 * @param role The IAM role to grant (e.g., 'roles/storage.objectAdmin').
 * @param member The member to add (e.g., 'user:user@example.com').
 * @param maxRetries The maximum number of times to retry the update.
 * @return *iampb.Policy The successfully updated IAM policy (or nil on failure).
 */
func UpdateIamPolicyWithOcc(
	ctx context.Context,
	projectId string,
	role string,
	member string,
	maxRetries int,
) (*iampb.Policy, error) {
	// Setup Client (Example using ResourceManager - adjust for your service)
	client, err := resourcemanager.NewProjectsClient(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed to create client: %v", err)
	}
	defer client.Close()

	projectName := fmt.Sprintf("projects/%s", projectId)

	// START OCC LOOP (Read-Modify-Write-Retry)
	for retries := 0; retries < maxRetries; retries++ {
		// READ: Get the current policy. This includes the current etag.
		fmt.Printf("Attempt %d: Reading current IAM policy for %s...\n", retries, projectName)
		getReq := &iampb.GetIamPolicyRequest{
			Resource: projectName,
		}
		policy, err := client.GetIamPolicy(ctx, getReq)
		if err != nil {
			return nil, fmt.Errorf("failed to get policy: %v", err)
		}

		// MODIFY: Apply the desired changes to the local Policy object.
		var roleExists bool
		for _, binding := range policy.Bindings {
			if binding.Role == role {
				roleExists = true
				for _, m := range binding.Members {
					if m == member {
						fmt.Printf("Policy for role %s and member %s exists already!\n", role, member)
						return policy, nil
					}
				}
				binding.Members = append(binding.Members, member)
				break
			}
		}

		if !roleExists {
			policy.Bindings = append(policy.Bindings, &iampb.Binding{
				Role:    role,
				Members: []string{member},
			})
		}

		// WRITE/CHECK: Attempt to write the modified policy.
		// The policy object now contains the modified bindings AND the original etag.
		fmt.Printf("Attempt %d: Setting modified IAM policy...\n", retries)
		setReq := &iampb.SetIamPolicyRequest{
			Resource: projectName,
			Policy:   policy,
		}
		updatedPolicy, err := client.SetIamPolicy(ctx, setReq)

		if err == nil {
			// SUCCESS: If the call succeeds, return the new policy and exit the loop.
			fmt.Printf("Successfully updated IAM policy in attempt %d.\n", retries)
			return updatedPolicy, nil
		}

		// If the etag is stale (concurrency conflict), check for retryable status codes.
		st, ok := status.FromError(err)
		if ok && (st.Code() == codes.Aborted || st.Code() == codes.FailedPrecondition) {
			fmt.Printf("Concurrency conflict detected (etag mismatch). Retrying... (%d/%d)\n", retries+1, maxRetries)
			// Exponential backoff (100ms * retry count)
			time.Sleep(time.Duration(retries+1) * 100 * time.Millisecond)
			continue
		}

		// If it's a different error, stop and return it.
		return nil, fmt.Errorf("failed to set policy: %v", err)
	}
	// END OCC LOOP

	return nil, fmt.Errorf("failed to update IAM policy after %d attempts due to persistent concurrency conflicts", maxRetries)
}
```