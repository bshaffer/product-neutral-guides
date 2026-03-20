## Update Masks

When updating resources (PATCH requests), {{ gcp_name }} APIs often use an **Update Mask** (`com.google.protobuf.FieldMask`). This tells the server *exactly* which fields you intend to update, preventing accidental overwrites of other fields.

If you do not provide a mask, some APIs update **all** fields, resetting missing ones to default values.

### Constructing a FieldMask

```java
import com.google.cloud.secretmanager.v1.Secret;
import com.google.cloud.secretmanager.v1.SecretManagerServiceClient;
import com.google.cloud.secretmanager.v1.UpdateSecretRequest;
import com.google.protobuf.FieldMask;

try (SecretManagerServiceClient client = SecretManagerServiceClient.create()) {
    // Prepare the resource with NEW values
    Secret secret = Secret.newBuilder()
        .setName("projects/my-project/secrets/my-secret")
        .putLabels("env", "production") // We only want to update this field
        .build();

    // Create the FieldMask
    // Paths MUST match the protobuf field names (snake_case)
    FieldMask updateMask = FieldMask.newBuilder()
        .addPaths("labels")
        .build();

    // Prepare the Request object
    UpdateSecretRequest request = UpdateSecretRequest.newBuilder()
        .setSecret(secret)
        .setUpdateMask(updateMask)
        .build();

    // Call the API
    client.updateSecret(request);
}
```

**Note:** The field names in `paths` should be the protocol buffer field names (usually `snake_case`), even though Java methods use `camelCase`.
