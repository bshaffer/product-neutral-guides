# {{ gcp_name }} Java Client Library: Core Concepts

This documentation covers essential patterns and usage for the {{ gcp_name }} Java Client Library, focusing on performance (gRPC), data handling (Protobuf, Update Masks), and flow control (Pagination, LROs, Streaming).

## 1. Pagination

Most list methods in the {{ gcp_name }} Java library return a `PagedResponse` object. This allows you to iterate over results without manually managing page tokens.

The easiest way to handle pagination is to use the `iterateAll()` method. The library automatically fetches new pages in the background as you iterate through the collection.

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
    // This returns a PagedResponse
    SecretManagerServiceClient.ListSecretsPagedResponse response = secretManager.listSecrets(request);

    // Automatically fetches subsequent pages of secrets
    for (Secret secret : response.iterateAll()) {
        System.out.printf("Secret: %s%n", secret.getName());
    }
}
```

### Manual Pagination (Accessing Tokens)

If you need to control pagination manually (e.g., for a web API that sends tokens to a frontend), you can access the `nextPageToken` from the response.

```java
// Prepare request with page size and optional token
ListSecretsRequest.Builder requestBuilder = ListSecretsRequest.newBuilder()
    .setParent(ProjectName.of("my-project").toString())
    .setPageSize(10);

// Check if we have a token from a previous request (e.g., from a query parameter)
String pageToken = request.getParameter("page_token");
if (pageToken != null) {
    requestBuilder.setPageToken(pageToken);
}

SecretManagerServiceClient.ListSecretsPagedResponse response = secretManager.listSecrets(requestBuilder.build());

// Process current page items
for (Secret secret : response.getPage().getValues()) {
    // Process current page items
}

// Get the token for the next page (empty string if no more pages)
String nextToken = response.getNextPageToken();
```

## 2. Long Running Operations (LROs)

Some operations, like creating a Compute Engine instance or training an AI model, take too long to complete in a single HTTP request. These return a **Long Running Operation (LRO)**.

The Java library uses the `OperationFuture` class to manage these.

### Polling for Completion

The standard pattern is to call `get()` on the future object. This will block the current thread until the operation is complete.

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

    // Call the method with the request object
    // Returns an OperationFuture
    OperationFuture<Operation, Operation> operation = instancesClient.insertAsync(request);

    // Wait for the operation to complete
    // This blocks the thread until completion
    Operation result = operation.get();

    if (operation.isDone()) {
        System.out.println("Operation completed successfully");
    }
} catch (Exception e) {
    // Handle error
    e.printStackTrace();
}
```

### Async / Non-Blocking Check

If you don't want to block the thread, you can store the Operation Name and check it later using the operations client.

```java
// Start operation
OperationFuture<Operation, Operation> operation = client.longRunningMethodAsync(...);
String operationName = operation.getName();

// ... later, or in a different worker process ...

// Resume operation
Operation currentOperation = client.getOperationsClient().getOperation(operationName);

if (currentOperation.getDone()) {
    // Handle success
}
```

## 3. Update Masks

When updating resources (PATCH requests), {{ gcp_name }} APIs often use an **Update Mask** (`com.google.protobuf.FieldMask`). This tells the server *exactly* which fields you intend to update, preventing accidental overwrites of other fields.

If you do not provide a mask, some APIs update **all** fields, resetting missing ones to default values.

### Constructing a FieldMask

```java
import com.google.cloud.secretmanager.v1.Secret;
import com.google.cloud.secretmanager.v1.SecretManagerServiceClient;
import com.google.cloud.secretmanager.v1.UpdateSecretRequest;
import com.google.protobuf.FieldMask;

try (SecretManagerServiceClient client = SecretManagerServiceClient.create()) {
    // Prepare the resource with NEW values
    Secret secret = Secret.newBuilder()
        .setName("projects/my-project/secrets/my-secret")
        .putLabels("env", "production") // We only want to update this field
        .build();

    // Create the FieldMask
    // Paths MUST match the protobuf field names (snake_case)
    FieldMask updateMask = FieldMask.newBuilder()
        .addPaths("labels")
        .build();

    // Prepare the Request object
    UpdateSecretRequest request = UpdateSecretRequest.newBuilder()
        .setSecret(secret)
        .setUpdateMask(updateMask)
        .build();

    // Call the API
    client.updateSecret(request);
}
```

**Note:** The field names in `paths` should be the protocol buffer field names (usually `snake_case`), even though Java methods use `camelCase`.

## 4. Protobuf and gRPC

The {{ gcp_name }} Java library primarily uses **gRPC** as the transport layer, with **REST** available for specific services.

* **Protobuf (Protocol Buffers):** A mechanism for serializing structured data. It is the interface language for gRPC.
* **gRPC:** A high-performance, open-source universal RPC framework. It is generally faster than REST due to efficient binary serialization and HTTP/2 support.

### Installation & Setup

To use the library in your Java project, you add the dependencies via Maven or Gradle. gRPC and Protobuf support are included by default in the client libraries.

**Maven:**
```xml
<dependency>
  <groupId>com.google.cloud</groupId>
  <artifactId>google-cloud-secretmanager</artifactId>
  <version>2.45.0</version>
</dependency>
```

**Gradle:**
```gradle
implementation 'com.google.cloud:google-cloud-secretmanager:2.45.0'
```

### Usage in Client

