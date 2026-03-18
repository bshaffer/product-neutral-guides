# Authentication

The recommended way to authenticate to the Google Cloud Go library is to use
[Application Default Credentials (ADC)](https://cloud.google.com/docs/authentication/application-default-credentials),
which discovers your credentials automatically, based on the environment where your code is running.
To review all of your authentication options see [Credential Lookup](#credential-lookup).

For more information about authentication at Google, see [the authentication guide](https://cloud.google.com/docs/authentication).
Specific instructions and environment variables for each individual service are linked from the README documents listed below for each service.

## Application Default Credentials (ADC)

The Google Cloud Go library provides several mechanisms to configure your system without providing
**Service Account Credentials** directly in code. These are known as Application Default Credentials.

**Credentials** are discovered in the following order:

1. Credentials specified in code via `option.WithCredentials` or `option.WithCredentialsFile`
2. Path to credential file in environment variables
3. Credentials specified in a local ADC file
4. Credentials from an attached service account (for code running on Google Cloud Platform)

### Environment Variables

The **Credentials JSON** can be placed in environment variables instead of
declaring them directly in code.

```go
import "os"

// ...
os.Setenv("GOOGLE_APPLICATION_CREDENTIALS", "/path/to/your-credentials-file.json")
```

Here are the environment variables that Google Cloud Go checks for credentials:

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

Most Go client libraries support setting up the project ID via the `GOOGLE_CLOUD_PROJECT` environment variable.
```go
import "os"

// ...
os.Setenv("GOOGLE_CLOUD_PROJECT", "<YOUR_PROJECT_ID>")
```
The libraries that support this environment variable include:
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

Each Google Cloud Go client may be authenticated in code when creating a client library instance.
Most clients use the `google.golang.org/api/option` package for providing explicit credentials:

```go
import (
    "context"
    video "cloud.google.com/go/videointelligence/apiv1"
    "google.golang.org/api/option"
)

func main() {
    ctx := context.Background()

    // Authenticating with a service account file.
    client, err := video.NewClient(ctx, option.WithCredentialsFile("/path/to/service-account.json"))
    if err != nil {
        // Handle error
    }
    defer client.Close()
}
```

If you have the JSON key file data already loaded into memory, you can use `WithCredentialsJSON` instead:

```go
import (
    "context"
    "os"
    "cloud.google.com/go/storage"
    "google.golang.org/api/option"
)

func main() {
    ctx := context.Background()

    // Load your keyfile data
    keyFileData, err := os.ReadFile("/path/to/keyfile.json")
    if err != nil {
        // Handle error
    }

    // Create the client and pass the JSON data using option.WithCredentialsJSON
    client, err := storage.NewClient(ctx, option.WithCredentialsJSON(keyFileData))
    if err != nil {
        // Handle error
    }
    defer client.Close()
}
```

All Google Cloud Go clients accept these optional parameters, including:

- [BigQuery](https://pkg.go.dev/cloud.google.com/go/bigquery)
- [Logging](https://pkg.go.dev/cloud.google.com/go/logging)
- [Storage](https://pkg.go.dev/cloud.google.com/go/storage)

We recommend checking the [Go reference documentation](https://pkg.go.dev/cloud.google.com/go) for the specific client library you're using for more in-depth information.

## API Keys

[API keys][api_keys] are a great way to quickly authenticate in a local environment during development. If
you'd like to authenticate your client with API keys, use the `option.WithAPIKey` functional option when creating a new
instance of your client:

```go
import (
    "context"
    recaptcha "cloud.google.com/go/recaptchaenterprise/v2/apiv1"
    recaptchapb "cloud.google.com/go/recaptchaenterprise/v2/apiv1/recaptchaenterprisepb"
    "google.golang.org/api/option"
)

func main() {
    ctx := context.Background()

    // Create a client with an API key.
    client, err := recaptcha.NewRecaptchaEnterpriseClient(ctx, option.WithAPIKey("your-api-key"))
    if err != nil {
        // Handle error
    }
    defer client.Close()

    // Prepare the request message.
    req := &recaptchapb.ListKeysRequest{
        Parent: "projects/[PROJECT]",
    }

    // Call the API
    it := client.ListKeys(ctx, req)
    // Iterate over results...
}
```

[api_keys]: https://cloud.google.com/docs/authentication/api-keys

## Troubleshooting

If you're having trouble authenticating, open a
[Github Issue](https://github.com/googleapis/google-cloud-go/issues/new?title=Authentication+question)
to get help. Also consider searching or asking
[questions](http://stackoverflow.com/questions/tagged/google-cloud-platform+go) on
[StackOverflow](http://stackoverflow.com). See [Troubleshooting](DEBUG.md) for more details.