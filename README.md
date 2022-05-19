# Contents

- [AWS Authorization and Authentication Tutorial](#aws-authorization-and-authentication-tutorial)
  - [Background Knowledge](#background-knowledge)
    - [JSON Web Tokens](#json-web-tokens)
    - [Cognito JWTs](#cognito-jwts)
    - [ID Token](#id-token)
    - [Access Token](#access-token)
    - [Refresh Token](#refresh-token)
  - [Lambda Authorizer](#lambda-authorizer)
    - [Lambda Authorizer Input Sample](#lambda-authorizer-input-sample)
    - [Lambda Authorizer Output Sample](#lambda-authorizer-output-sample)
    - [Verifying tokens](#verifying-tokens)
      - [Verify structure of token](#verify-structure-of-token)
      - [Verify signature](#verify-signature)
        - [1. Decode token](#1-decode-token)
        - [2. Compare local key ID (kid) to public key ID](#2-compare-local-key-id--kid--to-public-key-id)
          - [Sample jwks.json](#sample-jwksjson)
        - [3. Compare signature of the issuer to the signature of the tokens](#3-compare-signature-of-the-issuer-to-the-signature-of-the-tokens)
      - [Verify the claims](#verify-the-claims)
  * [Scenario: Multi-tenant purchase tracking microservice](#scenario--multi-tenant-purchase-tracking-microservice)
    - [Tenant Isolation](#tenant-isolation)
      - [Tenant ID](#tenant-id)
      - [Lambda Context Objects](#lambda-context-objects)
      - [Multi-tenant DynamoDB table](#multi-tenant-dynamodb-table)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

# AWS Authorization and Authentication Tutorial

### Setup

###### a. [Install aws cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

###### b. Install aws-cdk cli

    npm install -g aws-cdk             # install latest version
    npm install -g aws-cdk@X.YY.Z      # install specific version

### Deploy

    cdk bootstrap
    npm run deploy

### Testing

###### Using aws cli, add users to the userpool for testing purposes

###### admin user

```
  aws cognito-idp admin-update-user-attributes \
  --user-pool-id YOUR_USER_POOL_ID \
  --username <enter_email_here> \
  --user-attributes Name="custom:tenantId",Value="GA-3fvj" \
     Name="custom:org",Value="galactic empire" \
     Name="given_name", Value="Anakin" \
     Name="custom:group",Value="admin" \
     Name="family_name",Value="Skywalker"

```

###### regular user

```
  aws cognito-idp admin-update-user-attributes \
  --user-pool-id YOUR_USER_POOL_ID \
  --username <enter_2nd_email_here> \
  --user-attributes Name="custom:tenantId",Value="GA-3fvj" \
     Name="custom:org",Value="galactic empire" \
     Name="given_name", Value="Sheev" \
     Name="custom:group",Value="user" \
     Name="family_name",Value="Sidious"

```

###### Add .env to your project's root

    Region="xxxxxx"
    userPoolId="xxxxxxx"
    COGNITO_USER_NAME='xxxxxxx'
    COGNITO_USER_PASSWORD='xxxxxxxx'
    COGNITO_CLIENT_ID='xxxxxxxxx'

## Background Knowledge

###

For new developers, it is not immediately obvious that there is a difference between the terms Authorization and Authentication. Although related, these terms actually refer to two different concepts. Authentication verifies a user's identity whereas authorization verifies what a user is allowed to access once they have been authenticated. In short, authentication is about who is allowed in and authorization is about what they are allowed to access once they are in.

In this write up I'll demonstrate how one might go about using Cognito to develop the authentication and authorization layer in a contextualised scenario.

### JSON Web Tokens

JWT, JSON Web Token, is an open standard that is widely used to securely share authentication information (claims) between client and server. The standard is defined in the RFC7519 specification developed by the Internet Engineering Taskforce (IETF).

Valid JWTs contains three sections that are encoded as base64url strings separated by dot characters as shown below

```

1. Header
2. Payload
3. Signature

<Header>.<Payload>.<Signature>
```

### Cognito JWTs

AWS has adopted and adapted the RFC7519 standard for use with the Cognito service.
When a user successfully authenticates with cognito, cognito does the following -

1. creates a session
2. returns (3) JWTs - access token, id token and refresh token.

The tokens created by cognito can be used to grant access to server-side resources such as API Gateway resource paths. Alternatively they can be exchanged for temporary AWS credentials in order to access other AWS services.

### ID Token

An ID token is a JWT that contains claims related to the identity of the authenticated user i.e email, phone number and custom attributes. When used to authenticate users of a web app, the signature of the token must be verified before the claims stored in the token can be trusted.

### Access Token

An access token is a JWT that contains claims related to the authenticated user's groups and scopes. Access tokens are similar to id tokens with very few exceptions. For example ID tokens allow the use of custom attributes whereas access tokens do not. To get a full understanding of what access tokens are and how they differ from id tokens refer to the the following resources.

- [using access tokens](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-the-access-token.html)

- [using id tokens](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-the-id-token.html)

### Refresh Token

A refresh token is used to retrieve new access tokens. Refresh tokens have a default expiration of 30 days after a user signs into the designated userpool. This can be manually configured while creating an app for the userpool.

When a refresh token expires, the user must re-authenticate by signing in again.

## Lambda Authorizer

There are two types of Lambda Authorizers, `REQUEST` based and `TOKEN` based. This write up focuses on the latter.

A token based lambda authorizer receives the caller's identity in the form of a bearer token included in the request's header section while a request based lambda authorizer receives the caller's identity in a combination of headers and query string parameters.

When a request is received by an API gateway instance that is configured to use a `TOKEN` lambda authorizer for authorization purposes, the `bearer token` contained in the request header is forwarded to the lambda authorizer for verification. The forwarded payload is a JSON object that assumes a structure similar to the one shown in the `Input Sample` code block shown below .

###### Lambda Authorizer Input Sample

```
{
    "type":"TOKEN",
    "authorizationToken":"{caller-supplied-token}",
    "methodArn":"arn:aws:execute-api:{regionId}:{accountId}:{apiId}/{stage}/{httpVerb}/[{resource}/[{child-resources}]]"
}
```

###### Lambda Authorizer Output Sample

Once the token is verified, the Lambda Authorizer should return an output that assumes a structure such as the one provided below.

```
{
  "principalId": "yyyyyyyy", // The principal user identification associated with the token sent by the client.
  "policyDocument": {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Action": "execute-api:Invoke",
        "Effect": "<Allow|Deny>",
        "Resource": "<aws resource>"
      }
    ]
  },
  "context": {
    "key": "value",
  },
  "usageIdentifierKey": "{api-key}"
}
```

- The principalId is the user id associated with the token sent by the client.
- If the API uses a usage plan and the apiKeySource is set to AUTHORIZER, the lambda authorizer output must include the usage plan's API keys as the `usageIdentifierKey` property value- The principalId is the user id associated with the token sent by the client.
- If the API uses a usage plan and the apiKeySource is set to AUTHORIZER, the lambda authorizer output must include the usage plan's API keys as the `usageIdentifierKey` property value

### Verifying tokens

Token verification is done in 3 steps.

1. Verify structure of token
2. Verify signature
3. Verify the claims

#### Verify structure of token

Confirm that the token contains three dot separated base64url strings. If the token does not conform to this structure then it is invalid.
The first string is a header string followed by a payload string and then finally the signature string as shown below.

```
<Header>.<Payload>.<Signature>
```

#### Verify signature

##### 1. Decode token

    To validate the JWT signature, the token must first be decoded.

##### 2. Compare local key ID (kid) to public key ID

&nbsp;&nbsp;&nbsp;&nbsp;
i) Download and store your JWT's corresponding `JWK` (JSON Web Key) using the following url

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
`https://cognito-idp.{region}.amazonaws.com/{userPoolId}/.well-known/jwks.json`

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
substitute region and userPoolId with your user pool's region and user pool ID respectively

&nbsp;&nbsp;&nbsp;&nbsp;
ii) Search the downloaded `jwks.json` for a `kid` that matches the `kid` of your `JWT`

###### Sample jwks.json

```
        {
            "keys": [{
                "kid": "1234example=",
                "alg": "RS256",
                "kty": "RSA",
                "e": "AQAB",
                "n": "1234567890",
                "use": "sig"
            }, {
                "kid": "5678example=",
                "alg": "RS256",
                "kty": "RSA",
                "e": "AQAB",
                "n": "987654321",
                "use": "sig"
            }]
        }
```

##### 3. Compare signature of the issuer to the signature of the tokens

The signature of the issuer is derived from the JWK with a kid that matches the kid of the JWT.
Each `JWK` contains an `n` parameter that contains the `modulus value` of the `RSA public key`.
This is the value that'll be used to derive the `issuer's signature`.
The JWK will need to be `converted to PEM format` before that can happen.

#### Verify the claims

1. Verify that the token is not expired.

2. The aud claim in an ID token and the client_id claim in an access token should match the app client ID that was created in the Amazon Cognito user pool.

3. The issuer (iss) claim should match your user pool. For example, a user pool created in the us-east-1 Region will have the following iss value:`https://cognito-idp.us-east-1.amazonaws.com/<userpoolID>`

4. Check the `token_use` claim.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; If you are only accepting the access token in your web API operations, its value must be access.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; If you are only using the ID token, its value must be id.

## Scenario: Multi-tenant purchase tracking microservice

Consider a scenario where we'd like to build an e-commerce web application. To keep things simple let's contextualize the scenario so that we only have one micro service that uses a multi tenant dynamoDb table to store/retrieve customer purchases. This microservice can potentially be split into two read/write lambdas. An architectural diagram for this scenario has been provided below.
![image](architecture.png)

1. Users authenticate with a username and password, the web app passes these to amazon cognito for validation.
2. If the supplied credentials (username and password) are valid, cognito creates a session and subsequently issues three (3) JWTs (JSON Web Tokens). The aforementioned tokens are id token, access token and a refresh token. The authenticated user can now send requests to api gateway along with with the id token in the headers section.
3. API gateway sends the received id token to a lambda function called an authorizer.
4. The authorizer function verifies the claims attached to the id token.
5. The authorizer returns a policy and context.
6. API Gateway evaluates the policy and forwards the request to a lambda function along with the authorizer generated context.
7. The lambda function writes/reads data according to the tenantId listed in the forwarded context.
8. A response is returned by the lambda function.

### Tenant Isolation

#### Tenant ID

To create a secure `multi-tenant` environment there needs to be some notion of `tenant resource isolation`. In essence, `tenant A` should not be able to access the resources of `tenant B` and vice versa. To solve this problem it was decided that it is necessary to assign a unique identifier called `tenantID` to each user/customer. The aforementioned `tenantID` is a composite string that is generated and then attached to the calling user as a `custom attribute` during the registration stage.

After successful authentication the `tenantID` custom attribute becomes available as a parameter within the `ID token` as a parameter with the following key `custom:tenantID`. It is important to note that access tokens do not carry any of the user's custom attributes, only id tokens have this capability. In saying so,the application will exclusively use id tokens for authorization.

#### Lambda Context Objects

In order to make use of the idea of a `tenantID` it is important to realise that there needs to be some way of retrieving the correct `tenantID` parameter and propagating it to the running lambda function's execution scope. This is where the idea of Lambda context objects comes into play. When the Lambda service invokes a function, it passes a context object to the function handler.

Context objects can be modified to include custom parameters that can then be accessed during run time. With this in mind, it should be relatively straight forward to use the application's Lambda Authorizer to forward the retrieved `tenantID` as a custom parameter within the JSON output of the lambda authorizer.

#### Multi-tenant DynamoDB table

In the context of dynamoDb, the tenant ID will essentially be the partition key that'll be used to group together/ partition customer data. When a customer needs to write/retrieve data to/from the purchase history database they'll use their unique tenant ID to only access data that belongs to them. In essence the tenant ID can be thought of as being analogous to a key that can only open a single door.

The table provided below is a representation of how the data stored in the purchase history will be partitioned. The parameter `tenantId` is used as the `partition key` while the `dateTimePurchased` parameter is used as the `sort key`.

|     tenantID     | dateTimePurchased | products |
| :--------------: | :---------------: | :------: |
| `Customer1-xcv9` | `dd-mm-yyTh:m:sZ` |    []    |
| `Customer2-dgf1` | `dd-mm-yyTh:m:sZ` |    []    |
| `Customer1-xcv9` | `dd-mm-yyTh:m:sZ` |    []    |
| `Customer2-dgf1` | `dd-mm-yyTh:m:sZ` |    []    |

### Allow/Deny API Gateway Traffic

When the application's Lambda Authorizer is invoked it is expected to return a JSON object that not only includes a context object that contains the `tenantID` parameter but also a resource policy as detailed [here](#lambda-authorizer-output-sample). The policy should allow/deny access to the API depending on the following.

1. The outcome of the Lambda Authorizer's token claims verification
2. The type of user (admin/user)

#### Lambda Authorizer Resource Policies

###### Token Verification failed

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:${region}:${accountId}:*/*/*/purchaseHistory"
        },
    ]
}
```

###### Token Verification passed - Admin

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:${region}:${accountId}:*/*/*/purchaseHistory"
        },
    ]
}
```

###### Token Verification passed - User

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:${region}:${accountId}:*/*/GET/purchaseHistory"
        },
    ]
}
```
