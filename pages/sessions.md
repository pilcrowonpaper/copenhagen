---
title: "Sessions"
---

# Sessions

## Table of contents

- [Overview](#overview)
- [Session lifetime](#session-lifetime)
	- [Sudo mode](#sudo-mode)
- [Safeguards](#safeguards)
- [Session invalidation](#session-invalidation)
- [Session storage](#session-storage)
  - [Cookies](#cookies)
  - [Web Storage API](#web-storage-api)
- [Session fixation attacks](#session-fixation-attacks)

## Overview

Throughout a user's visit to your website, they will make multiple requests to your server. If you need to persist state, such as user preference, across those requests, HTTP doesn't provide a mechanism for it. It's a stateless protocol.

Sessions are a way to persist state in the server. It is especially useful for managing authentication state, such as the client's identity. We can assign each session with a unique ID and store it on the server to use it as a token. Then the client can associate the request with a session by sending the session ID with it. To implement authentication, we can simply store user data alongside the session.

It's important that the session ID is sufficiently long and random, or else someone could impersonate other users by just guessing their session IDs. See the [Server-side tokens](/server-side-tokens) guide for generating secure session IDs. Session IDs can be hashed before storage to provide an extra level of security.

Depending on your application, you may only have to manage sessions for authenticated users, or for both authenticated and unauthenticated users. You can even manage 2 different kinds of sessions - one for auth and another for non-auth related state.

## Session lifetime

If you only manage sessions for authenticated users, a new session is created whenever a user signs in. If you plan to manage session for non-authenticated users as well, sessions should be automatically created when an incoming request doesn't include a valid session. Make sure you don't make your application vulnerable to [session fixation attacks](#session-fixation-attacks).

For security critical applications, it is crucial for sessions to expire automatically. This minimizes the time an attacker has to hijack sessions. The expiration should match how long the user is expected to use your application in a single sitting.

However, for less critical websites, such as a social media app, it would be be annoying for users if they had to sign in every single day. A good practice here is to set the expiration to a reasonable time, like 30 days, but extend the expiration whenever the session is used. For example, sessions may expire in 30 days by default, but the expiration gets pushed back 30 days when it's used within 15 days before expiration. This effectively invalidates sessions for inactive users, while keeping active users signed in.

You can also combine both approaches. For example, you can set the expiration to an hour and extend it every 30 minutes, but set an absolute expiration of 12 hours so sessions won't last for longer than that.

```go
const sessionExpiresIn = 30 * 24 * time.Hour

func validateSession(sessionId string) (*Session, error) {
	session, ok := getSessionFromStorage(sessionId)
	if !ok {
		return nil, errors.New("invalid session id")
	}
	if time.Now().After(session.ExpiresAt) {
		return nil, errors.New("expired session")
	}
	if time.Now().After(session.expiresAt.Sub(sessionExpiresIn / 2)) {
		session.ExpiresAt = time.Now().Add(sessionExpiresIn)
		updateSessionExpiration(session.Id, session.ExpiresAt)
	}
	return session, nil
}
```

### Sudo mode

An alternative to short-lived sessions is to implement long-lived sessions coupled with sudo mode. Sudo mode allows authenticated users to access security-critical components for a limited time by re-authenticating with one of their credentials (passwords, passkeys, TOTP, etc). A simple way to implement this is by keeping track of when the user last used their credentials in each sessions. This approach provides the security benefits of short-lived sessions without annoying frequent users.

## Safeguards

Consider tracking the user agent (device) and IP address linked to the session to detect suspicious requests. IP addresses can be dynamic for mobile users so you may want to keep track of the general area (country) instead of the specific address. You can also limit the number of active sessions that a single user can have at any time.

## Session invalidation

Sessions can be invalidated by deleting it from both server and client storage.

When the user signs out, invalidate the current session, or for security-critical applications, invalidate all sessions belonging to that user. 

All sessions of the user should also be invalidated when they gain new permissions (email verification, new role, etc) or change passwords.

## Client storage

The client should store session ID in the user's device to be used for subsequent requests. The browser mainly provides 2 way to store data - cookies and the Web Storage API. Cookies should be preferred for websites as they're automatically included in requests by the browser.

### Cookies

Session cookies should have the following attributes:

- `HttpOnly`: Cookies are only accessible server-side
- `SameSite=Lax`: Use `Strict` for critical websites
- `Secure`: Cookies can only be sent over HTTPS
- `Max-Age` or `Expires`: Must be defined to persist cookies
- `Path=/`: Cookies can be accessed from all routes

[CSRF protection](/csrf) must be implemented when using cookies, and using the `SameSite` flag is not sufficient. Using cookies does not automatically protect your users from cross-site scripting attacks (XSS) as well. While the session ID can't be read directly, authenticated requests can still be made as browsers automatically include cookies in requests. 

The maximum expiration for a cookie is anywhere between 1 and 2 years. If you plan for the session to be long-lived, continuously set the cookie on a set interval (e.g. when you extend the session expiration). 

`Lax` should be preferred over `Strict` for the `SameSite` attribute as using `Strict` will cause the browser to not send the session cookie when the user visits your application via an external link.

### Web Storage API

Another option is to store session IDs inside `localStorage` or `sessionStorage`. If your website has an XSS vulnerability, this will allow attackers to directly read and steal the user's session ID. It is especially vulnerable to supply chain attacks since tokens can be stolen by just reading the entire local storage, without using any application-specific exploits.

Session tokens can be sent with the request using the `Authorization` header for example. Do not send them inside URLs as query parameters or inside form data, nor do not accept tokens sent in such ways. 

## Session fixation attacks

Applications that maintain sessions for both authenticated and unauthenticated users and reuses the current session when a user signs in, are vulnerable to session fixations attacks.

Say an application allows the session ID to be sent inside the URL as a query parameter. If an attacker shares a link to the sign in page with a session ID already included and the user signs in, the attacker now has a valid session ID to impersonate that user. A similar attack can be done if the application accepts session IDs in forms or cookies, though the latter requires an XSS vulnerability to exploit.

This can be avoided by only accepting session IDs via cookies and request headers, and always creating a new session when the user signs in.