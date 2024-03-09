---
title: "OAuth"
---

# OAuth

## Table of contents

- [Overview](#overview)
- [Create authorization URL](#create-authorization-url)
- [Validate authorization code](#validate-authorization-code)
- [Proof key for code exchange (PKCE)](#proof-key-for-code-exchange-pkce-flow)
- [Open ID Connect (OIDC)](#open-id-connect-oidc)
- [Account linking](#account-linking)
- [Other considerations](#other-considerations)

## Overview

OAuth is a widely used protocol for authorization. It's what's behind "Sign in with Google" and "Sign in with GitHub." It allows users to grant access to their resources on an external service, like Google, to your application without sharing their credentials. Instead of implementing a password based auth, we can replace it with OAuth to let a third party service handle authentication. You can then get the user's profile and use that to create users and sessions.

In a basic OAuth flow, the user is redirected to a third party service, the service authenticates the user, and the user is redirected back to your application. An access token for the user is made available which allows you to request resources on behalf of the user.

It requires 2 server endpoints in your application:

1. Login endpoint (GET): Redirects the user to the OAuth provider.
2. Callback endpoint (GET): Handles the redirect from the OAuth provider.

There are multiple versions of OAuth, with OAuth 2.0 being the latest one. This page will only cover OAuth 2.0, specifically the authorization code grant type, as standardized in [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749). Implicit grant type is deprecated and should not be used.

## Create authorization URL

Using GitHub as an example, the first step is to create a GET endpoint (login endpoint) that redirects the user to GitHub. The redirect location is the authorization URL with a few parameters.

```
https://github.com/login/oauth/authorize?
response_type=code
&client_id=<CLIENT_ID>
&redirect_uri=<CALLBACK_ENDPOINT>
&state=<STATE>
```

The state is used to ensure the user initiating the process and the one that's redirected back (in the next section) are the same user. As such, a new state must be generated on each request. While it is not strictly required by the spec, it is highly recommended and may be required depending on the provider. It should be generated using a cryptographically-secure random generator and have at least 112 bits of entropy. State can also be used to pass data from the login endpoint to the callback endpoint, though a cookie can just be used instead. 

Your server must keep track of the state associated with each attempt. One simple approach is to store it as a cookie with `HttpOnly`, `SameSite=Lax`, `Secure`, and `Path=/` attributes. You may also assign the state to the current session.

You can define a `scope` parameter to request access to additional resources. If you have multiple scopes, they should be separated by spaces.

```
&scope=email%20identity
```

You can create a "Sign in" button by adding a link to the login endpoint.

```html
<a href="/login/github">Sign in with GitHub</a>
```

## Validate authorization code

The user will be redirected to the callback endpoint (as defined in `redirect_uri`) with a single-use authorization code, which is included as a query parameter. This code is then exchanged for an access token. 

```
https://example.com/login/github/callback?code=<CODE>&state=<STATE>
```

If you added a state to the authorization URL, the redirect request will include a `state` parameter. It is critical to check that it matches the state associated with the attempt. Return an error if the state is missing or if they don't match. A common mistake is forgetting to check whether the `state` parameter exists in the URL.

The code is sent to the OAuth provider's token endpoint via an `application/x-www-form-urlencoded` POST request.

```
POST https://github.com/login/oauth/access_token
Accept: application/json
Authorization: Basic <CREDENTIALS>
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&client_id=<CLIENT_ID>
&redirect_uri=<CALLBACK_ENDPOINT>
&code=<CODE>
```

If your OAuth provider uses a client secret, it should be base64 encoded with the client ID and secret included in the Authorization header (HTTP basic authorization scheme).

```go
var clientId, clientSecret string
credentials := base64.StdEncoding.EncodeToString([]byte(clientId + ":" + clientSecret))
```

Some providers also allow the client secret to be included in the body.

```
POST https://github.com/login/oauth/access_token
Accept: application/json
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&client_id=<CLIENT_ID>
&client_secret=<CLIENT_SECRET>
&redirect_uri=<CALLBACK_ENDPOINT>
&code=<CODE>
```

The request will return an access token, which can then be used to get the user's identity. It may also include other fields such as `refresh_token` and `expires_in`.

```
{ "access_token": "<ACCESS_TOKEN>" }
```

For example, using the access token, you can get their GitHub profile and store their GitHub user ID, which will allow you to get their registered account when they sign in again. Be aware that the email address provided by the OAuth provider may not be verified. You may need to manually verify user emails or block users without a verified email.

The access token itself should never be used as a replacement for sessions.

## Proof key for code exchange (PKCE) flow

PKCE was introduced in [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636) to provide additional protection for OAuth 2.0. We recommend using it in addition to state and a client secret if your OAuth provider supports it. Be aware that some OAuth providers do not require a client secret when PKCE is enabled, in which case PKCE should not be used.

PKCE can replace state entirely, as both protects against CSRF attacks, but it may be required by your OAuth provider.

A new code verifier must be generated on each request. It should be generated using a cryptographically-secure random generator and have at least 112 bits of entropy (256 bits recommended by the RFC). Similar to state, your application must keep track of the code verifier associated with each attempt (using cookies or sessions). A base64url (no padding) encoded SHA256 hash of it called a code challenge is included the authorization URL.

```go
var codeVerifier string
codeChallengeBuf := sha256.Sum256([]byte(codeVerifier))
codeChallenge := base64.URLEncoding.WithPadding(base64.NoPadding).EncodeToString(codeChallengeBuf)
```

```
https://accounts.google.com/o/oauth2/v2/auth?
response_type=code
&client_id=<...>
&redirect_uri=<...>
&state=<...>
&code_challenge_method=S256
&code_challenge=<CODE_CHALLENGE>
```

In the callback endpoint, the code verifier of the current attempt should be sent alongside the authorization code.

```
POST https://oauth2.googleapis.com/token
Accept: application/json
Authorization: Basic <CREDENTIALS>
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&client_id=<...>
&redirect_uri=<...>
&code=<...>
&code_verifier=<CODE_VERIFIER>
```

## Open ID Connect (OIDC)

[Open ID Connect](https://openid.net/specs/openid-connect-core-1_0.html) is a widely used protocol built on top of OAuth 2.0. An important addition to OAuth is that identity provider returns an ID token alongside the access token. An ID token is a [JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519) that includes user data. It will always include a user ID in the `sub` field. 

```
{ 
	"access_token": "<ACCESS_TOKEN>",
	"id_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwiaXNzIjoiZXhhbXBsZS5jb20ifQ.uMMQPfp7LwcLiBbfZdoHdIPjKgS2HUfOr5vlY71el8A"
}
```

While you can validate the token with a public key, this is not necessary for server-side applications and you can safely assume that the token is valid.

## Account linking

Account linking allows users to sign in with any of their social accounts and be authenticated as the same user on your application. It it usually done by checking the email address registered with the provider. If you're using email to link accounts, make sure to validate the user's email. Most providers provide a `is_verified` field or similar in user profiles. Do not assume that the email has been verified unless the provider explicitly mentions it in their documentation. Users without a verified email should be prevented from completing the authentication process and prompted to verify their email first.

## Other considerations

- [Open redirect](/open-redirect).