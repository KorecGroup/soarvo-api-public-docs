# Soarvo API - Quick Start Guide
This guide contains a quick run through of the main endpoints you will need to use the Soarvo API end to end (create a project, create a location, create a feature type, create a feature, update a feature).

## Authentication
The Soarvo API uses AWS Cognito to authenticate a user both for API access and application access using a username (your email) and your password. How to use this can vary depending on what Cognito based library you are using. You will need to use this to get access to the **access token, id token, and refresh token. The id token is what will be used as the value for the authorization header for all API requests**. 

The access token and id token has a 60 minute expiration time and the refresh token has a 30 day expiration time. 

Below is the Cognito configuration required to configure your Cognito client from your chosen library, separated by environment:
```JSON
{
  "Dev": {
    "SoarvoApiBaseUrl": "https://api.dev.soarvo.com",
    "SoarvoCdnBaseUrl": "https://dev.soarvo.com",
    "CognitoUserPoolId": "eu-west-1_h0GBMcDC3",
    "CognitoUserPoolClientId": "5r3bgn5hf0udm2b34hci75cll7",
    "CognitoRegion": "eu-west-1"
  },
  "Uat": {
    "SoarvoApiBaseUrl": "https://api.uat.soarvo.com",
    "SoarvoCdnBaseUrl": "https://uat.soarvo.com",
    "CognitoUserPoolId": "eu-west-1_mGNFaogh7",
    "CognitoUserPoolClientId": "4fip8amj0p9q94lbc1oh5997op",
    "CognitoRegion": "eu-west-1"
  },
  "Live": {
    "SoarvoApiBaseUrl": "https://api.live.soarvo.com",
    "SoarvoCdnBaseUrl": "https://live.soarvo.com",
    "CognitoUserPoolId": "eu-west-1_q2ZrQ7DwG",
    "CognitoUserPoolClientId": "7p1d45f76v2lquet47bttnaj0k",
    "CognitoRegion": "eu-west-1"
  }
}

```

 Below is a list of the common libraries used for multiple programming languages and environments:

### Cognito Authentication Libraries (for Access Token)

#### JavaScript / TypeScript

