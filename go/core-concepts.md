# Google Cloud Go Client Library: Core Concepts

This documentation covers essential patterns and usage for the Google Cloud Go Client Library, focusing on performance (gRPC), data handling (Protobuf, Update Masks), and flow control (Pagination, LROs, Streaming).

## 1. Pagination

Most list methods in the Google Cloud Go library return an iterator. This allows you to iterate over results without manually managing page tokens.

The easiest way to handle pagination is to call `Next()` on the iterator in a loop. The library automatically fetches new pages in the background as you iterate.

```go
import (
	"context"
	"fmt"

	secretmanager "cloud.google.com/go/secretmanager/apiv1"
	"cloud.google.com/go/secretmanager/apiv1/secretmanagerpb"
	"google.golang.org/api/iterator"
)

ctx := context.Background()
client, err := secretmanager.NewClient(ctx)
if err != nil {
	// Handle error
}
defer client.Close()

// Prepare the request
req := &secretmanagerpb.ListSecretsRequest{
	Parent: "projects/my-project",
}

// Call the API
// This returns an iterator
it := client.ListSecrets(ctx, req)

// Automatically fetches subsequent pages of secrets
for {
	resp, err := it.Next()
	if err == iterator.Done {
		break
	}
	if err != nil {
		// Handle error
		break
	}
	fmt.Printf("Secret: %s\n", resp.GetName())
}
```

### Manual Pagination (Accessing Tokens)

If you need to control pagination manually (e.g., for a web API that sends tokens to a frontend), you can access the `PageToken` and `NextPageToken`.

```go
import (
	"cloud.google.com/go/secretmanager/apiv1/secretmanagerpb"
	"google.golang.org/api/iterator"
)

// Prepare request with page size and optional token
req := &secretmanagerpb.ListSecretsRequest{
	Parent:   "projects/my-project",
	PageSize: 10,
}

// Check if we have a token from a previous request
if queryToken := r.URL.Query().Get("page_token"); queryToken != "" {
	req.PageToken = queryToken
}

it := client.ListSecrets(ctx, req)
pager := iterator.NewPager(it, int(req.PageSize), req.PageToken)

var secrets []*secretmanagerpb.Secret
// Fetch the specific page
nextToken, err := pager.NextPage(&secrets)
if err != nil {
    // Handle error
}

for _, secret := range secrets {
	// Process current page items
}

// nextToken is the token for the next page (empty string if no more pages)
fmt.Printf("Next token: %s\n", nextToken)
```

## 2. Long Running Operations (LROs)

Some operations, like creating a Compute Engine instance or training an AI model, take too long to complete in a single request. These return a **Long Running Operation (LRO)**.

The Go library provides specific Operation types to manage these.

### Polling for Completion

The standard pattern is to call the `Wait()` method on the operation object.

```go
import (
	compute "cloud.google.com/go/compute/apiv1"
	computepb "cloud.google.com/go/compute/apiv1/computepb"
)

ctx := context.Background()
client, err := compute.NewInstancesRESTClient(ctx)
if err != nil {
	// Handle error
}
defer client.Close()

// Prepare the Request object
req := &computepb.InsertInstanceRequest{
	Project:          project,
	Zone:             zone,
	InstanceResource: instanceResource,
}

// Call the method with the request object
op, err := client.Insert(ctx, req)
if err != nil {
	// Handle error
}

// Wait for the operation to complete
// This blocks the goroutine, polling periodically
err = op.Wait(ctx)
if err != nil {
	// Handle error
}

// If successful, you can often retrieve the metadata or the result
// The specific return type depends on the service
fmt.Println("Operation completed successfully")
```

### Async / Non-Blocking Check

If you don't want to block the current execution, you can store the Operation Name and check it later.

```go
// Start operation
op, err := client.LongRunningMethod(ctx, req)
if err != nil {
    // Handle error
}
opName := op.Name()

// ... later, or in a different worker process ...

// Resume operation
// Each service usually provides a way to recreate an operation from its name
newOp := client.Operation(opName)

if newOp.Done() {
	// Handle success
}
```

## 3. Update Masks

When updating resources (PATCH requests), Google Cloud APIs often use an **Update Mask** (`fieldmaskpb.FieldMask`). This tells the server *exactly* which fields you intend to update, preventing accidental overwrites of other fields.

If you do not provide a mask, some APIs update **all** fields, resetting missing ones to default values.

### Constructing a FieldMask

```go
import (
	secretmanager "cloud.google.com/go/secretmanager/apiv1"
	"cloud.google.com/go/secretmanager/apiv1/secretmanagerpb"
	"google.golang.org/protobuf/types/known/fieldmaskpb"
)

client, err := secretmanager.NewClient(ctx)

// Prepare the resource with NEW values
secret := &secretmanagerpb.Secret{
	Name:   "projects/my-project/secrets/my-secret",
	Labels: map[string]string{"env": "production"}, // We only want to update this field
}

// Create the FieldMask
// Paths MUST match the protobuf field names (snake_case)
updateMask := &fieldmaskpb.FieldMask{
	Paths: []string{"labels"},
}

// Prepare the Request object
req := &secretmanagerpb.UpdateSecretRequest{
	Secret:     secret,
	UpdateMask: updateMask,
}

// Call the API
_, err = client.UpdateSecret(ctx, req)
```

