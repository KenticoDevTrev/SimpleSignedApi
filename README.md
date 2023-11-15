# SimpleSignedApi
A simple library to handle signed and secure APIs for limited applications

# Generate Public and Private Access Keys
Use the ApiSignedRequestHelper to generate your public and private keys.  

Store these values in a safe location.  Only the Public Key should be shared so users can sign their requests.

The default length of the RSA key is 2048 bits, you can modify this, however be careful as if your key is too small it may not be able to encrypt your signature.

``` csharp
var generatedKeys = ApiSignedRequestHelper.GeneratePublicPrivateRSAKeys();
// This key should be shared
var publicKey = generatedKeys.PublicXmlKey;

// This key is private and should not be shared
var privateKey = generatedKeys.PrivateXmlKey;

```

# Generate Access Token
Users will need to provide their access token (their identifier).  To generate, you can use the `ApiSignedRequestHelper.GenerateAccessKey()`.  Default length is 32, you can modify this but be careful if you do as this will increase the signature that needs to be encrypted and may cause an error during encryption if it's too large.

``` csharp
// This is an access token for a given user, and is their "ID".  Save this and give this to the user
var accessToken = ApiSignedRequestHelper.GenerateAccessKey();

```

# Create your Request Model
If using C#, your request model should inherit from `IApiSignedRequestObject` and should be serializable.  The `GetRequestKey()` should return a string that identifies the unique request (usually concatenate all the values into one big string), so that if any value changes that the Request Key would be different.

The object should also have a constructor that allows it to be deserialized into

``` csharp

[Serializable]
public class IncreaseUserPointsRequest : IApiSignedRequestObject
{
    public IncreaseUserPointsRequest(int increasePoints, string username)
    {
        IncreasePoints = increasePoints;
        Username = username;
    }

    [JsonPropertyName("increasePoints")]
    public int IncreasePoints { get; }

    [JsonPropertyName("username")]
    public string Username { get; }

    public string GetRequestKey()
    {
        return $"{IncreasePoints}|{Username}";
    }
}

```

# Sending a Request
Now that the model exists (can be shared with users), you can create your request object, then use the `ApiSignedRequest<T>.CreateRequest` to generate the request to send.

``` csharp

// The data you wish to send to the API
var requestObject = new IncreaseUserPointsRequest(increasePoints: 10, username: "foo-bar");

// Create your request with the object, your access token, and the public key
var signedRequest = ApiSignedRequest<IncreaseUserPointsRequest>.CreateRequest(requestObject, accessToken, publicKey);

// Post object
var client = new HttpClient();
await client.PostAsJsonAsync("https://theapi.url/api/increase-user-points", signedRequest);
// Or 
// await client.PostAsync("https://theapi.url/api/increase-user-points", new StringContent(JsonSerializer.Serialize(signedRequest), Encoding.UTF8, "application/json"));

```

## Sending a request from non C# sources
The Public Key has within it the `Modulus` and `Exponent` values, which are what are needed to encrypt.  Searching for how to RSA encrypt with a given language using these values often will reveal code snippets on how to perform this.

You will need to document / tell the client how to construct the Signature so they can properly create the JSON structure to match.

The Signature logic in C# is the Base64 RSA Encoded string:
`{accessToken}|{UTC Date in format MM/dd/yyyy HH:mm:ss TT}|{MD5 hash of the request's Key}|{salt value, usually 8 random numbers/characters}`

The actual JSON string should follow this structure:
```
{
    "request": { "increasePoints": 10, "username": "foo-bar"},
    "date":"11/15/2023 10:25:35 PM","signature":"dZt8RKr3koklPMKD36bqcazKmSfpEQWefQxSWSSNXiEScEZCN537yzvmAY1pfCTQb1GARQIe0LoPuq8ay4/VZMzri8cgbazN5BUEqSoWBpgx13TYVt8ZPxyPmGW5SD9eLPJpw8h6EyoIq\u002BIOOyLWTsBOSCxWh/INHSABJdymZRY="
}
```

# Receiving Request and Validation
On the receiving end, either get the JSON String and deserialize it, or receive as the actual `ApiSignedRequest<T>` and pass it to the `ApiSignedRequestProcessor` along with the private key, then check for errors and retrieve your data.

``` csharp
// Or have your API convert the request to the ApiSignedRequest<T> Right away, then pass
var processor = new ApiSignedRequestProcessessor<IncreaseUserPointsRequest>(theApiSignedRequestObject, privateKey);
// Or
// Parse the JSON String
// var processor = new ApiSignedRequestProcessessor<IncreaseUserPointsRequest>(requestJson, privateKey);

// Check if Successful
if (!processor.Successful)
{
    Console.WriteLine(processor.Error);
    return;
}

// Check access token
if (!_myAccessTokenValidator.IsValidAccessToken(processor))
{
    Console.WriteLine("Access token is not recognized");
    return;
}

// Only necessary if nullable checking, as at this point the processessor.RequestObject will exist.
if (processor.RequestObject == null)
{
    Console.WriteLine("Some reason the object is null, this shouldn't be.");
    return;
}

// You now can use the object as needed
var request = processor.RequestObject;
//request.IncreasePoints;
//request.Username;

```