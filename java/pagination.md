## Pagination

Most list methods in the {{ gcp_name }} Java library return a `PagedResponse` object. This allows you to iterate over results without manually managing page tokens.

The easiest way to handle pagination is to use the `iterateAll()` method. The library automatically fetches new pages in the background as you iterate through the collection.

```java
import com.google.cloud.secretmanager.v1.ListSecretsRequest;
import com.google.cloud.secretmanager.v1.ProjectName;
import com.google.cloud.secretmanager.v1.Secret;
import com.google.cloud.secretmanager.v1.SecretManagerServiceClient;

try (SecretManagerServiceClient secretManager = SecretManagerServiceClient.create()) {
    // Prepare the request
    ListSecretsRequest request = ListSecretsRequest.newBuilder()
        .setParent(ProjectName.of("my-project").toString())
        .build();

    // Call the API
    // This returns a PagedResponse
    SecretManagerServiceClient.ListSecretsPagedResponse response = secretManager.listSecrets(request);

    // Automatically fetches subsequent pages of secrets
    for (Secret secret : response.iterateAll()) {
        System.out.printf("Secret: %s%n", secret.getName());
    }
}
```

### Manual Pagination (Accessing Tokens)

If you need to control pagination manually (e.g., for a web API that sends tokens to a frontend), you can access the `nextPageToken` from the response.

```java
// Prepare request with page size and optional token
ListSecretsRequest.Builder requestBuilder = ListSecretsRequest.newBuilder()
    .setParent(ProjectName.of("my-project").toString())
    .setPageSize(10);

// Check if we have a token from a previous request (e.g., from a query parameter)
String pageToken = request.getParameter("page_token");
if (pageToken != null) {
    requestBuilder.setPageToken(pageToken);
}

SecretManagerServiceClient.ListSecretsPagedResponse response = secretManager.listSecrets(requestBuilder.build());

// Process current page items
for (Secret secret : response.getPage().getValues()) {
    // Process current page items
}

// Get the token for the next page (empty string if no more pages)
String nextToken = response.getNextPageToken();
```
