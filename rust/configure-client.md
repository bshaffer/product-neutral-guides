{% extends "rust/_base.html" %}
{% block page_title %}How to configure a client{% endblock %}
{% block custom_heading %}
<meta name="description" content="Learn about the features and benefits of {{product_name}}.">
<meta name="keywords" value="docType:Concept">
{% endblock %}

{% block body %}

The {{ gcp_name }} Rust Client Libraries allow you to configure client behavior
using a configuration object passed to the client constructor. This configuration
is typically handled by a `ClientConfig` or `Config` struct provided by the
specific service crate.

## 1. Customizing the API Endpoint

See [Override the default endpoint](/rust/override-default-endpoint).

## 2. Authentication Configuration

While the client attempts to find [Application Default Credentials (ADC)][adc]
automatically, you can explicitly provide them using the `with_auth` or
`with_api_key` methods on the configuration object. See
[`Override the default authentication method`][authentication] for details and
examples.

[adc]: https://cloud.google.com/docs/authentication/application-default-credentials
[authentication]: /rust/override-default-authentication

## 3. Logging

Logging is handled through the `tracing` ecosystem. You can configure a
subscriber to capture logs and traces from the client libraries.
See [Troubleshooting](/rust/troubleshooting) for a comprehensive guide.

## 3. Configuring a Proxy

The configuration method depends on whether you are using a gRPC or REST-based
transport.

### Proxy with gRPC

When using the gRPC transport (standard for most services), the client library
respects the
[standard environment variables](https://grpc.github.io/grpc/core/md_doc_environment_variables.html).
You don't need to configure this in the Rust code itself.

Set the following environment variables in your shell or container:

```bash
export http_proxy="http://proxy.example.com:3128"
export https_proxy="http://proxy.example.com:3128"
```

**Handling Self-Signed Certificates (gRPC):** If your proxy uses a self-signed
certificate (Deep Packet Inspection), you cannot "ignore" verification in gRPC.
You must provide the path to the proxy's CA certificate bundle.

```bash
# Point gRPC to a CA bundle that includes your proxy's certificate
export GRPC_DEFAULT_SSL_ROOTS_FILE_PATH="/path/to/roots.pem"
```

### Proxy with REST

If you are using a library that supports REST transport, you can configure the
proxy by providing a custom `reqwest` or `hyper` client to the configuration,
depending on the specific implementation of the crate.

```rust
use google_cloud_secret_manager::v1::client::{SecretManagerClient, ClientConfig};

async fn run() -> Result<(), Box<dyn std::error::Error>> {
    // Configure a proxy using standard environment variables or a custom connector
    let proxy = reqwest::Proxy::all("http://user:password@proxy.example.com")?;
    let http_client = reqwest::Client::builder()
        .proxy(proxy)
        .build()?;

    let config = ClientConfig::default()
        .with_http_client(http_client);

    let client = SecretManagerClient::new(config).await?;
    Ok(())
}
```

## 4. Configuring Retries and Timeouts

There are two ways to configure retries and timeouts: global client
configuration and per-call configuration.

### Per-Call Configuration (Recommended)

For most use cases, it is cleaner to override settings for specific calls using
the `RetryConfig` or request options.

#### Available `RetryConfig` Parameters

When configuring a `RetryConfig` struct, you can use the following fields to
fine-tune the exponential backoff strategy:

| Field | Type | Description |
| ----- | ----- | ----- |
| `retries_enabled` | `bool` | Enables or disables retries for this call. |
| `max_retries` | `u32` | The maximum number of retry attempts. |
| `initial_retry_delay` | `Duration` | Wait time before the first retry. |
| `retry_delay_multiplier` | `f64` | Multiplier applied to the delay after each failure (e.g., `2.0`). |
| `max_retry_delay` | `Duration` | The maximum wait time between any two retries. |
| `total_timeout` | `Duration` | Total time allowed for the request (including all retries) before giving up. |

#### Example: Advanced Backoff

```rust
use std::time::Duration;
use google_cloud_gax::retry::RetryConfig;

// Advanced Retry Configuration
let retry_config = RetryConfig {
    retries_enabled: true,
    max_retries: 3,
    initial_retry_delay: Duration::from_millis(500), // Start with 0.5s wait
    retry_delay_multiplier: 2.0,                    // Double the wait each time (0.5s -> 1s -> 2s)
    max_retry_delay: Duration::from_secs(5),        // Cap wait at 5s
    total_timeout: Duration::from_secs(15),         // Max 15s total
};

client.access_secret_version(request, Some(retry_config)).await?;
```

### Disabling Retries

You can also configure retries globally by modifying the `ClientConfig` passed
to the constructor. This is useful if you want to change the default retry
strategy for all calls made by that client instance.

```rust
use google_cloud_pubsub::client::{Client, ClientConfig};

async fn run() -> Result<(), Box<dyn std::error::Error>> {
    let config = ClientConfig::default()
        // Quickly disable retries for the entire client
        .with_retry_config(None);

    let client = Client::new(config).await?;
    Ok(())
}
```

## 5. Logging

You can use the `tracing` crate to capture client logs. By initializing a
subscriber, you can debug request metadata, status codes, and events.

```rust
use tracing_subscriber;

fn main() {
    // Initialize tracing subscriber to see debug output from the client
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::DEBUG)
        .init();
}
```

## 6. Other Common Configuration Options

The following options can be passed to the configuration builder of most clients.

| Option | Type | Description |
| ----- | ----- | ----- |
| `credentials` | `Option<Credentials>` | Explicit credentials object for authentication. |
| `api_key` | `String` | An API Key for services that support public API key authentication (bypassing OAuth2). |
| `endpoint` | `String` | The address of the API remote host. Used for Regional Endpoints (e.g., `https://us-central1-pubsub.googleapis.com:443`) or {{ private_service_connect_name }}. |
| `transport` | `enum` | Specifies the transport type, typically gRPC or HTTP/JSON. |
| `retry_config` | `Option<RetryConfig>` | If `None`, disables the default retry logic for all methods in the client. |
| `universe_domain` | `String` | Overrides the default service domain (defaults to `googleapis.com`) for Cloud Universe support. |

{% endblock %}