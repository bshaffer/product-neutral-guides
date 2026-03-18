# Configuring Client Options for Google Cloud Python

The Google Cloud Python Client Libraries (built on `google-api-core` and `google-cloud-core`) allow you to configure client behavior via keyword arguments passed to the client constructor or specific configuration objects. These options are primarily managed through the [`google.api_core.client_options.ClientOptions`](https://googleapis.dev/python/google-api-core/latest/client_options.html) class.

## 1. Customizing the API Endpoint

You can modify the API endpoint to connect to a specific Google Cloud region (to reduce latency or meet data residency requirements) or to a private endpoint (via Private Service Connect).

Some services, like Pub/Sub and Spanner, offer **regional endpoints**:

```python
from google.cloud import pubsub_v1
from google.api_core.client_options import ClientOptions

# Connect explicitly to the us-east1 region
options = ClientOptions(api_endpoint="us-east1-pubsub.googleapis.com:443")
publisher = pubsub_v1.PublisherClient(client_options=options)
```

## 2. Authentication Configuration

While the client attempts to find [Application Default Credentials][adc] automatically, you can explicitly provide them using the `credentials` or `client_options` arguments. See [`Authentication`][authentication.md] for details and examples.

[adc]: https://cloud.google.com/docs/authentication/application-default-credentials
[authentication.md]: https://cloud.google.com/python/docs/reference/google-cloud-core/latest/auth

## 3. Logging

Logging can be configured using the standard Python `logging` module. You can set the log level for specific library loggers to debug requests and responses. See [Troubleshooting](DEBUG.md) for a comprehensive guide.

## 4. Configuring a Proxy

The configuration method depends on whether you are using the `grpc` (default) or `rest` transport.

### Proxy with gRPC

When using the gRPC transport, the client library respects standard environment variables. You do not need to configure this in the Python code itself.

Set the following environment variables in your shell or Docker container:

```bash
export http_proxy="http://proxy.example.com:3128"
export https_proxy="http://proxy.example.com:3128"
```

**Handling Self-Signed Certificates (gRPC):** If your proxy uses a self-signed certificate (Deep Packet Inspection), you cannot simply "ignore" verification in gRPC. You must provide the path to the proxy's CA certificate bundle.

```bash
# Point gRPC to a CA bundle that includes your proxy's certificate
export GRPC_DEFAULT_SSL_ROOTS_FILE_PATH="/path/to/roots.pem"
```

### Proxy with REST

If you are using the `rest` transport, the client library typically uses the `requests` library under the hood. Like gRPC, `requests` honors the standard environment variables:

```bash
export HTTP_PROXY="http://proxy.example.com:3128"
export HTTPS_PROXY="http://proxy.example.com:3128"
```

If you need to provide a custom SSL bundle for REST calls via code, you can use the `SSL_CERT_FILE` environment variable or configure the underlying transport object.

## 5. Configuring Retries and Timeouts

There are two ways to configure retries and timeouts: global client configuration and per-call configuration (simple).

### Per-Call Configuration (Recommended)

For most use cases, it is cleaner to override settings for specific calls by passing `retry` and `timeout` arguments directly to the method.

#### Available `Retry` Parameters

When creating a `google.api_core.retry.Retry` object, you can use the following parameters to fine-tune the exponential backoff strategy:

| Parameter | Type | Description |
| ----- | ----- | ----- |
| `initial` | `float` | Wait time before the first retry (in seconds). |
| `maximum` | `float` | The maximum wait time between any two retries (in seconds). |
| `multiplier` | `float` | Multiplier applied to the delay after each failure (e.g., `2.0`). |
| `deadline` | `float` | Total time allowed for the request (including all retries) before giving up (in seconds). |
| `predicate` | `callable` | A function that determines which exceptions should be retried. |

#### Example: Advanced Backoff

```python
from google.cloud import secretmanager
from google.api_core import retry

client = secretmanager.SecretManagerServiceClient()

# Advanced Retry Configuration
custom_retry = retry.Retry(
    initial=0.5,      # Start with 0.5s wait
    multiplier=2.0,   # Double the wait each time (0.5s -> 1s -> 2s)
    maximum=5.0,      # Cap wait at 5s
    deadline=15.0     # Max 15s total
)

request = {"name": "projects/my-project/secrets/my-secret/versions/latest"}
response = client.access_secret_version(request=request, retry=custom_retry)
```

### Disabling Retries

You can disable retries for a specific call by passing `retry=None`.

```python
# Disable retries for this specific call
client.access_secret_version(request=request, retry=None)
```

## 6. Logging

You can use the standard Python `logging` module to debug request headers, status codes, and payloads.

```python
import logging
from google.cloud import pubsub_v1

# Enable debug logging for the Google Cloud libraries
logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger("google.cloud")

client = pubsub_v1.PublisherClient()
```

## 7. Other Common Configuration Options

The following options can be passed to the constructor of most Google Cloud clients or via the `client_options` argument.

| Option Key | Type | Description |
| ----- | ----- | ----- |
| `credentials` | `google.auth.credentials.Credentials` | Explicit credentials object to use for authentication. |
| `client_options` | `google.api_core.client_options.ClientOptions` | An object containing options like `api_endpoint` and `universe_domain`. |
| `transport` | `str` | Specifies the transport type. Common options: `'grpc'` (default) or `'rest'`. |
| `client_info` | `google.api_core.gapic_v1.client_info.ClientInfo` | Used to track library usage (e.g., user agent strings). |
| `universe_domain` | `str` | Overrides the default service domain (defaults to `googleapis.com`) for Cloud Universe support. |