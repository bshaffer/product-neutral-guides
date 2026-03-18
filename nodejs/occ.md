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
| **Read** | Fetch the current resource state, including the `etag`. | `const [policy] = await client.getIamPolicy({resource: projectName});` |
| **Modify** | Apply the desired changes to the local object. | `policy.bindings.push(newBinding);` |
| **Write/Check** | Attempt to save the modified resource using the old `etag`. This action must be inside a `try/catch` block. | `try { await client.setIamPolicy({resource: projectName, policy}); return policy; } catch (err) { // retry loop }` |
| **Success/Retry** | If the write succeeds, exit the loop. If it fails with a concurrency error, increment the retry counter and continue the loop (go back to the Read step). |  |

The following file provides a runnable example of how to implement the OCC loop using an IAM policy on a Project resource as the target.

*Note: This example uses the `@google-cloud/resource-manager` package, but the same OCC pattern applies to any service or database that implements versioned updates.*

### Example

```javascript
const { ProjectsClient } = require('@google-cloud/resource-manager').v3;

/**
 * Executes an Optimistic Concurrency Control (OCC) loop to safely update a resource.
 *
 * This function demonstrates the core Read-Modify-Write-Retry pattern.
 *
 * @param {string} projectId The Google Cloud Project ID (e.g., 'my-project-123').
 * @param {string} role The IAM role to grant (e.g., 'roles/storage.objectAdmin').
 * @param {string} member The member to add (e.g., 'user:user@example.com').
 * @param {number} maxRetries The maximum number of times to retry the update.
 * @returns {Promise<object|null>} The successfully updated IAM policy (or null on failure).
 */
async function updateIamPolicyWithOcc(
    projectId,
    role,
    member,
    maxRetries = 5
) {
    // Setup Client (Example using ResourceManager - adjust for your service)
    const projectsClient = new ProjectsClient();
    const projectName = `projects/${projectId}`;

    let retries = 0;

    // START OCC LOOP (Read-Modify-Write-Retry)
    while (retries < maxRetries) {
        try {
            // READ: Get the current policy. This includes the current etag.
            console.log(`Attempt ${retries}: Reading current IAM policy for ${projectName}...`);
            const [policy] = await projectsClient.getIamPolicy({
                resource: projectName,
            });

            // MODIFY: Apply the desired changes to the local Policy object.
            let roleExists = false;
            let bindings = policy.bindings || [];

            for (const binding of bindings) {
                if (binding.role === role) {
                    roleExists = true;
                    if (binding.members.includes(member)) {
                        console.log(`Policy for role ${role} and member ${member} exists already!`);
                        return policy;
                    }
                    binding.members.push(member);
                    break;
                }
            }

            if (!roleExists) {
                bindings.push({
                    role: role,
                    members: [member],
                });
            }

            // The policy object now contains the modified bindings AND the original etag.
            policy.bindings = bindings;

            // WRITE/CHECK: Attempt to write the modified policy.
            console.log(`Attempt ${retries}: Setting modified IAM policy...`);
            const [newPolicy] = await projectsClient.setIamPolicy({
                resource: projectName,
                policy: policy,
            });

            // SUCCESS: If the call succeeds, return the new policy and exit the loop.
            console.log(`Successfully updated IAM policy in attempt ${retries}.`);
            return newPolicy;

        } catch (err) {
            // If the etag is stale (concurrency conflict), the API returns ABORTED (10) or FAILED_PRECONDITION (9).
            if (err.code === 10 || err.code === 9) {
                retries++;
                console.log(`Concurrency conflict detected (etag mismatch). Retrying... (${retries}/${maxRetries})`);

                // Exponential backoff (100ms * retry count)
                const delay = 100 * retries;
                await new Promise(resolve => setTimeout(resolve, delay));
                continue;
            }

            // If it's a different error, rethrow it.
            throw err;
        }
    }
    // END OCC LOOP

    console.log(`Failed to update IAM policy after ${maxRetries} attempts due to persistent concurrency conflicts.`);
    return null;
}
```