**Note:** The field names in `Paths` should be the protocol buffer field names (usually `snake_case`), even if the Go struct fields use PascalCase.

## 4. Protobuf and gRPC

The Google Cloud Go library is built natively with **gRPC** and **Protobuf**.

* **Protobuf (Protocol Buffers):** A mechanism for serializing structured data. It is the interface language for gRPC.

* **gRPC:** A high-performance, open-source universal RPC framework. It is the primary transport for Go clients due to efficient binary serialization and HTTP/2 support.

### Installation & Setup

Go manages dependencies via modules. You don't need external extensions; simply use `go get` to add the necessary cloud libraries to your project.

```bash
# Add the core cloud package and specific service packages
go get cloud.google.com/go/secretmanager/apiv1
go get google.golang.org/protobuf
go get google.golang.org/grpc
```

### Usage in Client

The client library uses gRPC by default. In cases where a service supports both gRPC and REST, the library usually provides separate constructors or defaults to gRPC for optimal performance.

```go
import (
	"cloud.google.com/go/pubsub"
	"google.golang.org/api/option"
)

// Default gRPC client
client, err := pubsub.NewClient(ctx, "project-id")

// If you specifically need to use a REST-based client (where supported)
// some services offer specific NewRESTClient constructors.
```

## 5. gRPC Streaming

gRPC Streaming allows continuous data flow between client and server. In Go, this is handled through stream objects that provide `Send()` and `Recv()` methods.

### Streaming Types

| Type | Description | Common Go Use Case |
| :--- | :--- | :--- |
| **Server-Side** | Client sends one request; Server sends a stream of messages. | Reading large datasets (BigQuery Storage) or watching logs. |
| **Client-Side** | Client sends a stream of messages; Server waits for stream to close before sending a response. | Uploading large files or ingesting bulk data. |
| **Bidirectional** | Both Client and Server send a stream of messages independently. | Real-time audio processing (Speech-to-Text). |

### Server-Side Streaming

An example of this is the **BigQuery Storage API**. The `ReadRows` method returns a stream that you can iterate through.

```go
import (
	"context"
	"io"

	storage "cloud.google.com/go/bigquery/storage/apiv1"
	"cloud.google.com/go/bigquery/storage/apiv1/storagepb"
)

client, err := storage.NewBigQueryReadClient(ctx)

// Streaming requires a valid read stream name
req := &storagepb.ReadRowsRequest{
	ReadStream: "projects/my-proj/locations/us/sessions/session-id/streams/stream-id",
}

// readRows is a server-side streaming method
stream, err := client.ReadRows(ctx, req)

// Read from the stream
for {
	resp, err := stream.Recv()
	if err == io.EOF {
		break
	}
	if err != nil {
		// Handle error
		break
	}
	// Process binary row data
	rowData := resp.GetAvroRows().GetSerializedBinaryRows()
	fmt.Printf("Row size: %d bytes\n", len(rowData))
}
```

### gRPC Bidirectional Streaming

If you are using **Cloud Speech-to-Text**, you will interact with a stream object that allows you to write requests and read responses concurrently.

The protocol requires you to send a configuration request first, followed by audio data requests.

```go
import (
	speech "cloud.google.com/go/speech/apiv2"
	"cloud.google.com/go/speech/apiv2/speechpb"
)

client, err := speech.NewClient(ctx)

// streamingRecognize is a bidirectional streaming method
stream, err := client.StreamingRecognize(ctx)

// Send the Initial Configuration Request
configReq := &speechpb.StreamingRecognizeRequest{
	Recognizer: recognizerName,
	StreamingRequest: &speechpb.StreamingRecognizeRequest_StreamingConfig{
		StreamingConfig: &speechpb.StreamingRecognitionConfig{
			Config: &speechpb.RecognitionConfig{
				DecodingConfig: &speechpb.RecognitionConfig_ExplicitDecodingConfig{
					ExplicitDecodingConfig: &speechpb.ExplicitDecodingConfig{
						Encoding:          speechpb.ExplicitDecodingConfig_LINEAR16,
						SampleRateHertz:   16000,
						AudioChannelCount: 1,
					},
				},
			},
		},
	},
}

stream.Send(configReq)

// Send Audio Data Request(s)
audioReq := &speechpb.StreamingRecognizeRequest{
	StreamingRequest: &speechpb.StreamingRecognizeRequest_Audio{
		Audio: audioBlob,
	},
}
stream.Send(audioReq)

// Close the sending side of the stream
stream.CloseSend()

// Read responses from the stream
for {
	resp, err := stream.Recv()
	if err == io.EOF {
		break
	}
	if err != nil {
		// Handle error
		break
	}
	for _, result := range resp.Results {
		fmt.Printf("Transcript: %s\n", result.Alternatives[0].Transcript)
	}
}
```