The client library uses gRPC by default. You can configure the transport via the `StubSettings` if you need to specifically switch to REST (where supported).

```java
import com.google.cloud.pubsub.v1.Publisher;
import com.google.pubsub.v1.TopicName;

// Most Java clients default to gRPC
Publisher publisher = Publisher.newBuilder(TopicName.of("my-project", "my-topic")).build();
```

## 5. gRPC Streaming

gRPC Streaming allows continuous data flow between client and server. In Java, this is commonly implemented as **Server-Side Streaming**, where the server sends a stream of responses. **Bidirectional Streaming** is also supported, often used in long-running applications or data processing workers.

### Streaming Types

| Type | Description | Common Java Use Case |
| :--- | :--- | :--- |
| **Server-Side** | Client sends one request; Server sends a stream of messages. | Reading large datasets (BigQuery, Spanner) or watching logs. |
| **Client-Side** | Client sends a stream of messages; Server waits for stream to close before sending a response. | Uploading large files or ingesting bulk data. |
| **Bidirectional** | Both Client and Server send a stream of messages independently. | Real-time audio processing (Speech-to-Text), chat applications. |

### Server-Side Streaming (High-Level)

When running a query in BigQuery, the results are handled as an iterable stream.

```java
import com.google.cloud.bigquery.BigQuery;
import com.google.cloud.bigquery.BigQueryOptions;
import com.google.cloud.bigquery.FieldValueList;
import com.google.cloud.bigquery.TableResult;

BigQuery bigquery = BigQueryOptions.getDefaultInstance().getService();
TableResult result = bigquery.query(QueryJobConfiguration.of("SELECT * FROM `bigquery-public-data.samples.shakespeare`"));

// This loop acts as a stream reader
for (FieldValueList row : result.iterateAll()) {
    System.out.println(row);
}
```

### Server-Side Streaming (Low-Level)

The generated clients support gRPC Streaming via `ServerStream`. An example of this is the **BigQuery Storage API**.

```java
import com.google.cloud.bigquery.storage.v1.BigQueryReadClient;
import com.google.cloud.bigquery.storage.v1.ReadRowsRequest;
import com.google.cloud.bigquery.storage.v1.ReadRowsResponse;
import com.google.api.gax.rpc.ServerStream;

try (BigQueryReadClient readClient = BigQueryReadClient.create()) {
    // Streaming requires a valid 'readStream' resource name
    ReadRowsRequest request = ReadRowsRequest.newBuilder()
        .setReadStream("projects/my-proj/locations/us/sessions/session-id/streams/stream-id")
        .build();

    // readRows is a server-side streaming method
    ServerStream<ReadRowsResponse> stream = readClient.readRowsCallable().call(request);

    // Read from the stream
    for (ReadRowsResponse response : stream) {
        // Process the response
        long rowCount = response.getRowCount();
        System.out.printf("Received %d rows%n", rowCount);
    }
}
```

### gRPC Bidirectional Streaming

For services using Bidirectional Streaming, such as **Cloud Speech-to-Text**, you interact with a `BidiStream` or use a `ResponseObserver`. This allows you to write requests and read responses continuously.

The protocol requires you to send a configuration request first, followed by audio data.

```java
import com.google.api.gax.rpc.BidiStream;
import com.google.cloud.speech.v2.AudioEncoding;
import com.google.cloud.speech.v2.ExplicitDecodingConfig;
import com.google.cloud.speech.v2.RecognitionConfig;
import com.google.cloud.speech.v2.SpeechClient;
import com.google.cloud.speech.v2.StreamingRecognitionConfig;
import com.google.cloud.speech.v2.StreamingRecognizeRequest;
import com.google.cloud.speech.v2.StreamingRecognizeResponse;
import com.google.protobuf.ByteString;

try (SpeechClient client = SpeechClient.create()) {
    // streamingRecognize is a bidirectional streaming method
    BidiStream<StreamingRecognizeRequest, StreamingRecognizeResponse> stream =
        client.streamingRecognizeCallable().call();

    // Send the Initial Configuration Request
    RecognitionConfig recognitionConfig = RecognitionConfig.newBuilder()
        .setExplicitDecodingConfig(ExplicitDecodingConfig.newBuilder()
            .setEncoding(AudioEncoding.LINEAR16)
            .setSampleRateHertz(16000)
            .setAudioChannelCount(1)
            .build())
        .build();

    StreamingRecognitionConfig streamingConfig = StreamingRecognitionConfig.newBuilder()
        .setConfig(recognitionConfig)
        .build();

    StreamingRecognizeRequest configRequest = StreamingRecognizeRequest.newBuilder()
        .setRecognizer(recognizerName)
        .setStreamingConfig(streamingConfig)
        .build();

    stream.send(configRequest);

    // Send Audio Data Request(s)
    ByteString audioContent = ByteString.readFrom(new FileInputStream("audio.raw"));
    StreamingRecognizeRequest audioRequest = StreamingRecognizeRequest.newBuilder()
        .setAudio(audioContent)
        .build();

    stream.send(audioRequest);

    // Tell the server we are done sending audio
    stream.closeSend();

    // Read responses from the stream
    for (StreamingRecognizeResponse response : stream) {
        response.getResultsList().forEach(result -> {
            System.out.printf("Transcript: %s%n", result.getAlternatives(0).getTranscript());
        });
    }
}
```