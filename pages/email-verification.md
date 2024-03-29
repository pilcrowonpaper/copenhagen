---
title: "Email verification"
---

# Email verification

If your application requires user email addresses to be unique, email verification is a must. It discourages users from entering a random email address and, if password reset is implemented, allows users to take back accounts created with their email address. You may even want to block users from accessing your application's content until they verify their email address.

## Table of contents

- [Input validation](#input-validation)
	- [Sub-addressing](#sub-addressing)
- [Email verification codes](#email-verification-codes)
- [Email verification links](#email-verification-links)
- [Changing emails](#changing-emails)
- [Rate limiting](#rate-limiting)

## Input validation

Emails are complex and cannot be fully validated using Regex. Attempting to use Regex may also introduce [ReDoS vulnerabilities](https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS). Do not over-complicate it:

- Includes at least 1 `@` character.
- Has at least 1 character before the`@`.
- The domain part includes at least 1 `.` and has at least 1 character before it.
- It does not start or end with a whitespace.
- Maximum of 255 characters.

### Sub-addressing

Some email providers, including Google, allow users to specify a tag that will be ignored by their servers. For example, a user with `user@example.com` can use `user+foo@example.com` and `user+bar@example.com`. You can block emails with `+` to prevent users from making multiple accounts with the same email address, but users would still be able to use temporary email addresses or just create a new email address. Never silently remove the tag portion from the user input as an email address with `+` can just be a regular, valid email address.

## Email verification codes

One way to verify email is to send a secret code stored in the server to the user's mailbox.

This approach should be preferred over using links. People are increasingly less likely to click on links, and some filters may block emails with them. Using links also limits what device the user can use to create an account (eg. the user doesn't have access to their mailbox on their phone).

The verification code should be at least 8 digits if the code is numeric, and at least 6 digits if it's alphanumeric. You should avoid using both lowercase and uppercase letters. You may also want to remove numbers and letters that can be misread (0, O, 1, I, etc). It must be generated using a cryptographically secure random generator.

A single verification code should be tied to a single user and email. This is especially important if you allow users to change their email address after they're sent an email. Each code should be valid for at least 15 minutes (anywhere between 1-24 hours is recommended). The code must be single-use and immediately invalidated after validation. A new verification code should be generated every time the user asks for another email/code.

Similar to a regular login form, throttling or rate-limiting based on the user ID must be implemented. A good limit is around 10 attempts per hour. Assuming proper limiting is implemented, the code can be valid for up to 24 hours. You should generate and resend a new code if the user-inputted code has expired.

All sessions of a user should be invalidated when their email is verified.

## Email verification links

An alternative way to verify emails is to use a verification link that contains a long, random, single-use [token](/server-side-tokens).

```
https://example.com/verify-email/<TOKEN>
```

A single token should be tied to a single user and email. This is especially important if you allow users to change their email address after they're sent an email. Tokens should be single-use and be immediately deleted from storage after verification. The token should be valid for at least 15 minutes (anywhere between 1-24 hours is recommended). When a user asks for another verification email, you can resend the previous token instead of generating a new token if that token is still within expiration.

Make sure to set the [Referrer Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy) tag to `no-referrer` for any path that includes tokens to protect the tokens from referer leakage.

All sessions should be invalidated when the email is verified (and create a new one for the current user so they stay signed in).

## Changing emails

The user should be asked for their password, or if [multi-factor authentication](/mfa) is enabled, authenticated with one of their second factors. The new email should be stored separately from the current email until it's verified. For example, the new email could be stored with the verification token/code.

A notification should be sent to the previous email address when the user changes their email.

## Rate limiting

Any endpoint that can send emails should have strict rate limiting implemented.
