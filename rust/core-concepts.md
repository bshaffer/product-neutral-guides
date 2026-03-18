# Google Cloud Rust Client Library: Core Concepts

This documentation covers essential patterns and usage for the Google Cloud Rust client libraries, focusing on performance (gRPC), data handling (Protobuf, Update Masks), and flow control (Pagination, LROs, Streaming).

## 1. Pagination

Most list methods in the Google Cloud Rust libraries return a `Stream`. This allows you to iterate over results asynchronously without manually managing page tokens.

The standard way to handle pagination is to use the `next()` method from the `StreamExt` trait. The library automatically fetches new pages in the background as you consume the stream.

```rust
use google_cloud_secretmanager::v1::client::SecretManagerClient;
use google_cloud_secretmanager::v1::model::ListSecretsRequest;
use futures_util::StreamExt;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = SecretManagerClient::new().await?;

    // Prepare the request
    let request = ListSecretsRequest {
        parent: "projects/my-project".to_string(),
        ..Default::default()
    };

    // Call the API
    // This returns a Stream of secrets
    let mut stream = client.list_secrets(request).await?;

    // Automatically fetches subsequent pages of secrets
    while let Some(secret) = stream.next().await {
        let secret = secret?;
        println!("Secret: {}", secret.name);
    }

    Ok(())
}
```

### Manual Pagination (Accessing Tokens)

If you need to control pagination manually (e.g., for a web API that sends tokens to a frontend), you can access the `next_page_token` from the raw response.

```rust
// Prepare request with page size and optional token
let request = ListSecretsRequest {
    parent: "projects/my-project".to_string(),
    page_size: 10,
    page_token: Some(current_page_token),
    ..Default::default()
};

// Call the method to get a single page response
let response = client.list_secrets_raw(request).await?;

for secret in response.secrets {
    // Process current page items
}

// Get the token for the next page
let next_token = response.next_page_token;
```

## 2. Long Running Operations (LROs)

Some operations, like creating a Compute Engine instance or training an AI model, take too long to complete in a single request. These return a **Long Running Operation (LRO)**.

The Rust library provides an `Operation` handle to manage these.

### Polling for Completion

The standard pattern is to use the `wait()` method, which asynchronously polls the operation until it finishes.

```rust
use google_cloud_compute::v1::client::InstancesClient;
use google_cloud_compute::v1::model::InsertInstanceRequest;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = InstancesClient::new().await?;

    // Prepare the Request object
    let request = InsertInstanceRequest {
        project,
        zone,
        instance_resource: Some(instance_resource),
        ..Default::default()
    };

    // Call the method with the request object
    let mut operation = client.insert(request).await?;

    // Wait for the operation to complete
    // This polls periodically without blocking the executor
    let result = operation.wait(None).await?;

    match result {
        Ok(instance) => println!("Instance created: {:?}", instance),
        Err(status) => println!("Error: {:?}", status),
    }

    Ok(())
}
```

### Async / Non-Blocking Check

If you don't want to wait immediately, you can store the Operation Name and check it later.

```rust
// Start operation
let operation = client.long_running_method(...).await?;
let operation_name = operation.name().to_string();

// ... later, or in a different task ...

// Resume operation
let mut operation = client.get_operation(&operation_name).await?;

if operation.is_done() {
    // Handle success
}
```

## 3. Update Masks

When updating resources (PATCH requests), Google Cloud APIs often use an **Update Mask** (`prost_types::FieldMask`). This tells the server *exactly* which fields you intend to update, preventing accidental overwrites of other fields.

If you do not provide a mask, some APIs update **all** fields, resetting missing ones to default values.

### Constructing a FieldMask

```rust
use google_cloud_secretmanager::v1::client::SecretManagerClient;
use google_cloud_secretmanager::v1::model::{Secret, UpdateSecretRequest};
use prost_types::FieldMask;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = SecretManagerClient::new().await?;

    // Prepare the resource with NEW values
    let secret = Secret {
        name: "projects/my-project/secrets/my-secret".to_string(),
        labels: [("env".to_string(), "production".to_string())].into(),
        ..Default::default()
    };

    // Create the FieldMask
    // Paths must match the protobuf field names (snake_case)
    let update_mask = FieldMask {
        paths: vec!["labels".to_string()],
    };

    // Prepare the Request object
    let request = UpdateSecretRequest {
        secret: Some(secret),
        update_mask: Some(update_mask),
    };

    // Call the API
    client.update_secret(request).await?;

    Ok(())
}
```

