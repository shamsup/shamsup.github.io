---
title: Calculating the Secret Hash for AWS Cognito in Node.js
# author: shamsup
date: 2024-08-09 14:52:00 -0700
description: Use all the features of the Cognito API from Node.js by calculating the secret hash for your user pool.
# tags: [aws, cognito, nodejs, javascript]
---

While Amplify and the Cognito client libraries don't support user pools with a client secret, this is only to ensure that the client secret isn't exposed in the browser. However, this doesn't mean that you can't use the full Cognito API from Node.js.

Recently I was attempting to use the Cognito API from a Node.js Lambda function to customize our signup flow, but kept getting the error `SecretHash does not match for the client` when trying to sign up users. Some digging led me to the [Cognito documentation](https://docs.aws.amazon.com/cognito/latest/developerguide/signing-up-users-in-your-app.html#cognito-user-pools-computing-secret-hash), which contains some example code in Java, but otherwise just some pseudo-code to go off of for generating the secret hash:

> The `SecretHash` value is a Base 64-encoded keyed-hash message authentication code (HMAC) calculated using the secret key of a user pool client and username plus the client ID in the message. The following pseudocode shows how this value is calculated. In this pseudocode, `+` indicates concatenation, `HMAC_SHA256` represents a function that produces an HMAC value using HmacSHA256, and `Base64` represents a function that produces Base-64-encoded version of the hash output.
>
> ```
> Base64 ( HMAC_SHA256 ( "Client Secret Key", "Username" + "Client Id" ) )
> ```

HMAC is an authentication technique that allows multiple parties to verify that another party knows a shared secret key. This is used often for things like verifying the authenticity of data in cookies and JWTs by generating a signature, or verifying the origin of webhooks. This makes sense as a simple verification method for AWS to use since only our app and Cognito should know the client secret, and HTTPS is already encrypting the request.

I know that Node.js has the `crypto` module built-in, which I've used in the past to generate SHA-256 hashes, but I have never used a _specific key_ to do it. Time to dig into the [`crypto` docs](https://nodejs.org/docs/latest-v20.x/api/crypto.html#cryptocreatehmacalgorithm-key-options)! Luckily for us, the Node.js `crypto` module follows a similar "builder" pattern as the Java implementation in the Cognito documentation referenced earlier.

Here's how we can create an HMAC value using the SHA-256 algorithm in node:

```javascript
import { createHmac } from "crypto";

// create the hmac with the sha256 algorithm and a secret key
const hasher = createHmac("sha256", "a secret");

// add the value we want to hash
hasher.update("value to hash");

// get the hashed value as base64
const output = hasher.digest("base64");
```

We can plug in our User Pool values where needed to create our `SecretHash`. I'm using the [`SignUp`](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_SignUp.html) method as an example, but it's the same for `ConfirmSignUp`, `ForgotPassword`, `ConfirmForgotPassword`, and `ResendConfirmationCode` (and probably more). I'm also using the AWS SDK for JavaScript v2 which is what is included in the Lambda Node.js runtime, but the secret hash is generated the same way for v3 of the SDK.

```javascript
import { createHmac } from "crypto";
import { CognitoIdentityServiceProvider } from "aws-sdk";

// grab all the constant variables from the user pool
const CLIENT_SECRET = process.env.COGNITO_CLIENT_SECRET;
const CLIENT_ID = process.env.COGNITO_CLIENT_ID;
const USER_POOL_ID = process.env.COGNITO_USER_POOL_ID;

function signUp(username, password, attributes) {
  const cognito = new CognitoIdentityServiceProvider();

  const hasher = createHmac("sha256", CLIENT_SECRET);
  // AWS wants `"Username" + "Client Id"`
  hasher.update(`${username}${CLIENT_ID}`);
  const secretHash = hasher.digest("base64");

  return cognito
    .signUp({
      UserPoolId: USER_POOL_ID,
      ClientId: CLIENT_ID,
      UserName: username,
      Password: password,
      SecretHash: secretHash,
      UserAttributes: [
        // some attributes as an example
        { Name: "email", Value: attributes.email },
        { Name: "given_name", Value: attributes.firstName },
        { Name: "family_name", Value: attributes.lastName }
      ]
    })
    .promise();
}
```
{: file='signup.js' }

AWS Cognito has a very attractive pricing model and a lot of features to build out whatever type of authentication you want for your application, but it has more than its fair share of quirks that complicate adoption. Hopefully this saves you a few hours of digging.

_Originally published on June 30, 2022 to [dev.to](https://dev.to/shamsup/creating-the-secret-hash-for-aws-cognito-in-nodejs-50f7)._
