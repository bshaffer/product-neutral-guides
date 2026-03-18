# Troubleshooting

## **Debug Logging**

There are a few features built into the Google Cloud Go client libraries which can help you debug your application. This guide will show you how to log client library requests and responses.

> :warning:
>
> These logs are not intended to be used in production and are meant to be used only for quickly
> debugging a project. The logs consists of basic logging to standard output, which may or may not include
> sensitive information. Make sure that once you are done debugging to disable the debugging flag or
> configuration used to avoid leaking sensitive user data. This may also include authentication
> tokens.

### Log examples

```go
// debug_logging_example.go
package main

import (
	"context"
	"fmt"
	"log"

	translate "cloud.google.com/go/translate/apiv3"
	"cloud.google.com/go/translate/apiv3/translatepb"
)

func main() {
	ctx := context.Background()

	// The client will use default settings. To see transport-level logs,
	// see the "Configuration" section below.
	client, err := translate.NewTranslationClient(ctx)
	if err != nil {
		log.Fatalf("Failed to create client: %v", err)
	}
	defer client.Close()

	req := &translatepb.TranslateTextRequest{
		Parent:             "projects/go-docs-samples-kokoro",
		TargetLanguageCode: "en-US",
		Contents:           []string{"こんにちは"},
	}

	// When configured, request and response data will be sent to the logger
	resp, err := client.TranslateText(ctx, req)
	if err != nil {
		log.Fatalf("Failed to translate text: %v", err)
	}

	for _, translation := range resp.GetTranslations() {
		fmt.Printf("Translated text: %v\n", translation.GetTranslatedText())
	}
}
```


```sh
$ GRPC_GO_LOG_SEVERITY_LEVEL=info GRPC_GO_LOG_VERBOSITY_LEVEL=2 go run debug_logging_example.go
{"timestamp":"2024-12-11T19:40:00+00:00","severity":"DEBUG","jsonPayload":{"serviceName":"google.cloud.translation.v3.TranslationService","clientConfiguration":[]}}
{"timestamp":"2024-12-11T19:40:00+00:00","severity":"DEBUG","requestId":3821560043,"jsonPayload":{"request.method":"POST","request.url":"https://oauth2.googleapis.com/token","request.headers":{"Host":["oauth2.googleapis.com"],"Content-Type":["application/x-www-form-urlencoded"],"x-goog-api-client":["gl-go/1.22.0 auth/0.171.0"]},"request.payload":"grant_type=refresh_token&refresh_token=<REFRESH_TOKEN>&client_id=<CLIENT_ID>&client_secret=<CLIENT_SECRET>"}}
{"timestamp":"2024-12-11T19:40:00+00:00","severity":"DEBUG","requestId":3821560043,"jsonPayload":{"response.status":200,"response.headers":{"Date":["Wed, 11 Dec 2024 19:40:00 GMT"],"Content-Type":["application/json; charset=utf-8"]},"response.payload":"{\n  \"access_token\": \"<ACCESS_TOKEN>\",\n  \"expires_in\": 3599,\n  \"token_type\": \"Bearer\"}","latencyMillis":114}}
{"timestamp":"2024-12-11T19:40:00+00:00","severity":"DEBUG","requestId":4274868307,"jsonPayload":{"request.headers":{"x-goog-api-client":["gl-go/1.22.0 gapic/1.12.0 gax/2.12.0 grpc/1.64.0"],"User-Agent":["gcloud-go/1.12.0"],"X-Goog-User-Project":["<YOUR_PROJECT>"]},"request.payload":"{\"contents\":[\"こんにちは\"],\"targetLanguageCode\":\"en-US\",\"parent\":\"projects\\/<YOUR_PROJECT>\"}"}}
{"timestamp":"2024-12-11T19:40:00+00:00","severity":"DEBUG","requestId":4274868307,"jsonPayload":{"response.status":0,"response.payload":"{\"translations\":[{\"translatedText\":\"Hello\",\"detectedLanguageCode\":\"ja\"}]}","latencyMillis":242}}
```

<details>
<summary>Request example log (expanded)</summary>

