# Testing with Postman

As mentioned in the [Testing Utilities](../core/testing.md) Postman is an option for automated tests and much more.

## Problem with manual testing

If you just use Swagger UI with JWT and Bearer as Auth-Mode, you have todo a lot of repetitive work.
Let us take the following example with manual testing.
You make a change at the code and add an attribute to an entity.

- [x] First you load/reload the Swagger UI
- [x] Send your Credentials to the authentication Endpoint (/authentication_token).
- [x] Copy the token from the response (maybe save it in a temporary file).
- [x] Now you click on "Authorize" in Swagger and enter "Bearer [TOKEN]" and submit the form.
- [x] **Finally search the correct route, enter the params and check if everything is fine**

Make a new change and start all tasks from the beginning. Boring, isn't it?

## Solution: Postman with automatic authentication

The solution is to automate authentication and testing. One Option is Postman with automatic authentication.
The following steps are necessary for authentication with JWT in Postman.

1. Follow this guide to [add the TTL of the JWT-Tokens to the authentication Endpoint](../core/jwt.md#improve-return-values-of-the-jwt-token-endpoint).

2. Export OpenApi-Schemata from API-Platform

        symfony console api:openapi:export -o "openapi.json"

3. [Import Data in Postman](https://learning.postman.com/docs/getting-started/importing-and-exporting-data/)

4. Set Auth Type of the Collection to `Bearer Token` and the Token value to `{{AuthTokenVar}}`

5. Add Pre-Request Scripts. See section below.

6. Add/Set Collection Variables according to your settings.

   | Variable            | InitialValue                                  |
   | ------------------- |---------------------------------------------- |
   | baseUrl             | <https://localhost/>                          |
   | AuthUrl             | <https://localhost/authentication_token>      |
   | AuthTokenVar        |                                               |
   | AuthToken_CreatedAt |                                               |
   | AuthToken_ExpiresIn |                                               |
   | AuthEmail           |                                               |
   | AuthPassword        |                                               |

### Pre-Request Scripts for authentication with JWT

[Pre-Request Scripts](https://learning.postman.com/docs/writing-scripts/pre-request-scripts/), do exactly what the name says. The run before every request.
With that help the Auth Token is generated and renewed automatically.

There are different [variable scopes](https://learning.postman.com/docs/sending-requests/variables/#variable-scopes) in Postman.
In this code only collectionVariables are used for simplicity.

```js
var tokenCreatedAt = pm.collectionVariables.get("AuthToken_CreatedAt");

// ensure correct initial setup (yesterday)
if (!tokenCreatedAt) {
    tokenCreatedAt = new Date(new Date().setDate(new Date().getDate() - 1))
}

var tokenExpiresIn = pm.collectionVariables.get("AuthToken_ExpiresIn");

if (!tokenExpiresIn) {
    tokenExpiresIn = 3600; // Default: 1 hour
}

var tokenCreatedTime = (new Date() - Date.parse(tokenCreatedAt))

if (tokenCreatedTime >= (tokenExpiresIn*1000)) {

    console.log("The token has expired. Attempting to request a new token.");

    pm.sendRequest({
        url: pm.collectionVariables.get("AuthUrl"),
        method: 'POST',
        header: {
            "Accept": "application/json",
            "Content-Type": "application/json"
        },
        body: {
            mode: 'raw',
            raw: JSON.stringify(
                {
                    "email": pm.collectionVariables.get("AuthEmail"),
                    "password": pm.collectionVariables.get("AuthPassword")
                }
            )
        }
    }, function(error, response) {

        var token = response.json().token;
        var tokenExpiresIn = response.json().data.token_ttl; // in seconds

        if(token)
        {
            pm.collectionVariables.set("AuthToken_CreatedAt", new Date());
            pm.collectionVariables.set("AuthTokenVar", response.json().token);
            pm.collectionVariables.set("AuthToken_ExpiresIn", tokenExpiresIn);
        }
    });
}
```
