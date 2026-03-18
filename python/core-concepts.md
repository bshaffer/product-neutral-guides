# Google Cloud Python Client Library: Core Concepts

This documentation covers essential patterns and usage for the Google Cloud Python Client Library, focusing on performance (gRPC), data handling (Protobuf, Update Masks), and flow control (Pagination, LROs, Streaming).

## 1. Pagination

Most list methods in the Google Cloud Python library return an iterator. This allows you to iterate over results without manually managing page tokens.

The easiest way to handle pagination is to simply use a `for` loop over the response. The library automatically fetches new pages in the background as you iterate.

```python
from google.cloud import secretmanager_v1

client = secretmanager_v1.SecretManagerServiceClient()

# Prepare the request
parent = "projects/my-project"

# Call the API
# This returns a secret_manager_v1.services.secret_manager_service.pagers.ListSecretsPager
# which is an iterable of Secret objects
page_result = client.list_secrets(request={"parent": parent})

# Automatically fetches subsequent pages of secrets
for secret in page_result:
    print(f"Secret: {secret.name}")
```

### Manual Pagination (Accessing Tokens)

If you need to control pagination manually (e.g., for a web API that sends tokens to a frontend), you can access the `next_page_token`.

```python
# Prepare request with page size
request = {
    "parent": "projects/my-project",
    "page_size": 10,
}

# Check if we have a token from a previous request (e.g., from a query parameter)
page_token = request_params.get("page_token")
if page_token:
    request["page_token"] = page_token

page_result = client.list_secrets(request=request)

# Process current page items
for secret in page_result:
    # Handle secret
    pass

# Get the token for the next page (empty string if no more pages)
next_token = page_result.next_page_token
```

## 2. Long Running Operations (LROs)

Some operations, like creating a Compute Engine instance or training an AI model, take too long to complete in a single HTTP request. These return a **Long Running Operation (LRO)**.

The Python library provides an `Operation` object to manage these.

### Polling for Completion

The standard pattern is to call `result()`, which blocks until the operation is finished.

```python
from google.cloud import compute_v1

instances_client = compute_v1.InstancesClient()

# Prepare the request
request = compute_v1.InsertInstanceRequest(
    project=project,
    zone=zone,
    instance_resource=instance_resource,
)

# Call the method
operation = instances_client.insert(request=request)

# Wait for the operation to complete
# This blocks the script, polling periodically
result = operation.result()

# Handle completion
if not operation.error:
    print("Operation succeeded")
else:
    print(f"Error: {operation.error}")
```

### Async / Non-Blocking Check

If you don't want to block the script, you can check the status and store the operation name to resume later.

```python
# Start operation
operation = client.long_running_method(request=request)
operation_id = operation.operation.name

# ... later, or in a different worker process ...

# Resume operation using the operation name
# Note: Most clients provide a way to get an operation status by name
new_operation = client.get_operation(name=operation_id)

if new_operation.done():
    # Handle success or error
    pass
```

## 3. Update Masks

When updating resources (PATCH requests), Google Cloud APIs often use an **Update Mask** (`google.protobuf.field_mask_pb2.FieldMask`). This tells the server *exactly* which fields you intend to update, preventing accidental overwrites of other fields.

If you do not provide a mask, some APIs update **all** fields, resetting missing ones to default values.

### Constructing a FieldMask

```python
from google.cloud import secretmanager_v1
from google.protobuf import field_mask_pb2

client = secretmanager_v1.SecretManagerServiceClient()

# Prepare the resource with NEW values
secret = secretmanager_v1.Secret(
    name="projects/my-project/secrets/my-secret",
    labels={"env": "production"} # We only want to update this field
)

# Create the FieldMask
# paths MUST match the protobuf field names (snake_case)
update_mask = field_mask_pb2.FieldMask(paths=["labels"])

# Call the API
client.update_secret(
    request={
        "secret": secret,
        "update_mask": update_mask,
    }
)
```

**Note:** The field names in `paths` should be the protocol buffer field names (usually `snake_case`), which typically align with the attribute names in the Python objects.

## 4. Protobuf and gRPC

The Google Cloud Python library primarily uses **gRPC** for transport, though many services also support **REST (HTTP/1.1)**.

