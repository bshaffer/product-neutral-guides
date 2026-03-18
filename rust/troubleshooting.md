# Troubleshooting

## **Debug Logging**

There are a few features built into the Google Cloud Rust client libraries which can help you debug
your application. This guide will show you how to log client library requests and responses.

> :warning:
>
> These logs are not intended to be used in production and are meant to be used only for quickly
> debugging a project. The logs consists of basic logging to STDOUT, which may or may not include
> sensitive information. Make sure that once you are done debugging to disable the debugging flag or
> configuration used to avoid leaking sensitive user data. This may also include authentication
> tokens.

### Log examples

```rust
// debug-logging-example.rs
use google_cloud_translate::v3::client::TranslationServiceClient;
use google_cloud_translate::v3::model::TranslateTextRequest;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = TranslationServiceClient::new().await?;

    let request = TranslateTextRequest {
        target_language_code: "en-US".to_string(),
        contents: vec!["こんにちは".to_string()],
        parent: "projects/rust-docs-samples-kokoro".to_string(),
        ..Default::default()
    };

    // The request and response will be logged to STDOUT when the environment
    // variable GOOGLE_SDK_RUST_LOGGING=true is set and a tracing subscriber is initialized
    let response = client.translate_text(request).await?;

    Ok(())
}
```


```sh
$ GOOGLE_SDK_RUST_LOGGING=true cargo run --example debug-logging-example
{"timestamp":"2024-12-11T19:40:00+00:00","severity":"DEBUG","processId":44180,"jsonPayload":{"serviceName":"google.cloud.translation.v3.TranslationService","clientConfiguration":[]}}
{"timestamp":"2024-12-11T19:40:00+00:00","severity":"DEBUG","processId":44180,"requestId":3821560043,"jsonPayload":{"request.method":"POST","request.url":"https://oauth2.googleapis.com/token","request.headers":{"Host":["oauth2.googleapis.com"],"Cache-Control":["no-store"],"Content-Type":["application/x-www-form-urlencoded"],"x-goog-api-client":["gl-rust/1.0.0 auth/1.0.0 auth-request-type/at cred-type/u"]},"request.payload":"grant_type=refresh_token&refresh_token=<REFRESH_TOKEN>&client_id=<CLIENT_ID>&client_secret=<CLIENT_SECRET>"}}
{"timestamp":"2024-12-11T19:40:00+00:00","severity":"DEBUG","processId":44180,"requestId":3821560043,"jsonPayload":{"response.status":200,"response.headers":{"x-google-esf-cloud-client-params":["backend_service_name: \"oauth2.googleapis.com\" backend_fully_qualified_method: \"google.identity.oauth2.OAuth2Service.GetToken\""],"X-Google-Session-Info":["<SESSION_INFO>"],"Date":["Wed, 11 Dec 2024 19:40:00 GMT"],"Pragma":["no-cache"],"Expires":["Mon, 01 Jan 1990 00:00:00 GMT"],"Cache-Control":["no-cache, no-store, max-age=0, must-revalidate"],"Content-Type":["application/json; charset=utf-8"],"X-Google-Security-Signals":["FRAMEWORK=ONE_PLATFORM,ENV=borg,ENV_DEBUG=borg_user:identity-oauth2-proxy;borg_job:prod.identity-oauth2-proxy","FRAMEWORK=HTTPSERVER2,BUILD=GOOGLE3,BUILD_DEBUG=cl:694072944,ENV=borg,ENV_DEBUG=borg_user:identity-oauth2-proxy;borg_job:prod.identity-oauth2-proxy"],"Vary":["X-Origin","Referer","Origin,Accept-Encoding"],"Server":["scaffolding on HTTPServer2"],"X-Google-Netmon-Label":["/bns/dz/borg/dz/bns/identity-oauth2-proxy/prod.identity-oauth2-proxy/4"],"X-XSS-Protection":["0"],"X-Frame-Options":["SAMEORIGIN"],"X-Content-Type-Options":["nosniff"],"X-Google-GFE-Service-Trace":["google-identity-oauth2-oauth2proxyservice-prod"],"X-Google-Backends":["unix:/tmp/esfbackend.1733439890.116447.177528,/bns/dz/borg/dz/bns/identity-oauth2-proxy/prod.identity-oauth2-proxy/4,/bns/ncsfoa/borg/ncsfoa/bns/blue-layer1-gfe-prod-edge/prod.blue-layer1-gfe.sfo03s27/15"],"X-Google-GFE-Request-Trace":["acsfon13:443,/bns/dz/borg/dz/bns/identity-oauth2-proxy/prod.identity-oauth2-proxy/4,acsfon13:443"],"X-Google-DOS-Service-Trace":["main:google-identity-oauth2-oauth2proxyservice-prod,main:GLOBAL_all_non_cloud"],"X-Google-GFE-Handshake-Trace":["GFE: /bns/ncsfoa/borg/ncsfoa/bns/blue-layer1-gfe-prod-edge/prod.blue-layer1-gfe.sfo03s27/15,Mentat oracle: [2002:a05:635e:38e:b0:178:f5eb:ee40]:9801"],"X-Google-Service":["google-identity-oauth2-oauth2proxyservice-prod"],"X-Google-GFE-Response-Code-Details-Trace":["response_code_set_by_backend"],"X-Google-GFE-Response-Body-Transformations":["gunzipped,chunked"],"X-Google-Shellfish-Status":["CA0gBEBG"],"X-Google-GFE-Version":["2.903.2"],"Alt-Svc":["h3=\":443\"; ma=2592000,h3-29=\":443\"; ma=2592000"],"Accept-Ranges":["none"],"Transfer-Encoding":["chunked"]},"response.payload":"{\n  \"access_token\": \"<ACCESS_TOKEN>\",\n  \"expires_in\": 3599,\n  \"scope\": \"https://www.googleapis.com/auth/userinfo.email https://www.googleapis.com/auth/sqlservice.login https://www.googleapis.com/auth/cloud-platform openid\",\n  \"token_type\": \"Bearer\",\n  \"id_token\": \"<ID_TOKEN>","latencyMillis":114}}
{"timestamp":"2024-12-11T19:40:00+00:00","severity":"DEBUG","processId":44180,"requestId":4274868307,"jsonPayload":{"request.headers":{"x-goog-api-client":["gl-rust/1.0.0 gapic/1.20.0 gax/1.36.0 grpc/1.59.1 rest/1.36.0 pb/+n"],"User-Agent":["gcloud-rust/1.20.0"],"X-Goog-User-Project":["<YOUR_PROJECT>"],"x-goog-request-params":["parent=projects%2F<YOUR_PROJECT>"]},"request.payload":"{\"contents\":[\"こんにちは\"],\"targetLanguageCode\":\"en-US\",\"parent\":\"projects\\/<YOUR_PROJECT>\"}"}}
{"timestamp":"2024-12-11T19:40:00+00:00","severity":"DEBUG","processId":44180,"requestId":4274868307,"jsonPayload":{"response.status":0,"response.headers":{"pc-high-bwd-bin":["KgIYJQ"]},"response.payload":"{\"translations\":[{\"translatedText\":\"Hello\",\"detectedLanguageCode\":\"ja\"}]}","latencyMillis":242}}
```

