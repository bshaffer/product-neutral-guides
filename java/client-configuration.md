# Common Configuration Options for {{ gcp_name }} Java

The following options are typically configured via the `Settings.newBuilder()`  method any client.

| Setting Method | Description |
| ----- | ----- |
| `setCredentialsProvider` | Provides the credentials (Service Account, API Key, etc.) for authentication. |
| `setEndpoint` | The address of the API remote host. Used for Regional Endpoints (e.g., `us-central1-pubsub.googleapis.com:443`) or Private Service Connect. |
| `setTransportChannelProvider` | Specifies the transport type (gRPC or HTTP/REST) and manages connection pools. |
| `setHeaderProvider` | Allows adding custom headers to every request made by the client. |
| `setUniverseDomain` | Overrides the default service domain (defaults to `googleapis.com`) for Cloud Universe support. |
| `setQuotaProjectId` | Sets the project ID used for quota and billing, which may be different from the project being operated on. |

```java
// The project that will be billed and have its quota consumed for these API calls
String billingProjectId = "my-central-billing-project";
CloudTasksSettings cloudTasksSettings =
        .setQuotaProjectId(billingProjectId)
        .setTransportChannelProvider(transportChannelProvider)
        .build();
CloudTasksClient cloudTasksClient = CloudTasksClient.create(cloudTasksSettings);
```

## 1. Customizing the API Endpoint

See [Configure cloud client library endpoints](/java/docs/endpoints).

## 2. Authentication Configuration

See [Authenticate your requests](/java/docs/authentication).

## 3. Logging

See [Troubleshooting: Logging](java/docs/troubleshooting#logging).

## 4. Configuring a Proxy

See [Configure a proxy](/java/docs/configuring-a-proxy).

## 5. Configuring Retries and Timeouts

See [Configure client-side retries](/java/docs/client-retries).