* [`amazon-cognito-identity-js`](https://www.npmjs.com/package/amazon-cognito-identity-js) – Core library to sign in and get tokens.
* [`@aws-amplify/auth`](https://www.npmjs.com/package/@aws-amplify/auth) – Higher-level wrapper over Cognito flows.

#### Python

* [`boto3`](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/cognito-idp.html) – Direct access to `initiate_auth` and token retrieval.
* [`warrant`](https://github.com/capless/warrant) – Simplifies authentication/token retrieval with Cognito.

#### Java

* [`aws-java-sdk-cognitoidp`](https://mvnrepository.com/artifact/com.amazonaws/aws-java-sdk-cognitoidp) – Use for user authentication and token handling.

#### C\#

* [`AWSSDK.CognitoIdentityProvider`](https://www.nuget.org/packages/AWSSDK.CognitoIdentityProvider) – Provides `InitiateAuthAsync` and access to tokens.

#### Go

* [`aws-sdk-go-v2/service/cognitoidentityprovider`](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/cognitoidentityprovider) – Use for token-based auth flows.

#### Ruby

* [`aws-sdk-cognitoidentityprovider`](https://rubygems.org/gems/aws-sdk-cognitoidentityprovider) – Enables `admin_initiate_auth` and access token use.

#### PHP

* [`aws/aws-sdk-php`](https://github.com/aws/aws-sdk-php) – Use `CognitoIdentityProviderClient::initiateAuth`.

### C# Authentication Code Example
Below is a code example of retrieving the authentication details in C#:
```C#
    public class SoarvoAWSCognitoAuth
    {

        private AmazonCognitoIdentityProviderClient _provider;
        private readonly CognitoUserPool _userPool;
        private readonly SoarvoAWSCognitoAuthConfig _config;

        public SoarvoAWSCognitoAuth(SoarvoAWSCognitoAuthConfig config)
        {
            _config = config;
            _provider = new AmazonCognitoIdentityProviderClient(new AnonymousAWSCredentials(), _config.RegionEndpoint);
            _userPool = new CognitoUserPool(_config.CognitoUserPoolId, _config.CognitoUserPoolClientId, _provider);
            _config = config;
        }

        public async Task<SoarvoAuthResult> LoginInAsync(string username, string password)
        {
            try
            {
                var user = new CognitoUser(username, _config.CognitoUserPoolClientId, _userPool, _provider);

                var authRequest = new InitiateSrpAuthRequest()
                {
                    Password = password
                };

                var authResponse = await user.StartWithSrpAuthAsync(authRequest);
                return new SoarvoAuthResult
                {
                    IsSuccess = true,
                    AccessToken = authResponse.AuthenticationResult.AccessToken,
                    IdToken = authResponse.AuthenticationResult.IdToken,
                    RefreshToken = authResponse.AuthenticationResult.RefreshToken,
                    ExpiresIn = authResponse.AuthenticationResult.ExpiresIn,
                    DateTimeIssuedUtc = DateTime.UtcNow
                };
            }
            catch (NotAuthorizedException)
            {
                return new SoarvoAuthResult { IsSuccess = false, ErrorMessage = "Invalid username or password." };
            }
            catch (UserNotFoundException)
            {
                return new SoarvoAuthResult { IsSuccess = false, ErrorMessage = "Username does not exist." };
            }
            catch (UserNotConfirmedException)
            {
                return new SoarvoAuthResult { IsSuccess = false, ErrorMessage = "User is not confirmed. Please confirm your account." };
            }
            catch (PasswordResetRequiredException)
            {
                return new SoarvoAuthResult { IsSuccess = false, ErrorMessage = "Password reset required. Please reset your password." };
            }
            catch (InvalidParameterException ex)
            {
                return new SoarvoAuthResult { IsSuccess = false, ErrorMessage = $"Invalid parameter: {ex.Message}" };
            }
            catch (Exception ex)
            {
                return new SoarvoAuthResult { IsSuccess = false, ErrorMessage = $"An unexpected error occurred: {ex.Message}" };
            }
        }

        public async Task<SoarvoAuthResult> RefreshAccessTokenAsync(string refreshToken)
        {
            try
            {
                var refreshAuthRequest = new InitiateAuthRequest
                {
                    AuthFlow = AuthFlowType.REFRESH_TOKEN_AUTH,
                    ClientId = _config.CognitoUserPoolClientId,
                    AuthParameters = new Dictionary<string, string>
                    {
                        { "REFRESH_TOKEN", refreshToken }
                    }
                };

                var refreshAuthResponse = await _provider.InitiateAuthAsync(refreshAuthRequest);
                var newRefreshToken = string.IsNullOrWhiteSpace(refreshAuthResponse.AuthenticationResult.RefreshToken) ? refreshToken : refreshAuthResponse.AuthenticationResult.RefreshToken;

                return new SoarvoAuthResult
                {
                    IsSuccess = true,
                    AccessToken = refreshAuthResponse.AuthenticationResult.AccessToken,
                    IdToken = refreshAuthResponse.AuthenticationResult.IdToken,
                    RefreshToken = newRefreshToken,
                    ExpiresIn = refreshAuthResponse.AuthenticationResult.ExpiresIn,
                    DateTimeIssuedUtc = DateTime.UtcNow
                };
            }
            catch (Exception ex)
            {
                return new SoarvoAuthResult { IsSuccess = false, ErrorMessage = $"An error occurred while refreshing the access token: {ex.Message}" };
            }
        }
    }
```

You can then add the `IdToken` to the authorization header, like in the example below using a HTTP Handler:
```C#
    public class HttpClientSoarvoBearerTokenHandler : DelegatingHandler
    {
        private readonly ISoarvoAuthStorage _tokenCache;

        public HttpClientSoarvoBearerTokenHandler(ISoarvoAuthStorage tokenCache)
        {
            _tokenCache = tokenCache;
        }

        protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
        {
            var authDetails = await _tokenCache.GetCurrentUsersSoarvoAuthDetailsOrNull();

            if (!string.IsNullOrEmpty(authDetails?.IdToken))
            {
                request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", authDetails.IdToken);
            }

            return await base.SendAsync(request, cancellationToken);
        }
    }
```

## API Walkthrough
The Soarvo API is split into different services, which are:
- **Projects Service** - CRUD (create, read, update, delete) operations for Soarvo projects and locations.
- **Features Service** - CRUD operations for Soarvo features and feature types.
- **Assets Service** - Manages file uploads, tracks storage usage, and integrates with geospatial workflows by abstracting S3 multi-part uploads for efficient front-end handling of large files.

This guide will not cover every endpoint and how to use them, but instead will cover the most requested endpoints. The below documentation will have examples within [the postman collection you can find here. You will need to download the .json file and import it into Postman](./Soarvo%20API%20Walkthrough.postman_collection.json). You will need to create an environment variable of 'BEARER_TOKEN' with your Cognito Id token and replace the GUIDs in the URLs with GUIDs from the relevant resource that the URL is for (project, location, feature type, etc).

### Get Projects
---

#### Endpoint

GET [https://api.live.soarvo.com/projects/current-user/projects?sort-by-property=Name&sort-by-direction=Asc&offset=0&limit=20](https://api.live.soarvo.com/projects/current-user/projects?sort-by-property=Name&sort-by-direction=Asc&offset=0&limit=20)

#### Query Parameters

| Parameter          | Value   | Description                                              |
|--------------------|---------|----------------------------------------------------------|
| sort-by-property   | Name    | The field to sort the projects by                        |
| sort-by-direction  | Asc     | The sort order: `Asc` (ascending) or `Desc` (descending) |
| offset             | 0       | The index to start fetching records from                 |
| limit              | 20      | The maximum number of records to return                  |

#### Headers

| Header Name      | Value                                            | Description                          |
|------------------|--------------------------------------------------|--------------------------------------|
| accept           | application/json, text/plain, */*               | Expected response format             |
| authorization    | Bearer <your_token>                             | Bearer token for authentication      |

> 🔐 Replace `<your_token>` with your actual JWT token.

### Get Locations
---

#### Endpoint

GET [https://api.live.soarvo.com/projects/current-user/locations?sort-by-property=Name&sort-by-direction=Asc&offset=0&limit=20](https://api.live.soarvo.com/projects/current-user/locations?sort-by-property=Name&sort-by-direction=Asc&offset=0&limit=20)

#### Query Parameters

| Parameter          | Value   | Description                                              |
|--------------------|---------|----------------------------------------------------------|
| sort-by-property   | Name    | The field to sort the locations by                      |
| sort-by-direction  | Asc     | The sort order: `Asc` (ascending) or `Desc` (descending) |
| offset             | 0       | The index to start fetching records from                 |
| limit              | 20      | The maximum number of records to return                  |

#### Headers

| Header Name      | Value                                            | Description                          |
|------------------|--------------------------------------------------|--------------------------------------|
| accept           | application/json, text/plain, */*               | Expected response format             |
| authorization    | Bearer <your_token>                             | Bearer token for authentication      |

> 🔐 Replace `<your_token>` with your actual JWT token.

### Get Features as GeoJSON by Location
---

#### Endpoint

GET [https://api.live.soarvo.com/feature-service/projects/27457463-5a4e-47f8-bbeb-4a83413e4ed9/locations/434b0047-80f9-42db-9d58-c1d5086e6a7f/legacy-mobile-app-data/get-features-json-bylocation](https://api.live.soarvo.com/feature-service/projects/27457463-5a4e-47f8-bbeb-4a83413e4ed9/locations/434b0047-80f9-42db-9d58-c1d5086e6a7f/legacy-mobile-app-data/get-features-json-bylocation)

#### Path Parameters

| Parameter     | Example Value                                         | Description                       |
|---------------|-----------------------------------------------|-----------------------------------|
| projectId     | 27457463-5a4e-47f8-bbeb-4a83413e4ed9          | ID of the project                 |
| locationId    | 434b0047-80f9-42db-9d58-c1d5086e6a7f          | ID of the location                |

#### Headers

| Header Name      | Value                                            | Description                          |
|------------------|--------------------------------------------------|--------------------------------------|
| accept           | application/json, text/plain, */*               | Expected response format             |
| authorization    | Bearer <your_token>                             | Bearer token for authentication      |

> 🔐 Replace `<your_token>` with your actual JWT token.


### Upsert Feature in Location by ID
---

#### Endpoint

PUT [https://api.live.soarvo.com/feature-service/projects/27457463-5a4e-47f8-bbeb-4a83413e4ed9/locations/434b0047-80f9-42db-9d58-c1d5086e6a7f/legacy-mobile-app-data/features/fa2166c7-fd25-401c-affb-e912c1791db9](https://api.live.soarvo.com/feature-service/projects/27457463-5a4e-47f8-bbeb-4a83413e4ed9/locations/434b0047-80f9-42db-9d58-c1d5086e6a7f/legacy-mobile-app-data/features/fa2166c7-fd25-401c-affb-e912c1791db9)

#### Method

PUT

#### Path Parameters

| Parameter     | Value                                         | Description                       |
|---------------|-----------------------------------------------|-----------------------------------|
| projectId     | 27457463-5a4e-47f8-bbeb-4a83413e4ed9          | ID of the project                 |
| locationId    | 434b0047-80f9-42db-9d58-c1d5086e6a7f          | ID of the location                |
| featureId     | fa2166c7-fd25-401c-affb-e912c1791db9          | ID of the feature to update       |

#### Headers

| Header Name      | Value                   | Description                         |
|------------------|-------------------------|-------------------------------------|
| Content-Type     | application/json        | Format of request body              |
| Authorization    | Bearer `<your_token>`   | Bearer token for authentication     |

> 🔐 Replace `<your_token>` with your actual JWT token.

#### Request Body (JSON) for a Point Feature

```json
{
  "feature_type_name": "testfeaturetype",
  "geometry": {
    "type": "Point",
    "coordinates": [-2.9948647, 53.4038483, 60.599998474121094]
  },
  "custom_attributes": {
    "_accuracy": "123.140",
    "testattribute": "ggg",
    "_fieldUser": "soarvotest"
  }
}
```

#### Request Body (JSON) for a Linear Feature
```JSON
{
  "feature_type_name": "Line_Feature",
  "geometry": {
    "type": "LineString",
    "coordinates": [
      [24.945831, 60.192059, 0.0],
      [24.946831, 60.193059, 0.0],
      [24.947831, 60.194059, 0.0],
      [24.948831, 60.195059, 0.0]
    ]
  },
  "custom_attributes": {
    "name": "Pipeline Route",
    "material": "steel",
    "diameter": 300,
    "installation_date": "2024-01-15",
    "length": 245.8,
    "status": "operational"
  }
}
```

#### Request Body (JSON) for a Polygon Feature
```json
{
  "feature_type_name": "Polygon_Feature",
  "geometry": {
    "type": "Polygon",
    "coordinates": [
      [
        [24.945831, 60.192059, 0.0],
        [24.948831, 60.192059, 0.0],
        [24.948831, 60.195059, 0.0],
        [24.945831, 60.195059, 0.0],
        [24.945831, 60.192059, 0.0]
      ]
    ]
  },
  "custom_attributes": {
    "name": "Construction Site",
    "area": 25000.5,
    "classification": "industrial",
    "owner": "Example Company Ltd",
    "permit_number": "P2025-1234",
    "start_date": "2025-05-01",
    "end_date": "2026-08-31"
  }
}
```
#### Important Notes
- A feature type with the name in 'feature_type_name' must already exist in the project.