<details>
<summary>Request example log (expanded)</summary>

```json
{
    "timestamp": "2024-12-03T15:21:47-05:00",
    "severity": "DEBUG",
    "processId": 44180,
    "requestId": 3821560043,
    "jsonPayload": {
        "request.method": "POST",
        "request.url": "https://translate.googleapis.com/v3/projects/<YOUR_PROJECT",
        "request.headers": {
            "Host": [
                "translate.googleapis.com"
            ],
            "Content-Type": [
                "application/json"
            ],
            "x-goog-api-client": [
                "gl-rust/1.0.0 gapic/1.20.0 gax/1.35.0 grpc/1.66.0 rest/1.35.0 pb/+n"
            ],
            "User-Agent": [
                "gcloud-rust/1.20.0"
            ],
            "X-Goog-User-Project": [
                "<YOUR_PROJECT>"
            ],
            "x-goog-request-params": [
                "parent=projects%2F<YOUR_PROJECT>"
            ],
            "authorization": [
                "Bearer <YOUR_AUTHORIZATION_TOKEN>"
            ]
        },
        "request.payload": "{\"contents\":[\"こんにちは\"],\"targetLanguageCode\":\"en-US\",\"parent\":\"projects\\/<YOUR_PROJECT>\"}"
    }
}
```

</details>
<details>
<summary>Response example log (expanded)</summary>

```json
{
    "timestamp": "2024-12-03T15:21:47-05:00",
    "severity": "DEBUG",
    "processId": 44180,
    "requestId": 3821560043,
    "jsonPayload": {
        "response.headers": {
            "Content-Type": [
                "application/json; charset=UTF-8"
            ],
            "Vary": [
                "X-Origin",
                "Referer",
                "Origin,Accept-Encoding"
            ],
            "Date": [
                "Tue, 03 Dec 2024 20:21:47 GMT"
            ],
            "Server": [
                "ESF"
            ],
            "Cache-Control": [
                "private"
            ],
            "X-XSS-Protection": [
                "0"
            ],
            "X-Frame-Options": [
                "SAMEORIGIN"
            ],
            "X-Content-Type-Options": [
                "nosniff"
            ],
            "Accept-Ranges": [
                "none"
            ],
            "Transfer-Encoding": [
                "chunked"
            ]
        },
        "response.payload": "{\"translations\":[{\"translatedText\": \"Hello\",\"detectedLanguageCode\":\"ja\"}]}",
        "latencyMillis": 152
    }
}
```

