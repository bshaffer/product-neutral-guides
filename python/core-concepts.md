# Google Cloud Python Client Library: Core Concepts

This documentation covers essential patterns and usage for the Google Cloud Python Client Library, focusing on performance (gRPC), data handling (Protobuf, Update Masks), and flow control (Pagination, LROs, Streaming).

## 1. Pagination

Most list methods in the Google Cloud Python library return a `Pager` object. This allows you to iterate over results without manually managing page tokens.

The easiest way to handle pagination is to simply iterate over the response using a `for` loop. The library automatically fetches new pages in the background as you iterate.

```python
from google.cloud import secretmanager_v1

client = secretmanager_v1.SecretManagerServiceClient()

# Prepare the request
# The parent follows the format 'projects/{project_id}'
request = secretmanager_v1.ListSecretsRequest(
    parent="projects/my-project"
)

# Call the API
# This returns a ListSecretsPager, which is an iterable
page_result = client.list_secrets(request=request)

# Automatically fetches subsequent pages of secrets
for secret in page_result:
    print(f"Secret: {secret.name}")
```

### Manual Pagination (Accessing Tokens)

If you need to control pagination manually (e.g., for a web API that sends tokens to a frontend), you can access the `next_page_token` from the page object.

```python
# Prepare request with page size and optional token
page_size = 10
page_token = "previous_token_from_frontend" # Or None

request = secretmanager_v1.ListSecretsRequest(
    parent="projects/my-project",
    page_size=page_size,
    page_token=page_token
)

# Call the API
pager = client.list_secrets(request=request)

# Iterate through the items of the CURRENT page only
# Accessing .pages allows you to see the current page's metadata
current_page = next(pager.pages)

for secret in current_page:
    # Process current page items
    pass

# Get the token for the next page (empty string if no more pages)
next_token = pager.next_page_token
```

## 2. Long Running Operations (LROs)

Some operations, like creating a Compute Engine instance or training an AI model, take too long to complete in a single HTTP request. These return a **Long Running Operation (LRO)**.

The Python library provides an `Operation` object (specifically from `google.api_core.operation`) to manage these.

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
# This returns a google.api_core.operation.Operation object
operation = instances_client.insert(request=request)

# Wait for the operation to complete
# This blocks the script and polls periodically
try:
    result = operation.result()
    # The operation succeeded
    print(f"Result: {result}")
except Exception as e:
    # Handle error
    print(f"Error: {e}")
```

### Async / Non-Blocking Check

If you don't want to block the script, you can store the operation name and check its status later.

```python
# 1. Start operation
operation = client.long_running_method(...)
operation_name = operation.name

# ... later, or in a different worker process ...

# 2. Resume operation
# You can reconstruct the operation object using the name
from google.api_core import operations_v1
# (Note: Specific implementation varies by service, 
# but most clients have a way to fetch an operation by name)
new_operation = client.get_operation(name=operation_name)

if new_operation.done():
    # Handle success or error
    print(new_operation.result())
```

## 3. Update Masks

When updating resources (PATCH requests), Google Cloud APIs often use an **Update Mask** (`google.protobuf.field_mask_pb2.FieldMask`). This tells the server *exactly* which fields you intend to update, preventing accidental overwrites of other fields.

If you do not provide a mask, some APIs update **all** fields, resetting missing ones to default values.

### Constructing a FieldMask

```python
from google.cloud import secretmanager_v1
from google.protobuf import field_mask_pb2

client = secretmanager_v1.SecretManagerServiceClient()

# 1. Prepare the resource with NEW values
secret = secretmanager_v1.Secret(
    name="projects/my-project/secrets/my-secret",
    labels={"env": "production"} # We only want to update this field
)

# 2. Create the FieldMask
# 'paths' MUST match the protobuf field names (snake_case)
update_mask = field_mask_pb2.FieldMask(paths=["labels"])

# 3. Prepare the Request object
request = secretmanager_v1.UpdateSecretRequest(
    secret=secret,
    update_mask=update_mask
)

