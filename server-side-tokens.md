# Server-side tokens

- [Overview](#overview)
- [Generating tokens](#generating-tokens)
- [Storing tokens](#storing-tokens)

## Overview

Server-side tokens refer to any long, random string that is stored in the server, usually in a database or memory storage, used for authentication and verification. A token can be validated by checking if it exists in storage. Examples include session IDs, email verification tokens, and access tokens.

```
CREATE TABLE token (
	token STRING NOT NULL UNIQUE,
	expires_at INTEGER NOT NULL,
	user_id INTEGER NOT NULL,

	FOREIGN KEY (user_id) REFERENCES user(id)
)
```

For single-use tokens, atomic operations such as transactions in SQL should be used to get and delete the token.

## Generating tokens

Tokens should have at least 112 bits of entropy (120-256 is a good range). For example, you could generate 15 random bytes and encode it with base32 to get a 24 character token. If you generate tokens by choosing a random character one-by-one, it should provide a similar entropy. 

Tokens must be generated using a cryptographically-secure random generator. Do not use a fast, pseudo-random generator often provided by the standard math package.

Tokens should be case sensitive, but you may only want to use lowercase letters if your storage is case insensitive (e.g. MySQL).

> For a 120 bit token, it would take someone 2 quintillion years before they guess a valid token if they generate 10,000 tokens per second and there are 1,000,000 valid tokens in the system.

```go
bytes := make([]byte, 15)
rand.Read(bytes)
sessionId := base32.StdEncoding.EncodeToString(bytes)
```

UUID v4 may fit these requirements (122 bits of entropy), but keep in mind that UUID v4 is space inefficient and the spec does not guarantee the use of cryptographically-secure random generator.

## Storing tokens

Tokens that requires an extra level of security, such as password reset tokens, should be hashed with SHA-256. SHA-256 can be used instead of a slower algorithm here as the token is sufficient long and random. Tokens can be validated by hashing the incoming token before querying the storage.