</details>

### Configuration

There are a few ways to configure debug logging which we will go through in this document.

### The `GOOGLE_SDK_RUST_LOGGING` environment variable

You can enable logging on all the different clients in your code by setting this environment variable
to `true`. Once this environment variable is set, all the clients used in your code will start
logging the requests into `STDOUT` if a logging subscriber is present.

```rust
std::env::set_var("GOOGLE_SDK_RUST_LOGGING", "true");

let client = TranslationServiceClient::new().await?;
```

Logs usually come with a request log and a response log, with the exception being streaming requests
where, depending on the type of streaming, it logs each stream packet. This means that if the client
performs a request to the auth server, it will also log that request-response pair before the main
request.


### Passing a tracing subscriber

The debugging code has been designed to integrate with the `tracing` ecosystem. We can
initialize a subscriber to handle the client logs.

```rust
use tracing_subscriber::fmt;
use tracing::Level;

let subscriber = fmt()
    .with_max_level(Level::DEBUG)
    .with_writer(std::io::stdout)
    .finish();

tracing::subscriber::set_global_default(subscriber)?;

let client = TranslationServiceClient::new().await?;
```

With this, you will be using a standard tracing subscriber. This also opens
the opportunity to extend the capabilities of logging in case you have specific needs; any
subscriber implementation can be used to manage the logs in any way that is needed.

### Disabling logging in configuration

The client configuration allows you to disable logging for a specific client.

```rust
let config = google_cloud_translate::v3::Config {
    logging_enabled: false,
    ..Default::default()
};

let client = TranslationServiceClient::new_with_config(config).await?;
```

With this, you can have different clients and either log in only one or disable individual clients
from logging to avoid excessive noise.

```rust
std::env::set_var("GOOGLE_SDK_RUST_LOGGING", "true");

// The Big Table client will log all the requests
let bigtable = BigtableClient::new().await?;

// The TranslationServiceClient will not log any requests
let translation_config = google_cloud_translate::v3::Config {
    logging_enabled: false,
    ..Default::default()
};
let translation = TranslationServiceClient::new_with_config(translation_config).await?;
```

## **How can I trace gRPC issues?**

When working with libraries that use gRPC, you can use the underlying gRPC environment variables to enable logging. Most Rust clients use pure-Rust gRPC implementations like `tonic`.

### **Prerequisites**

Ensure your crate includes the necessary features for the gRPC transport. You can verify your dependencies in `Cargo.toml`.

### **Transport logging with gRPC**

The primary method for debugging gRPC calls in Rust is using the `tracing` subscriber filters. You can target specific gRPC crates to see underlying transport details.

For example, setting the `RUST_LOG` environment variable to include `tonic=debug` or `h2=debug` will dump a lot of information regarding the gRPC and HTTP/2 layers.

```sh
RUST_LOG=debug,tonic=debug,h2=debug cargo run --example your_script
```

If you are using a client that wraps the gRPC C-core, environment variables like `GRPC_TRACE` and `GRPC_VERBOSITY` may also be relevant.

## **How can I diagnose proxy issues?**

See [Client Configuration: Configuring a Proxy](/CLIENT_CONFIGURATION.md).

## **Reporting a problem**

If none of the above advice helps to resolve your issue, please ask for help. If you have a support contract with Google, please create an issue in the [support console](https://cloud.google.com/support/) instead of filing on GitHub. This will ensure a timely response.

Otherwise, please either file an issue on GitHub or ask a question on [Stack Overflow](https://stackoverflow.com/). In most cases creating a GitHub issue will result in a quicker turnaround time, but if you believe your question is likely to help other users in the future, Stack Overflow is a good option. When creating a Stack Overflow question, please use the [google-cloud-platform tag](https://stackoverflow.com/questions/tagged/google-cloud-platform) and [rust tag](https://stackoverflow.com/questions/tagged/rust).

Although there are multiple GitHub repositories associated with the Google Cloud Libraries, we recommend filing an issue in [https://github.com/googleapis/google-cloud-rust](https://github.com/googleapis/google-cloud-rust) unless you are certain that it belongs elsewhere. The maintainers may move it to a different repository where appropriate, but you will be notified of this via the email associated with your GitHub account.

When filing an issue or asking a Stack Overflow question, please include as much of the following information as possible. This will enable us to help you quickly.