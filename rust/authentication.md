# Authentication

The recommended way to authenticate to the Google Cloud Rust libraries is to use
[Application Default Credentials (ADC)](https://cloud.google.com/docs/authentication/application-default-credentials),
which discovers your credentials automatically, based on the environment where your code is running.
To review all of your authentication options see [Credential Lookup](#credential-lookup).

For more information about authentication at Google, see [the authentication guide](https://cloud.google.com/docs/authentication).
Specific instructions and environment variables for each individual service are linked from the documentation for each crate.

## Application Default Credentials (ADC)

The Google Cloud Rust libraries provide several mechanisms to configure your system without providing
**Service Account Credentials** directly in code. These are known as Application Default Credentials.

**Credentials** are discovered in the following order:

1. Credentials specified in code
2. Path to credential file in environment variables
3. Credentials specified in a local ADC file
4. Credentials from an attached service account (for code running on Google Cloud Platform)

### Environment Variables

The **Credentials JSON** can be placed in environment variables instead of
declaring them directly in code.

```rust
std::env::set_var("GOOGLE_APPLICATION_CREDENTIALS", "/path/to/your-credentials-file.json");
```

Here are the environment variables that Google Cloud Rust checks for credentials:

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
```rust
std::env::set_var("GOOGLE_CLOUD_PROJECT", "<YOUR_PROJECT_ID>");
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

Each Google Cloud Rust client may be authenticated in code when creating a client library instance.
Most clients allow providing explicit credentials via a configuration object:

```rust
use google_cloud_auth::credentials::CredentialsFile;
use google_cloud_videointelligence::v1::video_intelligence_service_client::VideoIntelligenceServiceClient;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let scopes = vec!["https://www.googleapis.com/auth/cloud-platform"];
    let creds = CredentialsFile::new_from_file("/path/to/service-account.json").await?;

    // Authenticating with keyfile data.
    let config = VideoIntelligenceClientConfig::default()
        .with_credentials(creds)
        .with_scopes(scopes);

    let video = VideoIntelligenceServiceClient::new(config).await?;

    Ok(())
}
```

Some clients use a specific configuration builder or config struct for providing credentials:

```rust
use google_cloud_storage::client::{Client, ClientConfig};
use google_cloud_auth::credentials::CredentialsFile;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create the service account credentials and pass them in using the config
    let scopes = vec!["https://www.googleapis.com/auth/devstorage.read_only"];
    let creds = CredentialsFile::new_from_file("/path/to/keyfile.json").await?;

    let config = ClientConfig::default()
        .with_credentials(creds)
        .with_scopes(scopes);

    let storage = Client::new(config);

    Ok(())
}
```

Common crates that accept these parameters include:

- [BigQuery](https://github.com/khulnasoft/google-cloud-rust/tree/main/bigquery)
- [Logging](https://github.com/khulnasoft/google-cloud-rust/tree/main/logging)
- [Storage](https://github.com/khulnasoft/google-cloud-rust/tree/main/storage)

We recommend checking the crate documentation for the specific library you're using
for more in-depth information.

## API Keys

[API keys][api_keys] are a great way to quickly authenticate in a local environment during development. If
you'd like to authenticate your client with API keys, use the API key option when creating a new
instance of your client:

```rust
use google_cloud_recaptcha_enterprise::v1::recaptcha_enterprise_service_client::RecaptchaEnterpriseServiceClient;
use google_cloud_recaptcha_enterprise::v1::ListKeysRequest;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create a client.
    let config = Config::default().with_api_key(your_api_key);
    let mut client = RecaptchaEnterpriseServiceClient::new(config).await?;

    // Prepare the request message.
    let request = ListKeysRequest {
        parent: format!("projects/{}", "[PROJECT]"),
        ..Default::default()
    };

    // Call the API
    let response = client.list_keys(request).await?;

    Ok(())
}
```

[api_keys]: https://cloud.google.com/docs/authentication/api-keys

## Troubleshooting

If you're having trouble authenticating, open a
[Github Issue](https://github.com/khulnasoft/google-cloud-rust/issues/new)
to get help. Also consider searching or asking
[questions](http://stackoverflow.com/questions/tagged/google-cloud-platform+rust) on
[StackOverflow](http://stackoverflow.com). See the troubleshooting guide for more details.