# Authentication

The recommended way to authenticate to the {{ gcp_name }} Java client libraries is to use
[Application Default Credentials (ADC)](https://cloud.google.com/docs/authentication/application-default-credentials),
which discovers your credentials automatically, based on the environment where your code is running.
To review all of your authentication options see [Credential Lookup](#credential-lookup).

For more information about authentication at Google, see [the authentication guide](https://cloud.google.com/docs/authentication).
Specific instructions and environment variables for each individual service are linked from the README documents listed below for each service.

## Application Default Credentials (ADC)

The {{ gcp_name }} Java client libraries provide several mechanisms to configure your system without providing
**Service Account Credentials** directly in code. These are known as Application Default Credentials.

**Credentials** are discovered in the following order:

1. Credentials specified in code
2. Path to credential file in environment variables
3. Credentials specified in a local ADC file
4. Credentials from an attached service account (for code running on {{ gcp_name }} Platform)

### Environment Variables

The **Credentials JSON** can be placed in environment variables instead of
declaring them directly in code.

```java
// Environment variables are typically set in your shell or execution environment
// export GOOGLE_APPLICATION_CREDENTIALS="/path/to/your-credentials-file.json"
```

Here are the environment variables that {{ gcp_name }} Java checks for credentials:

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

### {{ gcp_name }} Platform environments

While running on {{ gcp_name }} Platform environments such as Google Compute Engine, Google App Engine
and Google Kubernetes Engine, no extra work is needed. The **Credentials** are discovered
automatically from the attached service account. Code should be written as if already authenticated.

For more information, see
[Set up ADC for {{ gcp_name }} services](https://cloud.google.com/docs/authentication/provide-credentials-adc#attached-sa).

## Credentials Options

Each {{ gcp_name }} Java client may be authenticated in code when creating a client library instance.
Most clients use a `CredentialsProvider` for providing explicit credentials:

```java
import com.google.auth.oauth2.GoogleCredentials;
import com.google.api.gax.core.FixedCredentialsProvider;
import com.google.cloud.videointelligence.v1.VideoIntelligenceServiceClient;
import com.google.cloud.videointelligence.v1.VideoIntelligenceServiceSettings;
import java.io.FileInputStream;
import java.util.Collections;

// Authenticating with keyfile data.
GoogleCredentials credentials = GoogleCredentials.fromStream(new FileInputStream("/path/to/service-account.json"))
    .createScoped(Collections.singletonList("https://www.googleapis.com/auth/cloud-platform"));

VideoIntelligenceServiceSettings settings = VideoIntelligenceServiceSettings.newBuilder()
    .setCredentialsProvider(FixedCredentialsProvider.create(credentials))
    .build();

try (VideoIntelligenceServiceClient video = VideoIntelligenceServiceClient.create(settings)) {
    // Use the client
}
```

**Note**: Some clients use a service-specific options builder instead:

```java
import com.google.auth.oauth2.ServiceAccountCredentials;
import com.google.cloud.storage.Storage;
import com.google.cloud.storage.StorageOptions;
import java.io.FileInputStream;

// Create the service account credentials and pass them in using the options builder
ServiceAccountCredentials credentials = ServiceAccountCredentials.fromStream(new FileInputStream("/path/to/keyfile.json"));
Storage storage = StorageOptions.newBuilder()
    .setCredentials(credentials)
    .build()
    .getService();
```

A list of clients that commonly use this pattern includes:

- [BigQuery](https://github.com/googleapis/google-cloud-java/tree/main/java-bigquery)
- [Logging](https://github.com/googleapis/google-cloud-java/tree/main/java-logging)
- [Storage](https://github.com/googleapis/google-cloud-java/tree/main/java-storage)

We recommend visiting the [client documentation](https://cloud.google.com/java/docs/reference) for the client library you're using for more in-depth information.

## API Keys

[API keys][api_keys] are a great way to quickly authenticate in a local environment during development. If
you'd like to authenticate your client with API keys, use the API key setting when creating a new
instance of your client:

```java
import com.google.cloud.recaptchaenterprise.v1.RecaptchaEnterpriseServiceClient;
import com.google.cloud.recaptchaenterprise.v1.RecaptchaEnterpriseServiceSettings;
import com.google.recaptchaenterprise.v1.ListKeysRequest;
import com.google.recaptchaenterprise.v1.ProjectName;

// Create a client settings with an API key.
RecaptchaEnterpriseServiceSettings settings = RecaptchaEnterpriseServiceSettings.newBuilder()
    .setApiKey(yourApiKey)
    .build();

try (RecaptchaEnterpriseServiceClient recaptchaClient = RecaptchaEnterpriseServiceClient.create(settings)) {
    // Prepare the request message.
    ProjectName parent = ProjectName.of("[PROJECT]");
    ListKeysRequest request = ListKeysRequest.newBuilder()
        .setParent(parent.toString())
        .build();

    // Call the API
    RecaptchaEnterpriseServiceClient.ListKeysPagedResponse response = recaptchaClient.listKeys(request);
}
```

[api_keys]: https://cloud.google.com/docs/authentication/api-keys

## Troubleshooting

If you're having trouble authenticating open a
[Github Issue](https://github.com/googleapis/google-cloud-java/issues/new)
to get help. Also consider searching or asking
[questions](http://stackoverflow.com/questions/tagged/google-cloud-platform+java) on
[StackOverflow](http://stackoverflow.com).