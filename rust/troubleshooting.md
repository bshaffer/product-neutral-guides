# Troubleshooting

## **Debug Logging**

The best way to troubleshoot is by enabling logging. See
[Enabling Logging](https://docs.cloud.google.com/rust/enable-logging) for more information.

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

See [Client Configuration: Configuring a Proxy](https://docs.cloud.google.com/rust/client-configuration).

## **Reporting a problem**

If none of the above advice helps to resolve your issue, please ask for help. If you have a support contract with Google, please create an issue in the [support console](https://cloud.google.com/support/) instead of filing on GitHub. This will ensure a timely response.

Otherwise, please either file an issue on GitHub or ask a question on [Stack Overflow](https://stackoverflow.com/). In most cases creating a GitHub issue will result in a quicker turnaround time, but if you believe your question is likely to help other users in the future, Stack Overflow is a good option. When creating a Stack Overflow question, please use the [google-cloud-platform tag](https://stackoverflow.com/questions/tagged/google-cloud-platform) and [rust tag](https://stackoverflow.com/questions/tagged/rust).

Although there are multiple GitHub repositories associated with the Google Cloud Libraries, we recommend filing an issue in [https://github.com/googleapis/google-cloud-rust](https://github.com/googleapis/google-cloud-rust) unless you are certain that it belongs elsewhere. The maintainers may move it to a different repository where appropriate, but you will be notified of this via the email associated with your GitHub account.

When filing an issue or asking a Stack Overflow question, please include as much of the following information as possible. This will enable us to help you quickly.