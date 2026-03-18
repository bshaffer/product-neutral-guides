# Google Cloud Node.js Client Library: Core Concepts

This documentation covers essential patterns and usage for the Google Cloud Node.js Client Library, focusing on performance (gRPC), data handling (Protobuf, Update Masks), and flow control (Pagination, LROs, Streaming).

## 1. Pagination

Most list methods in the Google Cloud Node.js library return a Promise that resolves to an array. By default, the library handles auto-pagination, meaning it will fetch all results across all pages unless you specify otherwise.

The easiest way to handle pagination is to simply use `await` on the method call. The library returns an array where the first element contains the results.

```javascript
const {SecretManagerServiceClient} = require('@google-cloud/secret-manager');

const secretManager = new SecretManagerServiceClient();

async function listSecrets() {
  const parent = 'projects/my-project';

  // Call the API
  // By default, this automatically fetches all pages of secrets
  const [secrets] = await secretManager.listSecrets({
    parent: parent,
  });

  secrets.forEach(secret => {
    console.log(`Secret: ${secret.name}`);
  });
}
```

### Manual Pagination (Accessing Tokens)

If you need to control pagination manually (e.g., for a web API that sends tokens to a frontend), you can set `autoPaginate: false` and access the `nextPageToken` from the response.

```javascript
async function listSecretsManual() {
  const options = {
    autoPaginate: false
  };
  
  const request = {
    parent: 'projects/my-project',
    pageSize: 10,
    // pageToken: 'previous_token_here'
  };

  // The method returns [resources, nextQuery, response]
  const [secrets, nextQuery, response] = await secretManager.listSecrets(request, options);

  secrets.forEach(secret => {
    // Process current page items
  });

  // Get the token for the next page
  const nextToken = response.nextPageToken;
  
  if (nextToken) {
    console.log(`Next page token: ${nextToken}`);
  }
}
```

## 2. Long Running Operations (LROs)

Some operations, like creating a Compute Engine instance or training an AI model, take too long to complete in a single HTTP request. These return a **Long Running Operation (LRO)**.

The Node.js library returns an operation object that you can use to wait for completion.

### Polling for Completion

The standard pattern is to use the `.promise()` method on the operation object.

```javascript
const compute = require('@google-cloud/compute');
const instancesClient = new compute.InstancesClient();

async function createInstance() {
  // Prepare the Request object
  const [operation] = await instancesClient.insert({
    project,
    zone,
    instanceResource,
  });

  // Wait for the operation to complete
  // This blocks the execution until the operation is done
  const [response] = await operation.promise();

  console.log('Operation complete:', response.name);
  
  // The result is usually the metadata or the resource itself
  // depending on the specific API method.
}
```

### Async / Non-Blocking Check

If you don't want to block the current process, you can store the Operation Name and check it later.

```javascript
// 1. Start operation
const [operation] = await client.longRunningMethod(...);
const operationName = operation.name;

// ... later, or in a different worker process ...

// 2. Resume operation
const [newOperation] = await client.checkOperation({name: operationName});

if (newOperation.done) {
    // Handle success
    const result = newOperation.response;
}
```

## 3. Update Masks

When updating resources (PATCH requests), Google Cloud APIs often use an **Update Mask** (`google.protobuf.FieldMask`). This tells the server *exactly* which fields you intend to update, preventing accidental overwrites of other fields.

If you do not provide a mask, some APIs update **all** fields, resetting missing ones to default values.

### Constructing a FieldMask

```javascript
const {SecretManagerServiceClient} = require('@google-cloud/secret-manager');
const client = new SecretManagerServiceClient();

async function updateSecret() {
  // 1. Prepare the resource with NEW values
  const secret = {
    name: 'projects/my-project/secrets/my-secret',
    labels: {env: 'production'}, // We only want to update this field
  };

  // 2. Prepare the Request object with the updateMask
  // 'paths' MUST match the protobuf field names (snake_case)
  const request = {
    secret,
    updateMask: {
      paths: ['labels'],
    },
  };

  // 3. Call the API
  const [response] = await client.updateSecret(request);
}
```

