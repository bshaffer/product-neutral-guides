# Configuring Client Options for Google Cloud Node.js

The Google Cloud Node.js Client Libraries (built on `google-gax`) allow you to configure client behavior via a configuration object passed to the client constructor. This object is used to initialize the underlying service and transport layers.

## 1. Customizing the API Endpoint

You can modify the API endpoint to connect to a specific Google Cloud region (to reduce latency or meet data residency requirements) or to a private endpoint (via Private Service Connect).

Some services, like Pub/Sub and Spanner, offer **regional endpoints**:

```javascript
const {PubSub} = require('@google-cloud/pubsub');

const pubsub = new PubSub({
    // Connect explicitly to the us-east1 region
    apiEndpoint: 'us-east1-pubsub.googleapis.com'
});
```

## 2. Authentication Configuration

While the client attempts to find [Application Default Credentials][adc] automatically, you can explicitly provide them using the `credentials` or `keyFilename` options. See [`Authentication`][authentication.md] for details and examples.

[adc]: https://cloud.google.com/docs/authentication/application-default-credentials
[authentication.md]: https://cloud.google.com/nodejs/docs/reference/google-auth-library/latest

## 3. Logging

Logging and debugging information can be enabled using environment variables. The Node.js client libraries use the `debug` module and gRPC's internal logging. See [Troubleshooting](DEBUG.md) for a comprehensive guide.

## 3. Configuring a Proxy

The configuration method depends on whether you are using the gRPC (default) or HTTP/1.1 (REST) transport.

### Proxy with gRPC

When using the gRPC transport, the client library respects standard environment variables. You do not need to configure this in the Node.js code itself.

Set the following environment variables in your shell or Docker container:

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

If you are using the REST transport (by setting the `fallback` option to `true`), you must configure the proxy by providing a custom HTTPS agent.

```javascript
const {SecretManagerServiceClient} = require('@google-cloud/secret-manager');
const {HttpsProxyAgent} = require('https-proxy-agent');

const proxy = 'http://user:password@proxy.example.com';
const agent = new HttpsProxyAgent(proxy);

const secretManagerClient = new SecretManagerServiceClient({
    fallback: true,
    gaxOpts: {
        agent: agent
    }
});
```

## 4. Configuring Retries and Timeouts

There are two ways to configure retries and timeouts: global client configuration and per-call configuration (simple).

### Per-Call Configuration (Recommended)

For most use cases, it is cleaner to override settings for specific calls using the `options` argument in the service methods.

#### Available `retrySettings` Keys

When passing a `retrySettings` object, you can use the following keys to fine-tune the exponential backoff strategy:

| Key | Type | Description |
| ----- | ----- | ----- |
| `retryCodes` | `number[]` | An array of gRPC status codes that trigger a retry. |
| `initialRetryDelayMillis` | `number` | Wait time before the first retry (in ms). |
| `retryDelayMultiplier` | `number` | Multiplier applied to the delay after each failure (e.g., `1.5`). |
| `maxRetryDelayMillis` | `number` | The maximum wait time between any two retries. |
| `initialRpcTimeoutMillis` | `number` | Initial timeout for the individual RPC call. |
| `totalTimeoutMillis` | `number` | Total time allowed for the request (including all retries) before giving up. |

#### Example: Advanced Backoff

```javascript
// Advanced Retry Configuration
const callOptions = {
    retrySettings: {
        initialRetryDelayMillis: 500,  // Start with 0.5s wait
        retryDelayMultiplier: 2.0,      // Double the wait each time (0.5s -> 1s -> 2s)
        maxRetryDelayMillis: 5000,     // Cap wait at 5s
        totalTimeoutMillis: 15000      // Max 15s total
    }
};

const [version] = await secretManagerClient.accessSecretVersion(request, callOptions);
```

### Disabling Retries

You can configure retries globally by passing `gaxOpts` to the constructor. To disable retries, you can set `retrySettings` to `null` or provide an empty array of retry codes.

```javascript
const {PubSub} = require('@google-cloud/pubsub');

const pubsub = new PubSub({
    // Disable retries for the entire client
    retrySettings: null
});
```

## 5. Logging

To debug request headers, status codes, and payloads, you can use environment variables to trigger internal gRPC and library tracing.

To see detailed gRPC logs, set the following environment variables:

```bash
# Enable logging for all gRPC events
export GRPC_VERBOSITY=DEBUG
export GRPC_TRACE=all
```

For general library debugging, many Google Cloud Node.js libraries use the `debug` package:

```bash
# Enable debugging for specific library components
export DEBUG=google-gax, @google-cloud/pubsub
```

## 6. Other Common Configuration Options

The following options can be passed to the constructor of Google Cloud clients (e.g., `PubSub`, `Spanner`, `Storage`).

| Option Key | Type | Description |
| ----- | ----- | ----- |
| `credentials` | `object` | An object containing the client email and private key. |
| `keyFilename` | `string` | Path to a JSON, OTF, or PEM key file. |
| `apiEndpoint` | `string` | The address of the API remote host. Useful for **Regional Endpoints** (e.g., `us-central1-pubsub.googleapis.com`) or Private Service Connect. |
| `fallback` | `boolean` | When set to `true`, uses HTTP/1.1 (REST) transport instead of gRPC. |
| `gaxOpts` | `object` | Configuration options passed to the underlying `google-gax` layer, including retry and timeout defaults. |
| `universeDomain` | `string` | Overrides the default service domain (defaults to `googleapis.com`) for Cloud Universe support. |