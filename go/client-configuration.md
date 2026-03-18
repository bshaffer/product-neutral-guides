# Configuring Client Options for Google Cloud Go

The Google Cloud Go Client Libraries (built on `google.golang.org/api` and `github.com/googleapis/gax-go`) allow you to configure client behavior using functional options passed to the client factory function. These options are typically implementations of the [`option.ClientOption`](https://pkg.go.dev/google.golang.org/api/option) interface.

## 1. Customizing the API Endpoint

You can modify the API endpoint to connect to a specific Google Cloud region (to reduce latency or meet data residency requirements) or to a private endpoint (via Private Service Connect).

Some services, like Pub/Sub and Spanner, offer **regional endpoints**:

```go
import (
    "context"
    "cloud.google.com/go/pubsub"
    "google.golang.org/api/option"
)

ctx := context.Background()
client, err := pubsub.NewClient(ctx, "project-id",
    // Connect explicitly to the us-east1 region
    option.WithEndpoint("us-east1-pubsub.googleapis.com:443"),
)
```

## 2. Authentication Configuration

While the client attempts to find [Application Default Credentials][adc] automatically, you can explicitly provide them using options like `WithCredentialsFile` or `WithAPIKey`. See [Authentication][authentication] for details and examples.

[adc]: https://cloud.google.com/docs/authentication/application-default-credentials
[authentication]: https://cloud.google.com/go/docs/reference/help/authentication

## 3. Logging

Logging can be enabled using environment variables. For gRPC-based clients, you can use the internal gRPC logger by setting the `GRPC_GO_LOG_SEVERITY_LEVEL` and `GRPC_GO_LOG_FORMATTER` environment variables.

## 3. Configuring a Proxy

The configuration method depends on whether the client uses a gRPC or HTTP (REST) transport.

### Proxy with gRPC

When using the gRPC transport, the client library respects the standard environment variables. You do not need to configure this in the Go code itself.

Set the following environment variables in your shell or container environment:

```
export http_proxy="http://proxy.example.com:3128"
export https_proxy="http://proxy.example.com:3128"
```

**Handling Self-Signed Certificates (gRPC):** If your proxy uses a self-signed certificate (Deep Packet Inspection), you must provide the proxy's CA certificate bundle when establishing the connection.

```go
import (
    "crypto/x509"
    "os"
    "google.golang.org/api/option"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
)

cert, _ := os.ReadFile("/path/to/roots.pem")
certPool := x509.NewCertPool()
certPool.AppendCertsFromPEM(cert)
creds := credentials.NewClientTLSFromCert(certPool, "")

client, err := service.NewClient(ctx, option.WithGRPCDialOption(grpc.WithTransportCredentials(creds)))
```

### Proxy with REST

If you are using a REST-based client, you must configure the proxy by providing a custom `http.Client`. This allows you to define a custom `http.Transport` with proxy settings.

```go
import (
    "context"
    "net/http"
    "net/url"
    "cloud.google.com/go/secretmanager/apiv1"
    "google.golang.org/api/option"
)

proxyURL, _ := url.Parse("http://user:password@proxy.example.com:3128")
httpClient := &http.Client{
    Transport: &http.Transport{
        Proxy: http.ProxyURL(proxyURL),
    },
}

ctx := context.Background()
client, err := secretmanager.NewClient(ctx, option.WithHTTPClient(httpClient))
```

## 4. Configuring Retries and Timeouts

There are two ways to configure retries and timeouts: global client configuration via `CallOption` and per-call configuration using `context`.

### Per-Call Configuration (Recommended)

For most use cases, it is cleaner to manage timeouts using a `context.Context` and override retry settings using `gax.CallOption`.

#### Available Retry Settings

When using `gax.WithRetry`, you can provide a `gax.Retryer` that utilizes a `gax.Backoff` struct to fine-tune the exponential backoff strategy:

| Field | Type | Description |
| ----- | ----- | ----- |
| `Initial` | `time.Duration` | Wait time before the first retry. |
| `Max` | `time.Duration` | The maximum wait time between any two retries. |
| `Multiplier` | `float64` | Multiplier applied to the delay after each failure (e.g., `2.0`). |

#### Example: Advanced Backoff

```go
import (
    "context"
    "time"
    "github.com/googleapis/gax-go/v2"
    secretmanager "cloud.google.com/go/secretmanager/apiv1"
    secretmanagerpb "cloud.google.com/go/secretmanager/apiv1/secretmanagerpb"
)

// Define custom backoff parameters
retryOpt := gax.WithRetry(func() gax.Retryer {
    return gax.OnCodes([]codes.Code{
        codes.Unavailable,
        codes.DeadlineExceeded,
    }, gax.Backoff{
        Initial:    500 * time.Millisecond,
        Max:        5 * time.Second,
        Multiplier: 2.0,
    })
})

// Set a total timeout for the call using context
ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
defer cancel()

req := &secretmanagerpb.AccessSecretVersionRequest{Name: "name"}
resp, err := client.AccessSecretVersion(ctx, req, retryOpt)
```

### Disabling Retries

You can disable retries for a specific call by passing `gax.WithRetry` returning `nil`, or by configuring the client to use a non-retrying option.

```go
import "github.com/googleapis/gax-go/v2"

// Disable retries for this specific call
resp, err := client.AccessSecretVersion(ctx, req, gax.WithRetry(func() gax.Retryer {
    return nil
}))
```

## 5. Logging

To debug request details such as headers and payloads, you can use the `option.WithTelemetryConfiguration` or configure the transport to output to `os.Stderr`.

```go
import (
    "context"
    "cloud.google.com/go/pubsub"
    "google.golang.org/api/option"
    "logging"
)

// Go clients often use environment variables for internal library tracing
// Set GRPC_GO_LOG_SEVERITY_LEVEL=info for gRPC tracing
ctx := context.Background()
client, err := pubsub.NewClient(ctx, "project-id")
```

## 6. Other Common Configuration Options

The following options can be passed to the factory function of any Google Cloud Go client (e.g., `pubsub.NewClient`, `spanner.NewClient`, `storage.NewClient`).

| Option | Description |
| ----- | ----- |
| `option.WithCredentialsFile(path string)` | Path to a JSON credentials file. |
| `option.WithAPIKey(key string)` | An API Key for services that support public API key authentication. |
| `option.WithEndpoint(url string)` | The address of the API remote host. Used for Regional Endpoints or Private Service Connect. |
| `option.WithGRPCConn(conn *grpc.ClientConn)` | Allows providing a pre-configured gRPC connection. |
| `option.WithHTTPClient(client *http.Client)` | Allows providing a pre-configured HTTP client for REST-based services. |
| `option.WithUserAgent(ua string)` | Sets a custom User-Agent header for requests. |
| `option.WithUniverseDomain(domain string)` | Overrides the default service domain (defaults to `googleapis.com`) for Cloud Universe support. |