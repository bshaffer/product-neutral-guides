# Google Cloud Node.js Client Library: Core Concepts

This documentation covers essential patterns and usage for the Google Cloud Node.js Client Library, focusing on performance (gRPC), data handling (Protobuf, Update Masks), and flow control (Pagination, LROs, Streaming).

## 1. Pagination

Most list methods in the Google Cloud Node.js library provide a way to iterate over results without manually managing page tokens. The libraries typically offer an asynchronous iterable pattern that handles fetching new pages automatically.

The easiest way to handle pagination is to use the `listXAsync` method or the standard list method within an asynchronous iteration. The library automatically fetches new pages in the background as you iterate.

```javascript
const {SecretManagerServiceClient} = require('@google-cloud/secret-manager');

const secretManager = new SecretManagerServiceClient();

async function listAllSecrets() {
  const parent = 'projects/my-project';

  // The listSecretsAsync method returns an async iterable
  // This automatically fetches subsequent pages of secrets
  for await (const secret of secretManager.listSecretsAsync({parent})) {
    console.log(`Secret: ${secret.name}`);
  }
}
```

### Manual Pagination (Accessing Tokens)

If you need to control pagination manually (e.g., for a web API that sends tokens to a frontend), you can access the `nextPageToken` from the response.

```javascript
// Prepare request with page size and optional token
const request = {
  parent: 'projects/my-project',
  pageSize: 10,
};

// Check if we have a token from a previous request
if (req.query.pageToken) {
  request.pageToken = req.query.pageToken;
}

// When using the standard method call without a callback, it returns a Promise
// resolving to an array: [resources, nextRequest, response]
const [secrets, nextRequest, response] = await secretManager.listSecrets(request);

for (const secret of secrets) {
  // Process current page items
}

// Get the token for the next page (null if no more pages)
const nextToken = response.nextPageToken;
```

## 2. Long Running Operations (LROs)

Some operations, like creating a Compute Engine instance or training an AI model, take too long to complete in a single HTTP request. These return a **Long Running Operation (LRO)**.

The Node.js library provides an `Operation` object to manage these.

### Polling for Completion

The standard pattern is to use the `.promise()` method on the operation object, which waits until the operation is finished.

```javascript
const {InstancesClient} = require('@google-cloud/compute').v1;

const instancesClient = new InstancesClient();

async function createInstance() {
  // Prepare the Request object
  const request = {
    project,
    zone,
    instanceResource,
  };

  // Call the method, which returns a promise for the operation
  const [operation] = await instancesClient.insert(request);

  // Wait for the operation to complete
  // This blocks the execution until the operation is done
  const [response] = await operation.promise();

  // Handle success
  console.log('Instance created:', response.targetLink);
}
```

### Async / Non-Blocking Check

If you don't want to wait for the operation in the current execution context, you can store the Operation Name and check it later.

```javascript
// Start operation
const [operation] = await client.longRunningMethod(...);
const operationName = operation.name;

// ... later, or in a different worker process ...

// Resume operation
const [newOperation] = await client.checkOperation({name: operationName});

if (newOperation.done) {
  // Handle success
  const result = newOperation.result;
}
```

## 3. Update Masks

When updating resources (PATCH requests), Google Cloud APIs often use an **Update Mask**. This tells the server *exactly* which fields you intend to update, preventing accidental overwrites of other fields.

If you do not provide a mask, some APIs update **all** fields, resetting missing ones to default values.

### Constructing a FieldMask

```javascript
const {SecretManagerServiceClient} = require('@google-cloud/secret-manager');

const client = new SecretManagerServiceClient();

async function updateSecret() {
  // Prepare the resource with NEW values
  const secret = {
    name: 'projects/my-project/secrets/my-secret',
    labels: {env: 'production'}, // We only want to update this field
  };

  // Create the updateMask
  // paths correspond to the field names in the resource
  const updateMask = {
    paths: ['labels'],
  };

  // Prepare the Request object
  const request = {
    secret,
    updateMask,
  };

  // Call the API
  const [response] = await client.updateSecret(request);
}
```

