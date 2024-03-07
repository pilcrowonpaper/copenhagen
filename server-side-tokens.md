# Server-side tokens

- [Overview](#overview)
- [Generating tokens](#generating-tokens)
- [Storing tokens](#storing-tokens)

## Overview

A "server-side token" is any long, random string that is stored on the server. It may be persisted in a database or in-memory data store (e.g. Redis) and is used for authentication and verification. A token can be validated by checking if it exists in storage. Examples include session IDs, email verification tokens, and access tokens.

```
CREATE TABLE token (
	token STRING NOT NULL UNIQUE,
	expires_at INTEGER NOT NULL,
	user_id INTEGER NOT NULL,

	FOREIGN KEY (user_id) REFERENCES user(id)
)
```

For single-use tokens, any retrieval should also guarantee deletion. In SQL for example, an atomic operation such as a transaction should be used when fetching a token.

## Generating tokens

Tokens should have at least 112 bits of entropy (120-256 is a good range). For example, you could generate 15 random bytes and encode it with base32 to get a 24 character token. If you generate tokens by choosing random characters one-by-one, you should ensure a similar level of entropy. See the [Generating random values](/random-values.md) page for more information.

Tokens must be generated using a cryptographically-secure random generator. Fast, pseudo-random generators like those generally provided by standard math packages should be avoided for this.

Tokens should be case sensitive, but you may want to constrain your token generation to lowercase letters if your storage is case insensitive (e.g. MySQL).

> For a 120 bit token, it would take someone 2 quintillion years before they guess a valid token if they generate 10,000 tokens per second and there are 1,000,000 valid tokens in the system.

```go
import (
	"crypto/rand"
	"encoding/base32"
)

bytes := make([]byte, 15)
rand.Read(bytes)
sessionId := base32.StdEncoding.EncodeToString(bytes)
```

UUID v4 may fit these requirements (122 bits of entropy), but keep in mind that UUID v4 is space inefficient and the spec does not guarantee the use of a cryptographically-secure random generator.

## Storing tokens

Tokens that require an extra level of security, such as password reset tokens, should be hashed with SHA-256. SHA-256 can be used instead of a slower algorithm here as the token is sufficiently long and random. Tokens can be validated by hashing the incoming token before querying.

Real-life examples of accidental leaks include [Paleohacks](https://www.vpnmentor.com/blog/report-paleohacks-breach/) and [Spoutible](https://www.troyhunt.com/how-spoutibles-leaky-api-spurted-out-a-deluge-of-personal-data/).
