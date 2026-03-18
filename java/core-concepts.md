# Google Cloud Java Client Library: Core Concepts

This documentation covers essential patterns and usage for the Google Cloud Java Client Library, focusing on performance (gRPC), data handling (Protobuf, Update Masks), and flow control (Pagination, LROs, Streaming).

## 1. Pagination

Most list methods in the Google Cloud Java library return a `PagedResponse` object. This allows you to iterate over results without manually managing page tokens.

The easiest way to handle pagination is to use `iterateAll()`. The library automatically fetches new pages in the background as you iterate.

```java
import com.google.cloud.secretmanager.v1.ListSecretsRequest;
import com.google.cloud.secretmanager.v1.ProjectName;
import com.google.cloud.secretmanager.v1.Secret;
import com.google.cloud.secretmanager.v1.SecretManagerServiceClient;

try (SecretManagerServiceClient secretManager = SecretManagerServiceClient.create()) {
    // Prepare the request
    ListSecretsRequest request = ListSecretsRequest.newBuilder()
        .setParent(ProjectName.of("my-project").toString())
        .build();

    // Call the API
    // This returns a ListSecretsPagedResponse
    SecretManagerServiceClient.ListSecretsPagedResponse response = secretManager.listSecrets(request);

    // Automatically fetches subsequent pages of secrets
    for (Secret secret : response.iterateAll()) {
        System.out.printf("Secret: %s%n", secret.getName());
    }
}
```

### Manual Pagination (Accessing Tokens)

If you need to control pagination manually (e.g., for a web API that sends tokens to a frontend), you can access the `nextPageToken` via the page object.

```java
// Prepare request with page size and optional token
ListSecretsRequest.Builder requestBuilder = ListSecretsRequest.newBuilder()
    .setParent("projects/my-project")
    .setPageSize(10);

// Check if we have a token from a previous request (e.g., from a web parameter)
String pageToken = request.getParameter("page_token");
if (pageToken != null) {
    requestBuilder.setPageToken(pageToken);
}

SecretManagerServiceClient.ListSecretsPagedResponse response = secretManager.listSecrets(requestBuilder.build());

// Process only the current page
for (Secret secret : response.getPage().getValues()) {
    // Process current page items
}

// Get the token for the next page (empty string if no more pages)
String nextToken = response.getNextPageToken();
```

## 2. Long Running Operations (LROs)

Some operations, like creating a Compute Engine instance or training an AI model, take too long to complete in a single HTTP request. These return a **Long Running Operation (LRO)**.

The Java library uses `OperationFuture` to manage these.

### Polling for Completion

The standard pattern is to call `get()` on the future object, which blocks until the operation is complete.

```java
import com.google.api.gax.longrunning.OperationFuture;
import com.google.cloud.compute.v1.Instance;
import com.google.cloud.compute.v1.InstancesClient;
import com.google.cloud.compute.v1.InsertInstanceRequest;
import com.google.cloud.compute.v1.Operation;

try (InstancesClient instancesClient = InstancesClient.create()) {
    // Prepare the Request object
    InsertInstanceRequest request = InsertInstanceRequest.newBuilder()
        .setProject(project)
        .setZone(zone)
        .setInstanceResource(instanceResource)
        .build();

    // Call the method, which returns an OperationFuture
    OperationFuture<Operation, Operation> resultFuture = instancesClient.insertAsync(request);

    // Wait for the operation to complete
    // This blocks the thread until completion
    Operation response = resultFuture.get();

    if (resultFuture.isDone()) {
        System.out.println("Operation completed successfully");
    }
} catch (Exception e) {
    // Handle error
    e.printStackTrace();
}
```

### Async / Non-Blocking Check

If you don't want to block the thread, you can store the Operation Name and check it later using the `OperationsClient`.

```java
// 1. Start operation
OperationFuture<Operation, Operation> resultFuture = client.longRunningMethodAsync(...);
String operationName = resultFuture.getName();

// ... later, or in a different worker process ...

// 2. Resume operation using the name
Operation operation = client.getOperationsClient().getOperation(operationName);

if (operation.getDone()) {
    // Handle success or error
}
```

## 3. Update Masks

When updating resources (PATCH requests), Google Cloud APIs often use an **Update Mask** (`com.google.protobuf.FieldMask`). This tells the server *exactly* which fields you intend to update, preventing accidental overwrites of other fields.

If you do not provide a mask, some APIs update **all** fields, resetting missing ones to default values.

### Constructing a FieldMask

```java
import com.google.cloud.secretmanager.v1.Secret;
import com.google.cloud.secretmanager.v1.SecretManagerServiceClient;
import com.google.cloud.secretmanager.v1.UpdateSecretRequest;
import com.google.protobuf.FieldMask;

try (SecretManagerServiceClient client = SecretManagerServiceClient.create()) {
    // 1. Prepare the resource with NEW values
    Secret secret = Secret.newBuilder()
        .setName("projects/my-project/secrets/my-secret")
        .putLabels("env", "production") // We only want to update this field
        .build();

    // 2. Create the FieldMask
    // Paths MUST match the protobuf field names (snake_case)
    FieldMask updateMask = FieldMask.newBuilder()
        .addPaths("labels")
        .build();

    // 3. Prepare the Request object
    UpdateSecretRequest request = UpdateSecretRequest.newBuilder()
        .setSecret(secret)
        .setUpdateMask(updateMask)
        .build();

    // 4. Call the API
    client.updateSecret(request);
}
```

**Note:** The field names in `paths` should be the protocol buffer field names (usually `snake_case`), even though Java methods use `camelCase`.