```json
{
    "timestamp": "2024-12-03T15:21:47-05:00",
    "severity": "DEBUG",
    "requestId": 3821560043,
    "jsonPayload": {
        "request.method": "POST",
        "request.url": "https://translate.googleapis.com/v3/projects/<YOUR_PROJECT>",
        "request.headers": {
            "Host": [
                "translate.googleapis.com"
            ],
            "Content-Type": [
                "application/json"
            ],
            "x-goog-api-client": [
                "gl-go/1.22.0 gapic/1.12.0 gax/2.12.0 grpc/1.64.0"
            ],
            "User-Agent": [
                "gcloud-go/1.12.0"
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
    "requestId": 3821560043,
    "jsonPayload": {
        "response.headers": {
            "Content-Type": [
                "application/json; charset=UTF-8"
            ],
            "Date": [
                "Tue, 03 Dec 2024 20:21:47 GMT"
            ],
            "Server": [
                "ESF"
            ],
            "Cache-Control": [
                "private"
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

### The gRPC environment variables

You can enable logging for gRPC-based clients by setting specific environment variables. When these are set, the underlying gRPC transport will output connection and payload information to standard error.

```sh
export GRPC_GO_LOG_SEVERITY_LEVEL=info
export GRPC_GO_LOG_VERBOSITY_LEVEL=2
```

Logs usually include connection attempts, handshakes, and individual RPC metadata. If the client performs a request to the authentication server, it may also log details related to token acquisition.

### Passing a custom Logger using Interceptors

Go clients allow you to inject custom logic into the request lifecycle using interceptors. You can use this to integrate with structured logging libraries like `slog` or the standard `log` package.

```go
import (
    "context"
    "log/slog"
    "os"

    translate "cloud.google.com/go/translate/apiv3"
    "google.golang.org/api/option"
    "google.golang.org/grpc"
)

logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelDebug}))

// Create an interceptor to log requests and responses
loggingInterceptor := func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
    logger.Debug("Calling method", "method", method, "request", req)
    err := invoker(ctx, method, req, reply, cc, opts...)
    logger.Debug("Completed call", "method", method, "response", reply, "error", err)
    return err
}

client, err := translate.NewTranslationClient(ctx,
    option.WithGRPCDialOption(grpc.WithUnaryInterceptor(loggingInterceptor)),
)
```

With this approach, you are using a structured logger to handle the output. This provides the flexibility to extend logging capabilities, such as sending logs to a centralized logging service or filtering sensitive fields.

### Disabling Logging

To disable logging for a specific client, simply omit the interceptor or logger configuration for that instance.

```go
// This client will not have the logging interceptor attached
client, err := translate.NewTranslationClient(ctx)
```

By controlling which clients receive the interceptor, you can isolate logging to specific services to avoid excessive noise in your output.

```go
// Bigtable client with logging enabled via interceptor
bigtableClient, err := bigtable.NewClient(ctx, projectID, instanceID,
    option.WithGRPCDialOption(grpc.WithUnaryInterceptor(loggingInterceptor)),
)

// Translation client without logging
translationClient, err := translate.NewTranslationClient(ctx)
```

## **How can I trace gRPC issues?**

When working with libraries that use gRPC (the default transport for most Google Cloud Go clients), you can use the gRPC-Go environment variables to enable detailed internal logging.

### **Prerequisites**

Ensure your project is using Go modules. You can verify your dependencies in your `go.mod` file.

For detailed instructions on setting up your environment, see the [Go Google Cloud documentation](https://cloud.google.com/go/docs/reference).

### **Transport logging with gRPC**

The primary method for debugging gRPC calls in Go is setting environment variables that affect the gRPC-Go implementation. The important variables for diagnostics are `GRPC_GO_LOG_SEVERITY_LEVEL` and `GRPC_GO_LOG_VERBOSITY_LEVEL`.

For example, setting `GRPC_GO_LOG_SEVERITY_LEVEL=info` and `GRPC_GO_LOG_VERBOSITY_LEVEL=2` will dump detailed information about transport events and RPC lifecycles.

```sh
GRPC_GO_LOG_SEVERITY_LEVEL=info GRPC_GO_LOG_VERBOSITY_LEVEL=2 go run your_script.go
```

## **How can I diagnose proxy issues?**

See [Client Configuration: Configuring a Proxy](/CLIENT_CONFIGURATION.md).

## **Reporting a problem**

If none of the above advice helps to resolve your issue, please ask for help. If you have a support contract with Google, please create an issue in the [support console](https://cloud.google.com/support/) instead of filing on GitHub. This will ensure a timely response.

Otherwise, please either file an issue on GitHub or ask a question on [Stack Overflow](https://stackoverflow.com/). In most cases creating a GitHub issue will result in a quicker turnaround time, but if you believe your question is likely to help other users in the future, Stack Overflow is a good option. When creating a Stack Overflow question, please use the [google-cloud-platform tag](https://stackoverflow.com/questions/tagged/google-cloud-platform) and [go tag](https://stackoverflow.com/questions/tagged/go).

Although there are multiple GitHub repositories associated with the Google Cloud Libraries, we recommend filing an issue in [https://github.com/googleapis/google-cloud-go](https://github.com/googleapis/google-cloud-go) unless you are certain that it belongs elsewhere. The maintainers may move it to a different repository where appropriate, but you will be notified of this via the email associated with your GitHub account.

When filing an issue or asking a Stack Overflow question, please include as much of the following information as possible. This will enable us to help you quickly.