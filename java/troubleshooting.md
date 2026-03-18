# Troubleshooting

## **Debug Logging**

There are a few features built into the Google Cloud Java client libraries which can help you debug
your application. This guide will show you how to log client library requests and responses.

> :warning:
>
> These logs are not intended to be used in production and are meant to be used only for quickly
> debugging a project. The logs consists of basic logging to STDOUT, which may or may not include
> sensitive information. Make sure that once you are done debugging to disable the debugging flag or
> configuration used to avoid leaking sensitive user data. This may also include authentication
> tokens.

### Log examples

```java
import com.google.cloud.translate.v3.LocationName;
import com.google.cloud.translate.v3.TranslateTextRequest;
import com.google.cloud.translate.v3.TranslateTextResponse;
import com.google.cloud.translate.v3.TranslationServiceClient;
import java.io.IOException;

public class DebugLoggingExample {
  public static void main(String[] args) throws IOException {
    try (TranslationServiceClient client = TranslationServiceClient.create()) {
      TranslateTextRequest request =
          TranslateTextRequest.newBuilder()
              .setTargetLanguageCode("en-US")
              .addContents("こんにちは")
              .setParent(LocationName.of("java-docs-samples-kokoro", "global").toString())
              .build();

      // The request and response will be logged when the logging level is set to FINE
      TranslateTextResponse response = client.translateText(request);
    }
  }
}
```


```sh
$ java -Djava.util.logging.config.file=logging.properties -jar debug-logging-example.jar
{"timestamp":"2024-12-11T19:40:00+00:00","severity":"DEBUG","threadId":1,"jsonPayload":{"serviceName":"google.cloud.translation.v3.TranslationService","clientConfiguration":[]}}
{"timestamp":"2024-12-11T19:40:00+00:00","severity":"DEBUG","threadId":1,"requestId":3821560043,"jsonPayload":{"request.method":"POST","request.url":"https://oauth2.googleapis.com/token","request.headers":{"Host":["oauth2.googleapis.com"],"Cache-Control":["no-store"],"Content-Type":["application/x-www-form-urlencoded"],"x-goog-api-client":["gl-java/21.0.1 auth/1.25.0"]},"request.payload":"grant_type=refresh_token&refresh_token=<REFRESH_TOKEN>&client_id=<CLIENT_ID>&client_secret=<CLIENT_SECRET>"}}
{"timestamp":"2024-12-11T19:40:00+00:00","severity":"DEBUG","threadId":1,"requestId":3821560043,"jsonPayload":{"response.status":200,"response.headers":{"x-google-esf-cloud-client-params":["backend_service_name: \"oauth2.googleapis.com\" backend_fully_qualified_method: \"google.identity.oauth2.OAuth2Service.GetToken\""],"Date":["Wed, 11 Dec 2024 19:40:00 GMT"],"Cache-Control":["no-cache, no-store, max-age=0, must-revalidate"],"Content-Type":["application/json; charset=utf-8"]},"response.payload":"{\n  \"access_token\": \"<ACCESS_TOKEN>\",\n  \"expires_in\": 3599,\n  \"token_type\": \"Bearer\"\n}","latencyMillis":114}}
{"timestamp":"2024-12-11T19:40:00+00:00","severity":"DEBUG","threadId":1,"requestId":4274868307,"jsonPayload":{"request.headers":{"x-goog-api-client":["gl-java/21.0.1 gapic/2.40.0 gax/2.48.0 grpc/1.62.2"],"X-Goog-User-Project":["<YOUR_PROJECT>"],"x-goog-request-params":["parent=projects%2F<YOUR_PROJECT>"]},"request.payload":"{\"contents\":[\"こんにちは\"],\"targetLanguageCode\":\"en-US\",\"parent\":\"projects\\/<YOUR_PROJECT>\"}"}}
{"timestamp":"2024-12-11T19:40:00+00:00","severity":"DEBUG","threadId":1,"requestId":4274868307,"jsonPayload":{"response.status":0,"response.payload":"{\"translations\":[{\"translatedText\":\"Hello\",\"detectedLanguageCode\":\"ja\"}]}","latencyMillis":242}}
```

<details>
<summary>Request example log (expanded)</summary>

