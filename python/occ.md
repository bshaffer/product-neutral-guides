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
| **Read** | Fetch the current resource state, including the `etag`. | `policy = client.get_iam_policy(request={"resource": resource_name})` |
| **Modify** | Apply the desired changes to the local object. | `policy.bindings.append(new_binding)` |
| **Write/Check** | Attempt to save the modified resource using the old `etag`. This action must be inside a `try` block. | `try: client.set_iam_policy(request={"resource": resource_name, "policy": policy}) except Aborted: # retry loop` |
| **Success/Retry** | If the write succeeds, exit the loop. If it fails with a concurrency error, increment the retry counter and continue the loop (go back to the Read step). |  |

The following file provides a runnable example of how to implement the OCC loop using an IAM policy on a Project resource as the target.

*Note: This example uses the `google-cloud-resource-manager` library, but the same OCC pattern applies to any service or database that implements versioned updates.*

### Example

```python
import time
from typing import Optional
from google.cloud import resourcemanager_v3
from google.iam.v1 import iam_policy_pb2, policy_pb2
from google.api_core import exceptions

def update_iam_policy_with_occ(
    project_id: str,
    role: str,
    member: str,
    max_retries: int = 5
) -> Optional[policy_pb2.Policy]:
    """
    Executes an Optimistic Concurrency Control (OCC) loop to safely update a resource.

    This function demonstrates the core Read-Modify-Write-Retry pattern.

    Args:
        project_id: The Google Cloud Project ID (e.g., 'my-project-123').
        role: The IAM role to grant (e.g., 'roles/storage.objectAdmin').
        member: The member to add (e.g., 'user:user@example.com').
        max_retries: The maximum number of times to retry the update.

    Returns:
        The successfully updated IAM policy (or None on failure).
    """
    # Setup Client (Example using ResourceManager - adjust for your service)
    client = resourcemanager_v3.ProjectsClient()
    project_name = f"projects/{project_id}"

    retries = 0

    # START OCC LOOP (Read-Modify-Write-Retry)
    while retries < max_retries:
        try:
            # READ: Get the current policy. This includes the current etag.
            print(f"Attempt {retries}: Reading current IAM policy for {project_name}...")
            request = iam_policy_pb2.GetIamPolicyRequest(resource=project_name)
            policy = client.get_iam_policy(request=request)

            # MODIFY: Apply the desired changes to the local Policy object.
            # Look for an existing binding for the role.
            target_binding = None
            for binding in policy.bindings:
                if binding.role == role:
                    target_binding = binding
                    break

            if target_binding:
                if member in target_binding.members:
                    print(f"Policy for role {role} and member {member} exists already!")
                    return policy
                target_binding.members.append(member)
            else:
                # Create a new binding if the role doesn't exist in the policy
                new_binding = policy_pb2.Binding(
                    role=role,
                    members=[member]
                )
                policy.bindings.append(new_binding)

            # WRITE/CHECK: Attempt to write the modified policy.
            # The policy object still contains the etag from the 'Read' step.
            print(f"Attempt {retries}: Setting modified IAM policy...")
            set_request = iam_policy_pb2.SetIamPolicyRequest(
                resource=project_name,
                policy=policy
            )
            updated_policy = client.set_iam_policy(request=set_request)

            # SUCCESS: If the call succeeds, return the new policy and exit the loop.
            print(f"Successfully updated IAM policy in attempt {retries}.")
            return updated_policy

        except (exceptions.Aborted, exceptions.FailedPrecondition) as e:
            # If the etag is stale (concurrency conflict), this will catch the retryable exception.
            retries += 1
            print(f"Concurrency conflict detected (etag mismatch). Retrying... ({retries}/{max_retries})")
            time.sleep(0.1 * retries)  # Exponential backoff (100ms * retry count)

    # END OCC LOOP

    print(f"Failed to update IAM policy after {max_retries} attempts due to persistent concurrency conflicts.")
    return None
```