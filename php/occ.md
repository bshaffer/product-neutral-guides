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

3. If the server finds that the stored `etag` does **not** match the `etag` you sent (meaning someone else modified the resource since you read it), the write operation fails with an `ABORTED` or `FAILED_PRECONDITION` error.

This failure forces the client to **retry** the entire process—re-read the *new* state, re-apply the changes, and try the write again with the new `etag`.

## Implementing the OCC Loop

The core of the OCC implementation is a `while` loop that handles the retry logic. You should set a reasonable maximum number of retries to prevent infinite loops in cases of high contention.

### Steps of the Loop:

| **Step** | **Action** | **Implementation Example** |
| --- | --- | --- |
| **Read** | Fetch the current resource state, including the `etag`. | `$policy = $client->getIamPolicy($resourceName);` |
| **Modify** | Apply the desired changes to the local object. | `$policy->setBindings($updatedBindings);` |
| **Write/Check** | Attempt to save the modified resource using the old `etag`. This action must be inside a `try` block. | `try {    $client->setIamPolicy($resourceName, $policy);    return $policy; } catch (AbortedException $e) {    // retry loop}` |
| **Success/Retry** | If the write succeeds, exit the loop. If it fails with a concurrency error, increment the retry counter and continue the loop (go back to the Read step). |  |

The following file provides a runnable example of how to implement the OCC loop using an IAM policy on a Project resource as the target.

*Note: This example uses the `google/cloud-resource-manager` component, but the same OCC pattern applies to any service or database that implements versioned updates.*

### Example

```php
use Google\Cloud\Core\Exception\AbortedException;
use Google\Cloud\Core\Exception\FailedPreconditionException;
use Google\Cloud\ResourceManager\V3\Client\ProjectsClient;
use Google\Cloud\Iam\V1\GetIamPolicyRequest;
use Google\Cloud\Iam\V1\SetIamPolicyRequest;
use Google\Cloud\Iam\V1\Policy;
use Google\Cloud\Iam\V1\Binding;

/**
 * Executes an Optimistic Concurrency Control (OCC) loop to safely update a resource.
 *
 * This function demonstrates the core Read-Modify-Write-Retry pattern.
 *
 * @param string $projectId The Google Cloud Project ID (e.g., 'my-project-123').
 * @param string $role The IAM role to grant (e.g., 'roles/storage.objectAdmin').
 * @param string $member The member to add (e.g., 'user:user@example.com').
 * @param int $maxRetries The maximum number of times to retry the update.
 * @return Policy The successfully updated IAM policy (or null on failure).
 */
function update_iam_policy_with_occ(
    string $projectId,
    string $role,
    string $member,
    int $maxRetries = 5
): ?Policy {
    // Setup Client (Example using ResourceManager - adjust for your service)
    $projectsClient = new ProjectsClient();
    $projectName = ProjectsClient::projectName($projectId);

    $retries = 0;

    // --- START OCC LOOP (Read-Modify-Write-Retry) ---
    while ($retries < $maxRetries) {
        try {
            // READ: Get the current policy. This includes the current etag.
            echo "Attempt $retries: Reading current IAM policy for $projectName...\n";
            $getIamPolicyRequest = new GetIamPolicyRequest(['resource' => $projectName]);
            $policy = $projectsClient->getIamPolicy($getIamPolicyRequest);

            // MODIFY: Apply the desired changes to the local Policy object ($policy).
            $bindings = $policy->getBindings();
            $binding = new Binding(['role' => $role, 'members' => [$member]]);
            foreach ($bindings as $existingBinding) {
                if ($existingBinding->getRole() === $role) {
                    $binding = $existingBinding;
                    foreach ($binding->getMembers() as $roleMember) {
                        if ($roleMember === $member) {
                            echo "Policy for role $role and member $member exists already!\n";
                            return $policy;
                        }
                    }
                    $members = $binding->getMembers();
                    $members[] = $member;
                    $binding->setMembers($members);
                }
            }

            // The policy object now contains the modified bindings AND the original etag.
            $bindings[] = $binding;
            $policy->setBindings($bindings);

            // WRITE/CHECK: Attempt to write the modified policy.
            echo "Attempt $retries: Setting modified IAM policy...\n";
            $setIamPolicyRequest = new SetIamPolicyRequest(['resource' => $projectName, 'policy' => $policy]);
            $newPolicy = $projectsClient->setIamPolicy($setIamPolicyRequest);

            // SUCCESS: If the call succeeds, return the new policy and exit the loop.
            echo "Successfully updated IAM policy in attempt $retries.\n";
            return (object) $policy; // Mock return
        } catch (AbortedException | FailedPreconditionException $e) {
            // If the etag is stale (concurrency conflict), this will throw a retryable exception.
            $retries++;
            echo "Concurrency conflict detected (etag mismatch). Retrying... ($retries/$maxRetries)\n";
            usleep(100000 * $retries); // Exponential backoff (100ms * retry count)
        }
    }
    // --- END OCC LOOP ---

    echo "Failed to update IAM policy after $maxRetries attempts due to persistent concurrency conflicts.\n";
    return null;
}
```
