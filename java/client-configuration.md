# Configuring Client Options for {{ gcp_name }} Java

The {{ gcp_name }} Java Client Libraries (built on `gax-java` and `google-cloud-core`) allow you to configure client behavior using a Settings class and its associated Builder. Each service client has a corresponding settings class (e.g., `PubSubSettings` for `TopicAdminClient` or `SubscriptionAdminClient`).

## 1. Customizing the API Endpoint

You can modify the API endpoint to connect to a specific {{ gcp_name }} region (to reduce latency or meet data residency requirements) or to a private endpoint (via Private Service Connect).

Some services, like Pub/Sub and Spanner, offer **regional endpoints**:

```java
import com.google.cloud.pubsub.v1.TopicAdminSettings;

TopicAdminSettings topicAdminSettings = TopicAdminSettings.newBuilder()
    // Connect explicitly to the us-east1 region
    .setEndpoint("us-east1-pubsub.googleapis.com:443")
    .build();
```

## 2. Authentication Configuration

While the client attempts to find [Application Default Credentials][adc] automatically, you can explicitly provide them using a `CredentialsProvider`. See [Authentication][authentication] for details and examples.

[adc]: https://cloud.google.com/docs/authentication/application-default-credentials
[authentication]: https://cloud.google.com/java/docs/setup#auth

## 3. Logging

The Java client libraries use standard logging frameworks. Most gRPC-based clients use `java.util.logging` (JUL) or can be bridged to SLF4J. You can configure logging levels via a `logging.properties` file or by programmatically adjusting the logger levels.

## 4. Configuring a Proxy

The configuration method depends on whether you are using the gRPC (default) or REST transport.

### Proxy with gRPC

When using the gRPC transport, the client library respects the [standard environment variables](https://grpc.github.io/grpc/core/md_doc_environment_variables.html). You do not need to configure this in the Java code itself.

Set the following environment variables in your shell or container environment:

```
export http_proxy="http://proxy.example.com:3128"
export https_proxy="http://proxy.example.com:3128"
```

**Handling Self-Signed Certificates (gRPC):** If your proxy uses a self-signed certificate (Deep Packet Inspection), you must provide the path to the proxy's CA certificate bundle.

```
# Point gRPC to a CA bundle that includes your proxy's certificate
export GRPC_DEFAULT_SSL_ROOTS_FILE_PATH="/path/to/roots.pem"
```

### Proxy with REST

If you are using the REST transport (via `HttpJson` transport providers), you can configure the proxy by providing a custom `HttpTransportOptions`.

```java
import com.google.api.gax.httpjson.HttpJsonTransportOptions;
import com.google.cloud.secretmanager.v1.SecretManagerServiceSettings;
import java.net.InetSocketAddress;
import java.net.Proxy;

Proxy proxy = new Proxy(Proxy.Type.HTTP, new InetSocketAddress("proxy.example.com", 3128));
HttpJsonTransportOptions transportOptions = HttpJsonTransportOptions.newBuilder()
    .setHttpTransportFactory(() -> {
        // Custom transport factory configuration
        return new com.google.api.client.http.javanet.NetHttpTransport.Builder()
            .setProxy(proxy)
            .build();
    })
    .build();

SecretManagerServiceSettings settings = SecretManagerServiceSettings.newHttpJsonBuilder()
    .setTransportChannelProvider(
        SecretManagerServiceSettings.defaultHttpJsonTransportProviderBuilder()
            .setHttpTransportOptions(transportOptions)
            .build())
    .build();
```

## 5. Configuring Retries and Timeouts

There are two ways to configure retries and timeouts: global client configuration via Settings and per-call configuration.

### Global Configuration (Recommended)

In Java, you configure retries by accessing the `UnaryCallSettings` for a specific RPC method within the Settings Builder.

#### Available RetrySettings Methods

When configuring a `RetrySettings` object, you use the following methods to fine-tune the exponential backoff strategy:

| Method | Description |
| ----- | ----- |
| `setInitialRetryDelayDuration` | Wait time before the first retry. |
| `setRetryDelayMultiplier` | Multiplier applied to the delay after each failure (e.g., `2.0`). |
| `setMaxRetryDelayDuration` | The maximum wait time between any two retries. |
| `setInitialRpcTimeoutDuration` | The timeout for the first RPC attempt. |
| `setRpcTimeoutMultiplier` | Multiplier applied to the RPC timeout for each retry. |
| `setMaxRpcTimeoutDuration` | The maximum timeout for any single RPC attempt. |
| `setTotalTimeoutDuration` | Total time allowed for the request (including all retries) before giving up. |

#### Example: Advanced Backoff

```java
import com.google.api.gax.retrying.RetrySettings;
import com.google.cloud.secretmanager.v1.SecretManagerServiceSettings;
import org.threeten.bp.Duration;

// Advanced Retry Configuration
RetrySettings retrySettings = RetrySettings.newBuilder()
    .setInitialRetryDelayDuration(Duration.ofMillis(500))  // Start with 0.5s wait
    .setRetryDelayMultiplier(2.0)                         // Double the wait each time
    .setMaxRetryDelayDuration(Duration.ofMillis(5000))    // Cap wait at 5s
    .setTotalTimeoutDuration(Duration.ofMillis(15000))    // Max 15s total
    .build();

SecretManagerServiceSettings.Builder settingsBuilder = SecretManagerServiceSettings.newBuilder();
settingsBuilder.accessSecretVersionSettings()
    .setRetrySettings(retrySettings);

SecretManagerServiceSettings settings = settingsBuilder.build();
```

### Disabling Retries

To disable retries globally for a specific method, you can set the retryable codes to an empty set.

```java
import com.google.cloud.pubsub.v1.TopicAdminSettings;
import java.util.HashSet;

TopicAdminSettings.Builder topicAdminSettingsBuilder = TopicAdminSettings.newBuilder();
topicAdminSettingsBuilder.createTopicSettings()
    .setRetryableCodes(new HashSet<>()); // Disable retries by providing no retryable status codes

TopicAdminSettings settings = topicAdminSettingsBuilder.build();
```

## 6. Logging

You can use the standard Java logging mechanisms to debug request headers, status codes, and payloads. For gRPC, you can enable verbose logging for the `io.grpc` package.

```java
import java.util.logging.ConsoleHandler;
import java.util.logging.Level;
import java.util.logging.Logger;

Logger logger = Logger.getLogger("io.grpc");
logger.setLevel(Level.FINEST);
ConsoleHandler handler = new ConsoleHandler();
handler.setLevel(Level.FINEST);
logger.addHandler(handler);
```

## 7. Other Common Configuration Options

The following options are typically configured via the `Settings.Builder` class for any client.

| Setting Method | Description |
| ----- | ----- |
| `setCredentialsProvider` | Provides the credentials (Service Account, API Key, etc.) for authentication. |
| `setEndpoint` | The address of the API remote host. Used for Regional Endpoints (e.g., `us-central1-pubsub.googleapis.com:443`) or Private Service Connect. |
| `setTransportChannelProvider` | Specifies the transport type (gRPC or HTTP/REST) and manages connection pools. |
| `setHeaderProvider` | Allows adding custom headers to every request made by the client. |
| `setUniverseDomain` | Overrides the default service domain (defaults to `googleapis.com`) for Cloud Universe support. |
| `setQuotaProjectId` | Sets the project ID used for quota and billing, which may be different from the project being operated on. |