# 4. Call the API
client.update_secret(request=request)
```

**Note:** The field names in `paths` should be the protocol buffer field names (usually `snake_case`), which aligns with Python's naming conventions.

## 4. Protobuf and gRPC

The Google Cloud Python library supports two transports: **gRPC** and **REST (HTTP/1.1)**.

* **Protobuf (Protocol Buffers):** A mechanism for serializing structured data. It is the interface language for gRPC.

* **gRPC:** A high-performance, open-source universal RPC framework. It is generally faster than REST due to efficient binary serialization and HTTP/2 support.

### Installation & Setup

In Python, the `grpcio` and `protobuf` dependencies are typically installed automatically when you install a Google Cloud library.

```bash
# Install the library
pip install google-cloud-secret-manager
```

For performance-critical applications, ensure you have the C-extension versions of protobuf installed (usually handled by `pip`).

### Usage in Client

The client library automatically chooses gRPC if available. You can force a specific transport using the `transport` argument when creating a client.

```python
from google.cloud import pubsub_v1

# Use gRPC (default)
publisher = pubsub_v1.PublisherClient(transport="grpc")

# Use REST
publisher_rest = pubsub_v1.PublisherClient(transport="rest")
```

## 5. gRPC Streaming

gRPC Streaming allows continuous data flow between client and server. In Python, this is handled via **iterators** and **generators**.

### Streaming Types

| Type | Description | Common Python Use Case |
| :--- | :--- | :--- |
| **Server-Side** | Client sends one request; Server returns an iterator. | Reading large datasets (BigQuery, Spanner) or watching logs. |
| **Client-Side** | Client sends an iterator of messages; Server returns a single response. | Uploading large files or ingesting bulk data. |
| **Bidirectional** | Client sends an iterator; Server returns an iterator. | Real-time audio processing (Speech-to-Text). |

### Server-Side Streaming Example (High-Level)

A common example is running a query in BigQuery. The Python client exposes results as an iterator.

```python
from google.cloud import bigquery

client = bigquery.Client()
query_job = client.query("SELECT * FROM `bigquery-public-data.samples.shakespeare` LIMIT 10")

# This iterator acts as a stream reader.
# Internally, it handles fetching batches from the API.
for row in query_job.result():
    print(dict(row))
```

### Server-Side Streaming Example (Low-Level)

The generated clients support low-level gRPC Streaming. An example is the **BigQuery Storage API**. The `read_rows` method returns an iterable of responses.

```python
from google.cloud import bigquery_storage_v1

client = bigquery_storage_v1.BigQueryReadClient()

# Streaming requires a valid 'read_stream' resource name
request = bigquery_storage_v1.ReadRowsRequest(
    read_stream="projects/my-proj/locations/us/sessions/session-id/streams/stream-id"
)

# read_rows is a server-side streaming method
stream = client.read_rows(request)

# Iterate over the responses
for response in stream:
    # response is a ReadRowsResponse object
    row_data = response.avro_rows.serialized_binary_rows
    # Process binary row data
    print(f"Row size: {len(row_data)} bytes")
```

### gRPC Bidirectional Streaming

In Python, bidirectional streaming methods usually accept an **iterator** (or generator) of request objects and return an **iterator** of response objects.

```python
from google.cloud import speech_v2

client = speech_v2.SpeechClient()

# 1. Define a generator to yield requests
def request_generator():
    # Send Initial Configuration
    yield speech_v2.StreamingRecognizeRequest(
        recognizer=recognizer_name,
        streaming_config=speech_v2.StreamingRecognitionConfig(
            config=speech_v2.RecognitionConfig(
                explicit_decoding_config=speech_v2.ExplicitDecodingConfig(
                    encoding=speech_v2.ExplicitDecodingConfig.AudioEncoding.LINEAR16,
                    sample_rate_hertz=16000,
                    audio_channel_count=1,
                )
            )
        )
    )
    
    # Send Audio Data
    with open("audio.raw", "rb") as f:
        while True:
            chunk = f.read(4096)
            if not chunk:
                break
            yield speech_v2.StreamingRecognizeRequest(audio=chunk)

# 2. Call the bidirectional method
responses = client.streaming_recognize(requests=request_generator())

# 3. Process the stream of responses
for response in responses:
    for result in response.results:
        print(f"Transcript: {result.alternatives[0].transcript}")
```

