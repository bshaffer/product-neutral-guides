# Authentication

The recommended way to authenticate to the Google Cloud Python library is to use
[Application Default Credentials (ADC)](https://cloud.google.com/docs/authentication/application-default-credentials),
which discovers your credentials automatically, based on the environment where your code is running.
To review all of your authentication options see [Credential Lookup](#credential-lookup).

For more information about authentication at Google, see [the authentication guide](https://cloud.google.com/docs/authentication).
Specific instructions and environment variables for each individual service are linked from the README documents listed below for each service.

## Application Default Credentials (ADC)

The Google Cloud Python library provides several mechanisms to configure your system without providing
**Service Account Credentials** directly in code. These are known as Application Default Credentials.

**Credentials** are discovered in the following order:

1. Credentials specified in code
2. Path to credential file in environment variables
3. Credentials specified in a local ADC file
4. Credentials from an attached service account (for code running on Google Cloud Platform)

### Environment Variables

The **Credentials JSON** can be placed in environment variables instead of
declaring them directly in code.

```python
import os
os.environ["GOOGLE_APPLICATION_CREDENTIALS"] = "/path/to/your-credentials-file.json"
```

Here are the environment variables that Google Cloud Python checks for credentials:

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

## Credentials Options

Each Google Cloud Python client may be authenticated in code when creating a client library instance.
Most clients use the `credentials` argument for providing explicit credentials:

```python
from google.cloud import videointelligence_v1
from google.oauth2 import service_account

# Authenticating with keyfile path.
credentials = service_account.Credentials.from_service_account_file(
    '/path/to/service-account.json'
)
video_client = videointelligence_v1.VideoIntelligenceServiceClient(
    credentials=credentials
)
```

For some services, you might initialize a high-level client object:

```python
from google.cloud import storage
from google.oauth2 import service_account

# Create the service account credentials and pass them in
credentials = service_account.Credentials.from_service_account_file(
    '/path/to/keyfile.json'
)
storage_client = storage.Client(credentials=credentials)
```

A list of clients that commonly use these patterns includes:

- [BigQuery](https://github.com/googleapis/google-cloud-python/tree/main/packages/google-cloud-bigquery)
- [Logging](https://github.com/googleapis/google-cloud-python/tree/main/packages/google-cloud-logging)
- [Storage](https://github.com/googleapis/google-cloud-python/tree/main/packages/google-cloud-storage)

We recommend checking the [client documentation][python-ref-docs] for the client library you're using
for more in-depth information.

## API Keys

[API keys][api_keys] are a great way to quickly authenticate in a local environment during development. If
you'd like to authenticate your client with API keys, use the `api_key` argument within `client_options` when creating a new
instance of your client:

```python
from google.cloud import recaptchaenterprise_v1

# Create a client.
client_options = {"api_key": "your-api-key"}
client = recaptchaenterprise_v1.RecaptchaEnterpriseServiceClient(
    client_options=client_options
)

# Prepare the request message.
project_name = f"projects/{project_id}"
request = recaptchaenterprise_v1.ListKeysRequest(parent=project_name)

# Call the API
response = client.list_keys(request=request)
```

[api_keys]: https://cloud.google.com/docs/authentication/api-keys

## Troubleshooting

If you're having trouble authenticating open a
[Github Issue](https://github.com/googleapis/google-cloud-python/issues/new?title=Authentication+question)
to get help. Also consider searching or asking
[questions](http://stackoverflow.com/questions/tagged/google-cloud-platform+python) on
[StackOverflow](http://stackoverflow.com). See [Troubleshooting](DEBUG.md) for more details.