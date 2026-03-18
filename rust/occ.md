# Optimistic Concurrency Control (OCC) Loop

Optimistic Concurrency Control (OCC) is a strategy used to manage shared resources and prevent "lost updates" or race conditions when multiple users or processes attempt to modify the same resource simultaneously.

As an example, consider systems like Google Cloud IAM, where the shared resource is an **IAM Policy** applied to a resource (like a Project, Bucket, or Service). To implement OCC, systems typically use a version number or an `etag` (entity tag) field on the resource struct.

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

The core of the OCC implementation is a loop that handles the retry logic. You should set a reasonable maximum number of retries to prevent infinite loops in cases of high contention.

### Steps of the Loop:

| **Step** | **Action** | **Implementation Example** |
| --- | --- | --- |
| **Read** | Fetch the current resource state, including the `etag`. | `let mut policy = client.get_iam_policy(request).await?;` |
| **Modify** | Apply the desired changes to the local struct. | `policy.bindings.push(new_binding);` |
| **Write/Check** | Attempt to save the modified resource using the old `etag`. This action is checked for specific error codes. | `match client.set_iam_policy(request).await { Ok(p) => return Ok(p), Err(e) => { /* retry logic */ } }` |
| **Success/Retry** | If the write succeeds, exit the loop. If it fails with a concurrency error, increment the retry counter and continue the loop (go back to the Read step). |  |

The following code provides an example of how to implement the OCC loop using an IAM policy on a Project resource as the target.

*Note: This example assumes the use of a Cloud Resource Manager client, but the same OCC pattern applies to any service or database that implements versioned updates.*

### Example

To use this in your project, add the following to your `Cargo.toml`:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
google-cloud-resourcemanager = "0.10.0" # Example crate
tonic = "0.10"
```

```rust
use google_cloud_resourcemanager::v3::projects_client::ProjectsClient;
use google_cloud_resourcemanager::v3::{GetIamPolicyRequest, SetIamPolicyRequest, Policy, Binding};
use tonic::{Status, Code};
use std::time::Duration;
use tokio::time::sleep;

/**
 * Executes an Optimistic Concurrency Control (OCC) loop to safely update a resource.
 *
 * This function demonstrates the core Read-Modify-Write-Retry pattern.
 *
 * @param project_id The Google Cloud Project ID (e.g., "my-project-123").
 * @param role The IAM role to grant (e.g., "roles/storage.objectAdmin").
 * @param member The member to add (e.g., "user:user@example.com").
 * @param max_retries The maximum number of times to retry the update.
 * @return The successfully updated IAM policy.
 */
async fn update_iam_policy_with_occ(
    project_id: &str,
    role: &str,
    member: &str,
    max_retries: u32,
) -> Result<Policy, Box<dyn std::error::Error>> {
    // Setup Client
    let mut client = ProjectsClient::connect("https://cloudresourcemanager.googleapis.com").await?;
    let resource_name = format!("projects/{}", project_id);

    // START OCC LOOP (Read-Modify-Write-Retry)
    for attempt in 0..max_retries {
        // READ: Get the current policy. This includes the current etag.
        println!("Attempt {}: Reading current IAM policy for {}...", attempt, resource_name);

        let get_request = GetIamPolicyRequest {
            resource: resource_name.clone(),
            options: None,
        };

        let mut policy = client.get_iam_policy(get_request).await?.into_inner();

        // MODIFY: Apply the desired changes to the local Policy struct.
        let mut already_exists = false;
        let mut role_found = false;

        for binding in policy.bindings.iter_mut() {
            if binding.role == role {
                role_found = true;
                if binding.members.contains(&member.to_string()) {
                    println!("Policy for role {} and member {} exists already!", role, member);
                    already_exists = true;
                    break;
                }
                binding.members.push(member.to_string());
            }
        }

        if already_exists {
            return Ok(policy);
        }

        if !role_found {
            policy.bindings.push(Binding {
                role: role.to_string(),
                members: vec![member.to_string()],
                condition: None,
            });
        }

        // The policy struct now contains the modified bindings AND the original etag.
        // WRITE/CHECK: Attempt to write the modified policy.
        println!("Attempt {}: Setting modified IAM policy...", attempt);

        let set_request = SetIamPolicyRequest {
            resource: resource_name.clone(),
            policy: Some(policy),
            update_mask: None,
        };

        match client.set_iam_policy(set_request).await {
            Ok(response) => {
                // SUCCESS: If the call succeeds, return the new policy and exit the loop.
                println!("Successfully updated IAM policy in attempt {}.", attempt);
                return Ok(response.into_inner());
            }
            Err(status) if status.code() == Code::Aborted || status.code() == Code::FailedPrecondition => {
                // If the etag is stale (concurrency conflict), handle the retryable error.
                println!(
                    "Concurrency conflict detected (etag mismatch). Retrying... ({}/{})",
                    attempt + 1,
                    max_retries
                );
                // Exponential backoff (100ms * attempt count)
                sleep(Duration::from_millis(100 * (attempt as u64 + 1))).await;
            }
            Err(e) => return Err(Box::new(e)),
        }
    }
    // END OCC LOOP

    Err(format!("Failed to update IAM policy after {} attempts due to persistent concurrency conflicts.", max_retries).into())
}
```