```json
{
    "timestamp": "2024-12-03T15:21:47-05:00",
    "severity": "DEBUG",
    "threadId": 1,
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
                "gl-java/21.0.1 gapic/2.40.0 gax/2.48.0 grpc/1.62.2"
            ],
            "User-Agent": [
                "gcloud-java/2.40.0"
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
    "threadId": 1,
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

### The `java.util.logging` configuration

You can enable logging for the clients by configuring the Java logging system. Create a `logging.properties` file to set the log level to `FINE` for the Google Cloud libraries. Once this configuration is loaded, the clients will start logging requests to the console.

```properties
# logging.properties
handlers=java.util.logging.ConsoleHandler
com.google.cloud.level=FINE
java.util.logging.ConsoleHandler.level=FINE
```

Logs usually come with a request log and a response log, the exception being streaming requests where it logs each stream packet. This means that if the client performs a request to the auth server, it will also log that request-response pair before the main request.

### Using a custom Logger

The Google Cloud Java libraries use standard logging abstractions. If you are using a logging framework like SLF4J with Logback or Log4j2, you can configure the levels in your respective configuration files (e.g., `logback.xml` or `log4j2.xml`).

```xml
<!-- logback.xml example -->
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <logger name="com.google.cloud" level="DEBUG"/>
  <root level="INFO">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

This allows you to manage logs using your existing enterprise logging infrastructure. This also opens the opportunity to extend logging capabilities, such as sending logs to a centralized logging server or formatting them as JSON.

### Disabling logging for specific clients

If you want to disable logging for a specific client while keeping it enabled for others, you can configure the specific logger associated with that client's package.

```java
import java.util.logging.Level;
import java.util.logging.Logger;

// Disable logging for the Translation service specifically
Logger.getLogger("com.google.cloud.translate").setLevel(Level.OFF);
```

With this you can have different clients and either log in only one or disable individual clients from logging to avoid excessive noise.

```java
// The BigQuery client will log requests if the global level is DEBUG/FINE
BigQuery bigquery = BigQueryOptions.getDefaultInstance().getService();

// Disable logging specifically for the Translation service
Logger.getLogger("com.google.cloud.translate").setLevel(Level.OFF);
TranslationServiceClient translation = TranslationServiceClient.create();
```

## **How can I trace gRPC issues?**

When working with libraries that use gRPC (which is the default transport for many Google Cloud Java clients), you can enable tracing by configuring the `io.grpc` logger levels.

### **Prerequisites**

Ensure you have the necessary gRPC dependencies in your `pom.xml` or `build.gradle`. gRPC is included by default when you use the Google Cloud client libraries.

### **Transport logging with gRPC**

The primary method for debugging gRPC calls in Java is setting the log level for the `io.grpc` package. You can do this via a `logging.properties` file or programmatically.

```properties
# logging.properties
io.grpc.level=FINE
io.grpc.netty.level=FINE
```

If you need even more detailed information about the HTTP/2 frames and low-level network traffic, you can set the level to `FINEST`. Note that this will generate a significant volume of logs.

```java
Logger.getLogger("io.grpc").setLevel(Level.FINEST);
```

## **How can I diagnose proxy issues?**

See [Client Configuration: Configuring a Proxy](/CLIENT_CONFIGURATION.md).

## **Reporting a problem**

If none of the above advice helps to resolve your issue, please ask for help. If you have a support contract with Google, please create an issue in the [support console](https://cloud.google.com/support/) instead of filing on GitHub. This will ensure a timely response.

Otherwise, please either file an issue on GitHub or ask a question on [Stack Overflow](https://stackoverflow.com/). In most cases creating a GitHub issue will result in a quicker turnaround time, but if you believe your question is likely to help other users in the future, Stack Overflow is a good option. When creating a Stack Overflow question, please use the [google-cloud-platform tag](https://stackoverflow.com/questions/tagged/google-cloud-platform) and [java tag](https://stackoverflow.com/questions/tagged/java).

Although there are multiple GitHub repositories associated with the Google Cloud Libraries, we recommend filing an issue in [https://github.com/googleapis/google-cloud-java](https://github.com/googleapis/google-cloud-java) unless you are certain that it belongs elsewhere. The maintainers may move it to a different repository where appropriate, but you will be notified of this via the email associated with your GitHub account.

When filing an issue or asking a Stack Overflow question, please include as much of the following information as possible. This will enable us to help you quickly.