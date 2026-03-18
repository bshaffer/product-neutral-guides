# Authentication

The recommended way to authenticate to the Google Cloud Node.js library is to use
[Application Default Credentials (ADC)](https://cloud.google.com/docs/authentication/application-default-credentials),
which discovers your credentials automatically, based on the environment where your code is running.
To review all of your authentication options see [Credential Lookup](#credential-lookup).

For more information about authentication at Google, see [the authentication guide](https://cloud.google.com/docs/authentication).
Specific instructions and environment variables for each individual service are linked from the README documents listed below for each service.

## Application Default Credentials (ADC)

The Google Cloud Node.js library provides several mechanisms to configure your system without providing
**Service Account Credentials** directly in code. These are known as Application Default Credentials.

**Credentials** are discovered in the following order:

1. Credentials specified in code
2. Path to credential file in environment variables
3. Credentials specified in a local ADC file
4. Credentials from an attached service account (for code running on Google Cloud Platform)

### Environment Variables

The **Credentials JSON** can be placed in environment variables instead of
declaring them directly in code.

```javascript
const path = require('path');
process.env.GOOGLE_APPLICATION_CREDENTIALS = path.join(__dirname, 'your-credentials-file.json');
```

Here are the environment variables that Google Cloud Node.js checks for credentials:

1. `GOOGLE_APPLICATION_CREDENTIALS` - Path to JSON file

The JSON file can contain credentials created for
[workload identity federation](https://cloud.google.com/iam/docs/workload-identity-federation),
[workforce identity federation](https://cloud.google.com/iam/docs/workforce-identity-federation), or a
[service account key](https://cloud.google.com/docs/authentication/provide-credentials-adc#local-key).

Note: Service account keys are a security risk if not managed correctly. You should
[choose a more secure alternative to service account keys](https://cloud.google.com/docs/authentication#auth-decision-tree)
whenever possible.

### Local ADC file

This option allows for an easy way to authenticate in a local environment during development. If
credentials are not provided in code or in environment variables, then your user credentials can be
discovered from your local ADC file.

To set up a local ADC file:

1. [Download, install, and initialize the Cloud SDK](https://cloud.google.com/sdk)
2. Create your local ADC file:
    ```sh
    gcloud auth application-default login
    ```

3. Write code as if already authenticated.

**NOTE:** Because this method relies on your user credentials, it is _not_ recommended for running
in production.

### Google Cloud Platform environments

While running on Google Cloud Platform environments such as Google Compute Engine, Google App Engine
and Google Kubernetes Engine, no extra work is needed. The **Credentials** are discovered
automatically from the attached service account. Code should be written as if already authenticated.

For more information, see
[Set up ADC for Google Cloud services](https://cloud.google.com/docs/authentication/provide-credentials-adc#attached-sa).

### Project ID detection

Some libraries support setting up the project ID via the `GOOGLE_CLOUD_PROJECT` environment variable.
```javascript
process.env.GOOGLE_CLOUD_PROJECT = '<YOUR_PROJECT_ID>';
```
The libraries that support this environment variable are:
- Bigtable
- PubSub
- Storage
- Spanner
- BigQuery
- Datastore
- Firestore
- Logging
- Trace
- Translate

## Credentials Options

Each Google Cloud Node.js client may be authenticated in code when creating a client library instance.
Most clients use the `keyFilename` or `credentials` option for providing explicit credentials:

```javascript
const {VideoIntelligenceServiceClient} = require('@google-cloud/video-intelligence');

// Authenticating with a keyfile path
const video = new VideoIntelligenceServiceClient({
  keyFilename: '/path/to/service-account.json'
});

// Authenticating with keyfile data
const credentials = require('/path/to/service-account.json');
const videoWithData = new VideoIntelligenceServiceClient({
  credentials: credentials
});
```

Specific clients like Storage also follow this pattern:

```javascript
const {Storage} = require('@google-cloud/storage');

// Create the storage client and pass in the keyfile path
const storage = new Storage({
  keyFilename: '/path/to/keyfile.json'
});
```

A list of clients that accept these parameters are:

- [BigQuery](https://github.com/googleapis/nodejs-bigquery)
- [Logging](https://github.com/googleapis/nodejs-logging)
- [Storage](https://github.com/googleapis/nodejs-storage)

We recommend checking the documentation for the client library you're using for more in depth information.

## API Keys

[API keys][api_keys] are a great way to quickly authenticate in a local environment during development. If
you'd like to authenticate your client with API keys, use the `apiKey` client option when creating a new
instance of your client:

```javascript
const {RecaptchaEnterpriseServiceClient} = require('@google-cloud/recaptcha-enterprise');

// Create a client
const client = new RecaptchaEnterpriseServiceClient({
  apiKey: 'your-api-key',
});

async function listKeys() {
  // Prepare the request message
  const projectPath = client.projectPath('your-project-id');
  const request = {
    parent: projectPath,
  };

  // Call the API
  const [response] = await client.listKeys(request);
  console.log(response);
}

listKeys();
```

[api_keys]: https://cloud.google.com/docs/authentication/api-keys

## Troubleshooting

If you're having trouble authenticating open a
[Github Issue](https://github.com/googleapis/google-cloud-node/issues/new?title=Authentication+question)
to get help. Also consider searching or asking
[questions](http://stackoverflow.com/questions/tagged/google-cloud-platform+node.js) on
[StackOverflow](http://stackoverflow.com). See [Troubleshooting](DEBUG.md) for more details.