**Note:** The field names in `paths` should be the protocol buffer field names (usually `snake_case`), which typically match the field names in the generated Rust structs.

## 4. Protobuf and gRPC

The Google Cloud Rust libraries are built on top of **gRPC** and **Protobuf**.

* **Protobuf (Protocol Buffers):** A mechanism for serializing structured data. It is the interface language for gRPC, handled in Rust by the `prost` crate.

* **gRPC:** A high-performance, open-source universal RPC framework. It uses HTTP/2 for transport and is the default for high-performance communication with Google Cloud, handled in Rust by the `tonic` crate.

### Installation & Setup

To use the client libraries, add the desired service crate to your `Cargo.toml`. Most clients include the necessary gRPC and Protobuf infrastructure automatically.

```toml
[dependencies]
google-cloud-secretmanager = "0.1.0"
tokio = { version = "1", features = ["full"] }
futures-util = "0.3"
prost = "0.12"
prost-types = "0.12"
```

### Usage in Client

The client library uses gRPC by default. The transport layer is managed internally, but you can configure connection settings like retry policies and timeouts during client initialization.

```rust
use google_cloud_pubsub::v1::client::PublisherClient;
use google_cloud_gax::conn::Environment;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // The client uses gRPC transport by default
    let client = PublisherClient::new().await?;

    Ok(())
}
```

## 5. gRPC Streaming

gRPC Streaming allows continuous data flow between client and server. In Rust, this is implemented using `tonic::Streaming` and `futures::Stream`.

### Streaming Types

| Type | Description | Common Rust Use Case |
| :--- | :--- | :--- |
| **Server-Side** | Client sends one request; Server sends a stream of messages. | Reading large datasets (BigQuery, Spanner) or watching logs. |
| **Client-Side** | Client sends a stream of messages; Server waits for stream to close before sending a response. | Uploading large files or ingesting bulk data. |
| **Bidirectional** | Both Client and Server send a stream of messages independently. | Real-time audio processing (Speech-to-Text). |

### Server-Side Streaming Example

A common example is reading rows from BigQuery Storage. The client returns a stream that you can iterate over.

```rust
use google_cloud_bigquery_storage::v1::client::BigQueryReadClient;
use google_cloud_bigquery_storage::v1::model::ReadRowsRequest;
use futures_util::StreamExt;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut client = BigQueryReadClient::new().await?;

    let request = ReadRowsRequest {
        read_stream: "projects/my-proj/locations/us/sessions/session-id/streams/stream-id".to_string(),
        ..Default::default()
    };

    // read_rows is a server-side streaming method
    let mut stream = client.read_rows(request).await?;

    // Read from the stream
    while let Some(response) = stream.next().await {
        let response = response?;
        if let Some(rows) = response.avro_rows {
            let row_data = rows.serialized_binary_rows;
            println!("Row size: {} bytes", row_data.len());
        }
    }

    Ok(())
}
```

### gRPC Bidirectional Streaming

Bidirectional streaming involves sending a stream of requests and receiving a stream of responses.

```rust
use google_cloud_speech::v2::client::SpeechClient;
use google_cloud_speech::v2::model::{StreamingRecognizeRequest, StreamingRecognitionConfig, RecognitionConfig};
use futures_util::stream;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut client = SpeechClient::new().await?;

    // Create a stream for sending requests
    let (tx, rx) = tokio::sync::mpsc::channel(10);

    // Initial Configuration Request
    let config_request = StreamingRecognizeRequest {
        recognizer: recognizer_name,
        streaming_request: Some(StreamingRequest::StreamingConfig(StreamingRecognitionConfig {
            config: Some(RecognitionConfig {
                // ... configuration ...
                ..Default::default()
            }),
            ..Default::default()
        })),
    };
    tx.send(config_request).await?;

    // Send Audio Data
    let audio_request = StreamingRecognizeRequest {
        streaming_request: Some(StreamingRequest::Audio(audio_bytes)),
        ..Default::default()
    };
    tx.send(audio_request).await?;

    // Call the bidirectional method
    let mut response_stream = client.streaming_recognize(rx).await?;

    // Read responses from the stream
    while let Some(response) = response_stream.next().await {
        let response = response?;
        for result in response.results {
            if let Some(alt) = result.alternatives.get(0) {
                println!("Transcript: {}", alt.transcript);
            }
        }
    }

    Ok(())
}
```