**Note:** The field names in `paths` should be the protocol buffer field names (usually `snake_case`), even if the Node.js object properties are typically `camelCase` in some library versions.

## 4. Protobuf and gRPC

The Google Cloud Node.js library supports transport via **gRPC** and **REST (HTTP/1.1)**.

* **Protobuf (Protocol Buffers):** A mechanism for serializing structured data. It is the interface language for gRPC.

* **gRPC:** A high-performance, open-source universal RPC framework. It is generally faster than REST due to efficient binary serialization and HTTP/2 support.

### Installation & Setup

In Node.js, gRPC support is built into the client libraries using the `@grpc/grpc-js` package. You don't need to install external C extensions like PECL.

```bash
# Install the specific client library
npm install @google-cloud/secret-manager
```

### Usage in Client

The client library automatically uses gRPC by default. You can force a specific transport using the `fallback` option (to use REST over HTTP/1.1) when creating a client.

```javascript
const {PublisherClient} = require('@google-cloud/pubsub').v1;

const publisher = new PublisherClient({
    fallback: true // Forces REST (HTTP/1.1) instead of gRPC
});
```

## 5. gRPC Streaming

gRPC Streaming allows continuous data flow between client and server. In Node.js, this is implemented using the standard **Node.js Stream API**.

### Streaming Types

| Type | Description | Common Node.js Use Case |
| :--- | :--- | :--- |
| **Server-Side** | Client sends one request; Server sends a stream of messages. | Reading large datasets (BigQuery, Spanner) or watching logs. |
| **Client-Side** | Client sends a stream of messages; Server waits for stream to close before sending a response. | Uploading large files or ingesting bulk data. |
| **Bidirectional** | Both Client and Server send a stream of messages independently. | Real-time audio processing (Speech-to-Text). |

### Server-Side Streaming Example (High-Level)

Commonly used in BigQuery or Spanner, the Node.js client exposes streaming through the standard Readable Stream interface.

```javascript
const {BigQuery} = require('@google-cloud/bigquery');
const bigquery = new BigQuery();

const query = 'SELECT * FROM `bigquery-public-data.samples.shakespeare`';

bigquery
  .createQueryStream(query)
  .on('error', console.error)
  .on('data', row => {
    console.log(row);
  })
  .on('end', () => {
    console.log('Stream finished.');
  });
```

### Server-Side Streaming Example (Low-Level)

The generated clients also support gRPC Streaming. An example of this is the **BigQuery Storage API**.

```javascript
const {BigQueryReadClient} = require('@google-cloud/bigquery-storage');
const client = new BigQueryReadClient();

async function readRows() {
  const request = {
    readStream: 'projects/my-proj/locations/us/sessions/session-id/streams/stream-id',
  };

  // readRows is a server-side streaming method
  const stream = client.readRows(request);

  stream.on('data', response => {
    // response is a ReadRowsResponse object
    const rowData = response.avroRows.serializedBinaryRows;
    console.log(`Row size: ${rowData.length} bytes`);
  });

  stream.on('error', err => {
    console.error(err);
  });
}
```

### gRPC Bidirectional Streaming

For Bidirectional APIs like **Cloud Speech-to-Text**, you interact with a `Duplex` stream. This allows you to write requests and read responses continuously.

```javascript
const {SpeechClient} = require('@google-cloud/speech').v2;
const client = new SpeechClient();

async function streamSpeech() {
  // streamingRecognize is a bidirectional streaming method
  const stream = client.streamingRecognize();

  // 1. Send the Initial Configuration Request
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

  // write the configuration
  stream.write(configRequest);

  // 2. Read responses from the stream using listeners
  stream.on('data', response => {
    response.results.forEach(result => {
      console.log(`Transcript: ${result.alternatives[0].transcript}`);
    });
  });

  // 3. Send Audio Data Request(s)
  const fs = require('fs');
  const audioBuffer = fs.readFileSync('audio.raw');
  
  stream.write({
    audio: audioBuffer,
  });

  // Signal that no more data will be written
  stream.end();
}
```

