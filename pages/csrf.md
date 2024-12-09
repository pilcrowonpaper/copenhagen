---
title: "Cross-site request forgery (CSRF)"
---

# Cross-site request forgery (CSRF)

## Table of contents

- [Overview](#overview)
	- [Cross-site vs cross-origin](#cross-site-vs-cross-origin)
- [Prevention](#prevention)
	- [Anti-CSRF tokens](#anti-csrf-tokens)
	- [Signed double-submit cookies](#signed-double-submit-cookies)
	- [Origin header](#origin-header)
- [SameSite cookie attribute](#samesite-cookie-attribute)

## Overview

CSRF attacks allow an attacker to make authenticated requests on behalf of users when credentials are stored in cookies.

When a client makes a cross-origin request, the browser sends a preflight request to check whether the request is allowed (CORS). However, for certain "simple" requests, including form submissions, this step is omitted. And since cookies are automatically included even for cross-origin requests, it allows a malicious actor to make requests as the authenticated user without ever directly stealing the token from any domain. The [same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy) prohibits cross-origin clients from reading responses by default, but the request still goes through.

For example, if you are signed into `bank.com`, your session cookie will be sent alongside this form submission even if the form is hosted on a different domain.

```html
<form action="https://bank.com/send-money" method="post">
	<input name="recipient" value="attacker" />
	<input name="value" value="$100" />
	<button>Send money</button>
</form>
```

This can just be a `fetch()` request so no user input is required.

```ts
const body = new URLSearchParams();
body.set("recipient", "attacker");
body.set("value", "$100");

await fetch("https://bank.com/send-money", {
	method: "POST",
	body
});
```

### Cross-site vs cross-origin

While requests between 2 totally different domains are considered as both cross-site and cross-origin, those between 2 subdomains are not considered cross-site but are considered cross-origin requests. While the name cross-site request forgery implies cross-site requests, you should be strict by default and protect your application from cross-origin attacks as well.

## Prevention

CSRF can be prevented by only accepting POST and POST-like requests made by browsers from a trusted origin.

Protection must be implemented for all routes that deal with forms. If your application does not currently use forms, it may still be a good idea to at least [check the `Origin` header](#origin-header) to prevent future issues. It's also a generally good idea to only modify resources using POST and POST-like methods (PUT, DELETE, etc).

For the common token-based approach, the token should not be single-use (e.g. a new token for every form submission) as it will break with a single back button. It is also crucial that your pages have a strict cross-origin resource sharing (CORS) policy. If `Access-Control-Allow-Credentials` is not strict, a malicious site can send a GET request to get an HTML form with a valid CSRF token.

### Anti-CSRF tokens

This is a very simple method where each session has a unique CSRF [token](/server-side-tokens) associated with it.

```html
<form method="post">
	<input name="message" />
	<input type="hidden" name="__csrf" value="<CSRF_TOKEN>" />
	<button>Submit</button>
</form>
```

### Signed double-submit cookies

If storing the token server-side is not an option, using signed double-submit cookies is another approach. This is different from the basic double submit cookie in that the token included in the form is signed with a secret.

A new [token](/server-side-tokens) is generated and hashed with HMAC SHA-256 using a secret key. Each HMAC must be linked to the user's session. You can alternatively encrypt the token with algorithms like AES.

```go
func generateCSRFToken(sessionId string) (string, []byte) {
	buffer := [10]byte{}
	crypto.rand.Read(buffer)
	csrfToken := base64.StdEncoding.encodeToString(buffer)
	mac := hmac.New(sha256.New, secret)
	mac.Write([]byte(csrfToken + "." + sessionId))
	csrfTokenHMAC := mac.Sum(nil)
	return csrfToken, csrfTokenHMAC
}
```

The token is stored as a cookie and the HMAC is embedded in the form. The cookie should have the `Secure`, `HttpOnly`, and `SameSite` attribute. To validate a request, the cookie can be used to verify the signature sent in the form data.

#### Traditional double-submit cookies

Regular double-submit cookies that aren't signed will still leave you vulnerable if an attacker has access to a subdomain of your application's domain. This would allow them to set their own double-submit cookies.

### Origin header

A very simple way to prevent CSRF attacks is to check the `Origin` header of the request for non-GET requests. This is a relatively new header that includes the request [origin](https://developer.mozilla.org/en-US/docs/Glossary/Origin). If you rely on this header, it is crucial that your application does not use GET requests for modifying resources.

While the `Origin` header can be spoofed by using a custom client, the important part is that it can't be done using client-side JavaScript. Users are only vulnerable to CSRF when using a browser.

```go
func handleRequest(w http.ResponseWriter, request *http.Request) {
  	if request.Method != "GET" {
		originHeader := request.Header.Get()
		// You can also compare it against the Host or X-Forwarded-Host header.
		if originHeader != "https://example.com" {
			// Invalid request origin
			w.WriteHeader(403)
			return
		}
  	}
  	// ...
}
```

The `Origin` header has been supported by all modern browsers since around 2020, though Chrome and Safari have supported it before that. If the `Origin` header is not included, do not allow the request.

The `Referer` header is a similar header introduced before the `Origin` header. This can be used as a fallback if the `Origin` header isn't defined.

## SameSite cookie attribute

Session cookies should have a `SameSite` flag. This flag determines when the browser includes the cookie in requests. `SameSite=Lax` cookies will only be sent on cross-site requests if the request uses a [safe HTTP method](https://developer.mozilla.org/en-US/docs/Glossary/Safe/HTTP) (such as GET), while `SameSite=Strict` cookies will not be sent on any cross site requests. We recommend using `Lax` as the default as `Strict` cookies will not be sent when a user accesses your website via an external link.

If you set the value to `Lax`, it is crucial that your application does not use GET requests for modifying resources. Browser support for the `SameSite` flag shows it is currently available to 96% of web users. It’s important to note that the flag only protects against *cross-site* request forgery (not *cross-origin* request forgery), and generally shouldn’t be your only layer of defense.