**Note:** The field names in `paths` should match the underlying API field names (usually `snake_case`), even if the object properties in JavaScript are sometimes handled as `camelCase`.

## 4. Protobuf and gRPC

The Google Cloud Node.js library uses **gRPC** as its primary transport for most services, which provides high performance through **Protobuf (Protocol Buffers)**.

* **Protobuf:** A mechanism for serializing structured data. It serves as the interface definition for gRPC services.

* **gRPC:** A high-performance RPC framework. It is typically faster than traditional REST because it uses binary serialization and takes advantage of HTTP/2 features.

### Installation & Setup

Node.js libraries include the necessary gRPC dependencies. You simply install the specific service library using `npm`.

```bash
# Install the library via npm
npm install @google-cloud/secret-manager
```

### Usage in Client

The client library uses gRPC by default. In some cases, you can configure the transport explicitly if the library supports it.

```javascript
const {PubSub} = require('@google-cloud/pubsub');

// Libraries often use a default transport, but can be configured
const pubsub = new PubSub({
  // Configuration options
});
```

## 5. gRPC Streaming

gRPC Streaming allows continuous data flow between the client and the server. In Node.js, this is implemented using the standard `stream` module, allowing for efficient data handling.

### Streaming Types

| Type | Description | Common Node.js Use Case |
| :--- | :--- | :--- |
| **Server-Side** | Client sends one request; Server sends a stream of messages. | Reading large datasets or watching real-time logs. |
| **Client-Side** | Client sends a stream of messages; Server sends one response. | Uploading large files or bulk data ingestion. |
| **Bidirectional** | Both Client and Server send a stream of messages independently. | Real-time audio processing (Speech-to-Text). |

### Server-Side Streaming Example

A common example is reading rows from a database or a storage stream. The Node.js client returns a Readable stream.

```javascript
const {BigQueryReadClient} = require('@google-cloud/bigquery-storage');

const client = new BigQueryReadClient();

async function readRows() {
  const request = {
    readStream: 'projects/my-proj/locations/us/sessions/session-id/streams/stream-id',
  };

  // The method returns a stream
  const stream = client.readRows(request);

  // Process the stream using event listeners
  stream.on('data', response => {
    // response is a ReadRowsResponse object
    const rowData = response.avroRows.serializedBinaryRows;
    console.log(`Row size: ${rowData.length} bytes`);
  });

  stream.on('error', err => {
    console.error(err);
  });

  stream.on('end', () => {
    console.log('Stream finished');
  });
}
```

### gRPC Bidirectional Streaming

For services like Cloud Speech-to-Text, you interact with a Duplex stream. This allows you to write requests and read responses simultaneously.

The protocol usually requires sending a configuration request first, followed by the data.

```javascript
const {SpeechClient} = require('@google-cloud/speech').v2;

const client = new SpeechClient();

async function streamingRecognize() {
  // streamingRecognize returns a bidirectional stream (Duplex)
  const stream = client.streamingRecognize();

  // Listen for responses
  stream.on('data', response => {
    const result = response.results[0];
    if (result && result.alternatives[0]) {
      console.log(`Transcript: ${result.alternatives[0].transcript}`);
    }
  });

  // Send the Initial Configuration Request
  const configRequest = {
    recognizer: recognizerName,
    streamingConfig: {
      config: {
        explicitDecodingConfig: {
          encoding: 'LINEAR16',
          sampleRateHertz: 16000,
          audioChannelCount: 1,
        },
      },
    },
  };

  stream.write(configRequest);

  // Send Audio Data Request(s)
  const audioData = require('fs').readFileSync('audio.raw');
  const audioRequest = {
    audio: audioData,
  };

  stream.write(audioRequest);

  // End the stream when finished
  stream.end();
}
```