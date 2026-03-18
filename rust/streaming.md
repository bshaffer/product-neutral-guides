# Google Cloud Rust Client Library: Streaming

gRPC Streaming allows continuous data flow between client and server. In Rust, this is implemented
using `tonic::Streaming` and `futures::Stream`.

## Streaming Types

| Type | Description | Common Rust Use Case |
| :--- | :--- | :--- |
| **Server-Side** | Client sends one request; Server sends a stream of messages. | Reading large datasets (BigQuery, Spanner) or watching logs. |
| **Client-Side** | Client sends a stream of messages; Server waits for stream to close before sending a response. | Uploading large files or ingesting bulk data. |
| **Bidirectional** | Both Client and Server send a stream of messages independently. | Real-time audio processing (Speech-to-Text). |

## Server-Side Streaming Example

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

## gRPC Bidirectional Streaming

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