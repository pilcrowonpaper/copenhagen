---
title: "Passkeys"
---

# Passkeys

- [Overview](#overview)
- [Challenge](#challenge)
- [Registration](#registration)
- [Authentication](#authentication)

## Overview

Passkeys are built on top the [Web Authentication (WebAuthn) standard](https://www.w3.org/TR/webauthn-2/) and allows applications to authenticate users with in-device authentication methods, including biometrics and device pin-code. It can be more secure than traditional passwords as it doesn't require the user to remember their passwords. It can replace passwords entirely or be used in addition to passwords as a [second factor](/mfa.md).

Passkeys are based on public key cryptography, where each user has a public-private key pair. The private key is stored in the user's device, while the public key is stored in your application. The device creates a signature with the private key and your application can use the public key to verify it.

## Challenge

Each attestation and assertion has a challenge associated to it. A challenge is a randomly generated single-use [token](/server-side-tokens.md) stored in the server for preventing replay attacks. The recommended minimum entropy is 16 bytes.

## Registration

In the client, get a new challenge from the server and create a new credential with the [Web Authentication API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API). This will prompt the user to authenticate with their device. Browsers such as Safari will only allow you to call this method if it was initiated by a user interaction (button click).

```ts
const publicKeyCredential: PublicKeyCredential = await navigator.credentials.create({
	publicKey: {
		rp: { name: "My app" },
		user: {
			id: crypto.getRandomValues(new Uint8Array(32)),
			name: userId,
			displayName: username
		},
		pubKeyCredParams: [
			{
				type: "public-key",
				// ECDSA with SHA-256
				alg: -7
			}
		],
		challenge
	}
});
const response: AuthenticatorAttestationResponse = publicKeyCredential.response;

const publicKey: ArrayBuffer = response.getPublicKey();
const clientDataJSON: ArrayBuffer = response.clientDataJSON;
const authenticatorData: ArrayBuffer = response.getAuthenticatorData();
const credentialId: string = publicKeyCredential.id;
```

- `rp.name`: Your application's name
- `user.id`: Random ID
- `user.name`: Unique user identifier (user ID, username, email)
- `user.displayName`: Does not need to unique

The algorithm ID is from the [IANA COSE Algorithms registry](https://www.iana.org/assignments/cose/cose.xhtml). ECDSA with SHA-256 (ES256) is recommended as it is widely supported. You can also pass `-257` for RSASSA-PKCS1-v1_5 (RS256) to support a more wider range of devices but device that only supports it is rare.

The public key, client data, authenticator data, credential ID, and the challenge is sent to the server for verification. A simple way to send binary data is by encoding it with base64.

The first step is to validate the challenge. Make sure to delete the challenge from storage as it is single-use. Next, check the client data and authenticator data. The origin is the domain your application is hosted on, including the protocol and port, and the relying party ID is the domain without the protocol or port.

```go
import (
	"bytes"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"errors"
)

var challenge []byte

// Verify the challenge and delete it from storage.

var publicKey, clientDataJSON, authenticatorData []byte
var credentialId string

var clientData ClientData
json.Unmarshal(clientDataJSON, &clientData)

if clientData.Type != "webauthn.create" {
	return errors.New("invalid type")
}
if clientData.Challenge != base64.URLEncoding.WithPadding(base64.NoPadding).EncodeToString(challenge) {
	return errors.New("invalid challenge")
}
if clientData.Origin != "https://example.com" {
	return errors.New("invalid origin")
}

if len(authenticatorData) < 37 {
	return errors.New("invalid authenticator data")
}
rpIdHash := authenticatorData[0:32]
expectedRpIdHash := sha256.Sum256([]byte("example.com"))
if bytes.Equal(rpIdHash, expectedRpIdHash[:]) {
	return errors.New("invalid relying party ID")
}
// Check for the "user present" flag.
if (authenticatorData[32] & 1) != 1 {
	return errors.New("invalid flag")
}

type ClientData struct {
	Type	  string `json:"type"`
	Challenge string `json:"challenge"`
	Origin	string `json:"origin"`
}
```

Optionally, validate the attestation statement to verify that the attestation came from a legitimate device. However, unless your application has strict security or needs to verify the authenticity of the user's device, this is likely unnecessary.

The authenticator data also includes a signature counter that is incremented every time a new signature is generated, which can be used to detect cloned authenticators. However, for passkeys specifically, this is not necessary as credentials are designed to be exported and shared.

Finally, check if the public key is valid, and create a new user with their public key and the credential ID. The public key is in the SubjectPublicKeyInfo format. If you support multiple algorithms, you can parse the public key to get the algorithm identifier.

## Authentication

Generate a challenge on the server and use it authenticate the user client side.

```ts
const publicKeyCredential: PublicKeyCredential = await navigator.credentials.get({
	publicKey: {
		challenge
	}
});

const response: AuthenticatorAssertionResponse = publicKeyCredential.response;
const clientDataJSON: ArrayBuffer = response.clientDataJSON);
const authenticatorData: ArrayBuffer = response.authenticatorData);
const signature: ArrayBuffer = response.signature);
const credentialId: string = publicKeyCredential.id;
```

The client data, authenticator data, signature, challenge, and credential ID is sent to the server. The challenge, the authenticator, and the client data is first verified. This part is nearly identical to the steps for verifying attestation.

```go
import (
	"bytes"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"errors"
)

var challenge []byte

// Verify the challenge and delete it from storage.

var clientDataJSON, authenticatorData []byte

var clientData ClientData
json.Unmarshal(clientDataJSON, &clientData)

if clientData.Type != "webauthn.get" {
	return errors.New("invalid type")
}
if clientData.Challenge != base64.URLEncoding.WithPadding(base64.NoPadding).EncodeToString(challenge) {
	return errors.New("invalid challenge")
}
if clientData.Origin != "https://example.com" {
	return errors.New("invalid origin")
}

if len(authenticatorData) < 37 {
	return errors.New("invalid authenticator data")
}
rpIdHash := authenticatorData[0:32]
expectedRpIdHash := sha256.Sum256([]byte("example.com"))
if !bytes.Equal(rpIdHash, expectedRpIdHash[:]) {
	return errors.New("invalid relying party ID")
}
// Check for the "user present" flag.
if (authenticatorData[32] & 1) != 1 {
	return errors.New("invalid flag")
}
```

Next step is to verify the signature. Use credential ID to get the user's public key and verify the signature, which is ASN.1 DER encoded. The algorithm depends on the parameters passed when the credential was created.

```go
import (
	"crypto/ecdsa"
	"crypto/sha256"
	"errors"
)

var publicKey *ecdsa.PublicKey
var signature []byte

hashedClientDataJSON := sha256.Sum256(clientDataJSON)
// Concatenate the authenticator data with the hashed client data JSON.
data := append(authenticatorData, hashedClientDataJSON[:]...)
hash := sha256.Sum256(data)

validSignature := ecdsa.VerifyASN1(publicKey, hash[:], signature)
if !validSignature {
	return errors.New("invalid signature")
}
```

