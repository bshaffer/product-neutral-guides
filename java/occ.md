# Optimistic Concurrency Control (OCC) Loop

Optimistic Concurrency Control (OCC) is a strategy used to manage shared resources and prevent "lost updates" or race conditions when multiple users or processes attempt to modify the same resource simultaneously.

As an example, consider systems like {{ gcp_name }} {{ iam_name_short }}, where the shared resource is an **{{ iam_name_short }} Policy** applied to a resource (like a Project, Bucket, or Service). To implement OCC, systems typically use a version number or an `etag` (entity tag) field on the resource object.

## Introduction to OCC

Imagine two processes, A and B, try to update a shared resource at the same time:

1. Process **A** reads the current state of the resource.

2. Process **B** reads the *same* current state.

3. Process **A** modifies its copy and writes it back to the server.

4. Process **B** modifies its copy and writes it back to the server.

Because Process **B** overwrites the resource *without* knowing that Process **A** already changed it, Process **A**'s updates are **lost**.

OCC solves this by introducing a unique fingerprint which changes every time an entity is modified. In many systems (like {{ iam_name_short }}), this is done using an `etag`. The server checks this tag on every write:

1. When you read the resource, the server returns an `etag` (a unique fingerprint).

2. When you send the modified resource back, you must include the original `etag`.

3. If the server finds that the stored `etag` does **not** match the `etag` you sent (meaning someone else modified the resource since you read it), the write operation fails with an `ABORTED` or `FAILED_PRECONDITION` error.

This failure forces the client to **retry** the entire process—re-read the *new* state, re-apply the changes, and try the write again with the new `etag`.

## Implementing the OCC Loop

The core of the OCC implementation is a `while` loop that handles the retry logic. You should set a reasonable maximum number of retries to prevent infinite loops in cases of high contention.

### Steps of the Loop:

| **Step** | **Action** | **Implementation Example** |
| --- | --- | --- |
| **Read** | Fetch the current resource state, including the `etag`. | `Policy policy = client.getIamPolicy(resourceName);` |
| **Modify** | Apply the desired changes to the local object. | `policy = policy.toBuilder().addBinding(newBinding).build();` |
| **Write/Check** | Attempt to save the modified resource using the old `etag`. This action must be inside a `try` block. | `try { client.setIamPolicy(resourceName, policy); return policy; } catch (AbortedException e) { // retry loop }` |
| **Success/Retry** | If the write succeeds, exit the loop. If it fails with a concurrency error, increment the retry counter and continue the loop (go back to the Read step). |  |

The following file provides a runnable example of how to implement the OCC loop using an {{ iam_name_short }} policy on a Project resource as the target.

*Note: This example uses the `google-cloud-resourcemanager` library, but the same OCC pattern applies to any service or database that implements versioned updates.*

### Installation

To use this example, add the following dependency to your `pom.xml`:

```xml
<dependency>
  <groupId>com.google.cloud</groupId>
  <artifactId>google-cloud-resourcemanager</artifactId>
  <version>1.45.0</version>
</dependency>
```

### Example

```java
import com.google.api.gax.rpc.AbortedException;
import com.google.api.gax.rpc.FailedPreconditionException;
import com.google.cloud.resourcemanager.v3.ProjectName;
import com.google.cloud.resourcemanager.v3.ProjectsClient;
import com.google.iam.v1.Binding;
import com.google.iam.v1.GetIamPolicyRequest;
import com.google.iam.v1.Policy;
import com.google.iam.v1.SetIamPolicyRequest;
import java.util.ArrayList;
import java.util.List;

public class IamOccExample {

    /**
     * Executes an Optimistic Concurrency Control (OCC) loop to safely update a resource.
     *
     * This method demonstrates the core Read-Modify-Write-Retry pattern.
     *
     * @param projectId  The {{ gcp_name }} Project ID (e.g., "my-project-123").
     * @param role       The {{ iam_name_short }} role to grant (e.g., "roles/storage.objectAdmin").
     * @param member     The member to add (e.g., "user:user@example.com").
     * @param maxRetries The maximum number of times to retry the update.
     * @return The successfully updated {{ iam_name_short }} policy (or null on failure).
     */
    public static Policy updateIamPolicyWithOcc(
            String projectId,
            String role,
            String member,
            int maxRetries
    ) throws Exception {
        // Setup Client
        try (ProjectsClient projectsClient = ProjectsClient.create()) {
            String projectName = ProjectName.of(projectId).toString();
            int retries = 0;

            // START OCC LOOP (Read-Modify-Write-Retry)
            while (retries < maxRetries) {
                try {
                    // READ: Get the current policy. This includes the current etag.
                    System.out.printf("Attempt %d: Reading current {{ iam_name_short }} policy for %s...%n", retries, projectName);
                    GetIamPolicyRequest getIamPolicyRequest = GetIamPolicyRequest.newBuilder()
                            .setResource(projectName)
                            .build();
                    Policy policy = projectsClient.getIamPolicy(getIamPolicyRequest);

                    // MODIFY: Apply the desired changes to the local Policy object.
                    List<Binding> bindings = new ArrayList<>(policy.getBindingsList());
                    Binding targetBinding = null;
                    int bindingIndex = -1;

                    for (int i = 0; i < bindings.size(); i++) {
                        if (bindings.get(i).getRole().equals(role)) {
                            targetBinding = bindings.get(i);
                            bindingIndex = i;
                            break;
                        }
                    }

                    if (targetBinding != null) {
                        if (targetBinding.getMembersList().contains(member)) {
                            System.out.printf("Policy for role %s and member %s exists already!%n", role, member);
                            return policy;
                        }
                        // Create a new binding based on existing one to add the member
                        Binding updatedBinding = targetBinding.toBuilder()
                                .addMembers(member)
                                .build();
                        bindings.set(bindingIndex, updatedBinding);
                    } else {
                        // Role not found, create a new binding
                        Binding newBinding = Binding.newBuilder()
                                .setRole(role)
                                .addMembers(member)
                                .build();
                        bindings.add(newBinding);
                    }

                    // The policy builder now contains the modified bindings AND the original etag.
                    Policy updatedPolicy = policy.toBuilder()
                            .clearBindings()
                            .addAllBindings(bindings)
                            .build();

                    // WRITE/CHECK: Attempt to write the modified policy.
                    System.out.printf("Attempt %d: Setting modified {{ iam_name_short }} policy...%n", retries);
                    SetIamPolicyRequest setIamPolicyRequest = SetIamPolicyRequest.newBuilder()
                            .setResource(projectName)
                            .setPolicy(updatedPolicy)
                            .build();
                    Policy resultPolicy = projectsClient.setIamPolicy(setIamPolicyRequest);

                    // SUCCESS: If the call succeeds, return the new policy and exit the loop.
                    System.out.printf("Successfully updated {{ iam_name_short }} policy in attempt %d.%n", retries);
                    return resultPolicy;

                } catch (AbortedException | FailedPreconditionException e) {
                    // If the etag is stale (concurrency conflict), this will throw a retryable exception.
                    retries++;
                    System.out.printf("Concurrency conflict detected (etag mismatch). Retrying... (%d/%d)%n",
                            retries, maxRetries);

                    // Exponential backoff (100ms * retry count)
                    Thread.sleep(100L * retries);
                }
            }
            // END OCC LOOP
        }

        System.out.printf("Failed to update {{ iam_name_short }} policy after %d attempts due to persistent concurrency conflicts.%n", maxRetries);
        return null;
    }
}
```