## 4. Protobuf and gRPC

The Google Cloud Java library primarily uses **gRPC** as its transport, with **Protobuf (Protocol Buffers)** for serialization.

* **Protobuf:** A mechanism for serializing structured data. It is the interface language for gRPC.
* **gRPC:** A high-performance RPC framework. Unlike PHP, Java has native support for gRPC via managed dependencies, and it does not require external C extensions.

### Installation & Setup

In Java, dependencies are managed via Maven or Gradle. gRPC and Protobuf dependencies are automatically pulled in when you include a Google Cloud client library.

**Maven (`pom.xml`):**
```xml
<dependency>
  <groupId>com.google.cloud</groupId>
  <artifactId>google-cloud-secretmanager</artifactId>
  <version>2.45.0</version>
</dependency>
```

### Usage in Client

The client library uses gRPC by default. If a service supports REST, you can configure it via the `ClientSettings`.

```java
SecretManagerServiceSettings secretManagerServiceSettings =
    SecretManagerServiceSettings.newBuilder()
        .setTransportChannelProvider(
            SecretManagerServiceSettings.defaultHttpJsonTransportProviderBuilder().build())
        .build();

SecretManagerServiceClient client = SecretManagerServiceClient.create(secretManagerServiceSettings);
```

## 5. gRPC Streaming

gRPC Streaming allows continuous data flow between client and server. In Java, this is handled through `StreamObserver` objects or `ServerStream` iterables.

### Streaming Types

| Type | Description | Common Java Use Case |
| :--- | :--- | :--- |
| **Server-Side** | Client sends one request; Server sends a stream of messages. | Reading large datasets (BigQuery Storage, Spanner) or watching logs. |
| **Client-Side** | Client sends a stream of messages; Server waits for stream to close before sending a response. | Uploading large files or ingesting bulk data. |
| **Bidirectional** | Both Client and Server send a stream of messages independently. | Real-time audio processing (Speech-to-Text). |

### Server-Side Streaming Example (High-Level)

For many services like BigQuery, streaming is abstracted into simple iterables.

```java
import com.google.cloud.bigquery.BigQuery;
import com.google.cloud.bigquery.BigQueryOptions;
import com.google.cloud.bigquery.FieldValueList;
import com.google.cloud.bigquery.TableResult;

BigQuery bigquery = BigQueryOptions.getDefaultInstance().getService();
TableResult result = bigquery.query(QueryJobConfiguration.of("SELECT * FROM ..."));

// Internally, this iterates over stream results
for (FieldValueList row : result.iterateAll()) {
    System.out.println(row);
}
```

### Server-Side Streaming Example (Low-Level)

The BigQuery Storage API provides a direct `ServerStream`.

```java
import com.google.api.gax.rpc.ServerStream;
import com.google.cloud.bigquery.storage.v1.BigQueryReadClient;
import com.google.cloud.bigquery.storage.v1.ReadRowsRequest;
import com.google.cloud.bigquery.storage.v1.ReadRowsResponse;

try (BigQueryReadClient readClient = BigQueryReadClient.create()) {
    ReadRowsRequest request = ReadRowsRequest.newBuilder()
        .setReadStream("projects/my-proj/locations/us/sessions/session-id/streams/stream-id")
        .build();

    // readRows returns a ServerStream
    ServerStream<ReadRowsResponse> stream = readClient.readRowsCallable().call(request);

    for (ReadRowsResponse response : stream) {
        byte[] rowData = response.getAvroRows().getSerializedBinaryRows().toByteArray();
        System.out.printf("Row size: %d bytes%n", rowData.length);
    }
}
```

### gRPC Bidirectional Streaming

For Bidirectional APIs like **Cloud Speech-to-Text**, Java uses a `BidiStream` or `ClientStream` pattern.

```java
import com.google.api.gax.rpc.BidiStream;
import com.google.cloud.speech.v2.RecognitionConfig;
import com.google.cloud.speech.v2.SpeechClient;
import com.google.cloud.speech.v2.StreamingRecognitionConfig;
import com.google.cloud.speech.v2.StreamingRecognizeRequest;
import com.google.cloud.speech.v2.StreamingRecognizeResponse;
import com.google.protobuf.ByteString;

try (SpeechClient client = SpeechClient.create()) {
    BidiStream<StreamingRecognizeRequest, StreamingRecognizeResponse> stream =
        client.streamingRecognizeCallable().call();

    // 1. Send the Initial Configuration Request
    StreamingRecognitionConfig streamingConfig = StreamingRecognitionConfig.newBuilder()
        .setConfig(RecognitionConfig.newBuilder()
            .setExplicitDecodingConfig(ExplicitDecodingConfig.newBuilder()
                .setEncoding(AudioEncoding.LINEAR16)
                .setSampleRateHertz(16000)
                .build())
            .build())
        .build();

    stream.send(StreamingRecognizeRequest.newBuilder()
        .setRecognizer(recognizerName)
        .setStreamingConfig(streamingConfig)
        .build());

    // 2. Send Audio Data Request
    stream.send(StreamingRecognizeRequest.newBuilder()
        .setAudio(ByteString.copyFrom(Files.readAllBytes(Paths.get("audio.raw"))))
        .build());

    // Tell the server we are done sending
    stream.closeSend();

    // 3. Read responses from the stream
    for (StreamingRecognizeResponse response : stream) {
        response.getResultsList().forEach(result -> {
            System.out.printf("Transcript: %s%n", result.getAlternatives(0).getTranscript());
        });
    }
}
```