* **Protobuf (Protocol Buffers):** A mechanism for serializing structured data. It is the interface language for gRPC.

* **gRPC:** A high-performance, open-source universal RPC framework. It is generally faster than REST due to efficient binary serialization and HTTP/2 support.

### Installation & Setup

To use gRPC and Protobuf, you simply install the libraries via pip. They are included as dependencies for most Google Cloud client libraries.

```bash
# Install the core libraries
pip install grpcio protobuf
```

### Usage in Client

The client library automatically uses gRPC by default. You can force a specific transport using the `transport` option when creating a client.

```python
from google.cloud import pubsub_v1

# Use the gRPC transport (default)
publisher = pubsub_v1.PublisherClient(transport="grpc")

# Or force REST transport if supported by the library
# publisher = pubsub_v1.PublisherClient(transport="rest")
```

## 5. gRPC Streaming

gRPC Streaming allows continuous data flow between client and server. In Python, this is most commonly handled using **Iterators** or **Generators**.

### Streaming Types

| Type | Description | Common Python Use Case |
| :--- | :--- | :--- |
| **Server-Side** | Client sends one request; Server sends a stream of messages. | Reading large datasets (BigQuery, Spanner) or watching logs. |
| **Client-Side** | Client sends a stream of messages; Server waits for stream to close before sending a response. | Uploading large files or ingesting bulk data. |
| **Bidirectional** | Both Client and Server send a stream of messages independently. | Real-time audio processing (Speech-to-Text), chat applications. |

### Server-Side Streaming Example (High-Level)

A common example is running a query in BigQuery. The Python client returns an iterator that streams rows.

```python
from google.cloud import bigquery

client = bigquery.Client()
query_job = client.query("SELECT * FROM `bigquery-public-data.samples.shakespeare` LIMIT 100")

# This loop acts as a stream reader.
# Internally, it reads results as they arrive.
for row in query_job.result():
    print(dict(row))
```

### Server-Side Streaming Example (Low-Level)

Low-level generated clients also support gRPC Streaming. For example, the **BigQuery Storage API** `read_rows` method returns an iterator of response objects.

```python
from google.cloud import bigquery_storage_v1

client = bigquery_storage_v1.BigQueryReadClient()

# Streaming requires a valid read_stream resource name
stream_name = "projects/my-proj/locations/us/sessions/session-id/streams/stream-id"
request = bigquery_storage_v1.ReadRowsRequest(read_stream=stream_name)

# read_rows is a server-side streaming method
stream = client.read_rows(request)

# Read from the stream
for response in stream:
    # response is a ReadRowsResponse object
    row_data = response.avro_rows.serialized_binary_rows
    # Process binary row data
    print(f"Row size: {len(row_data)} bytes")
```

### gRPC Bidirectional Streaming

Bidirectional streaming in Python typically involves passing an iterator of requests to a method and receiving an iterator of responses.

If you are using **Cloud Speech-to-Text**, you will interact with the `streaming_recognize` method.

```python
from google.cloud import speech_v2

client = speech_v2.SpeechClient()

# Define a generator to yield streaming requests
def request_generator():
    # Send the Initial Configuration Request
    config = speech_v2.RecognitionConfig(
        explicit_decoding_config=speech_v2.ExplicitDecodingConfig(
            encoding=speech_v2.ExplicitDecodingConfig.AudioEncoding.LINEAR16,
            sample_rate_hertz=16000,
            audio_channel_count=1,
        )
    )
    streaming_config = speech_v2.StreamingRecognitionConfig(config=config)

    yield speech_v2.StreamingRecognizeRequest(
        recognizer=recognizer_name,
        streaming_config=streaming_config
    )

    # Send Audio Data Request(s)
    with open("audio.raw", "rb") as f:
        while True:
            chunk = f.read(4096)
            if not chunk:
                break
            yield speech_v2.StreamingRecognizeRequest(audio=chunk)

# streaming_recognize is a bidirectional streaming method
# It takes an iterator of requests and returns an iterator of responses
responses = client.streaming_recognize(requests=request_generator())

# Read responses from the stream
for response in responses:
    for result in response.results:
        print(f"Transcript: {result.alternatives[0].transcript}")
```