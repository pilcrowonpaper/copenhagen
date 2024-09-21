---
title: "WebAuthn"
---

# WebAuthn

## Table of contents

-   [Overview](#overview)
-   [Vocabulary](#vocabulary)
-   [Registration](#registration)
-   [Authentication](#authentication)

## Overview

The [Web Authentication (WebAuthn) standard](https://www.w3.org/TR/webauthn-2/) allow users to authenticate with their device, either with a PIN code or biometrics. The private key is stored in the user's device, while the public key is stored in your application. Applications can authenticate users by verifying signatures. Since credentials are bounded to the user's device (or devices) and brute-forcing is impossible, a potential attacker needs physical access to a device.

WebAuthn are usually used in 2 ways - with passkeys or security tokens. While they don't have a strict definition, passkeys usually refer to credentials that can replace passwords and stored in the authenticator (resident keys). Security tokens, on the other hand, are meant to be used as a second factor, after authenticating with a password. Credentials for 2FA are usually encrypted and stored in the relying party's server. In both cases, they are a more secure alternatives to existing methods.

Using WebAuthn, applications can also verify the device with the manufacture. This requires attestation and is not covered in this page.

## Vocabulary

-   Relying party: Your application.
-   Authenticator: The device that holds the credential.
-   Challenge: A randomly generated, single-use [token](/server-side-tokens) to prevent replay attacks. The recommended minimum entropy is 16 bytes.
-   User presence: User has access to the device.
-   User verification: User has verified their identity via a pin-code or biometrics.
-   Resident keys, discoverable credentials: Credentials stored in stored in authenticators (user devices and security tokens). Non-resident keys are encrypted and stored in relying party servers (your database).

## Registration

During the registration step, the authenticator creates a new credential and returns its public key.

In the client, get a new challenge from the server and create a new credential with the [Web Authentication API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API). This will prompt the user to authenticate with their device. Browsers such as Safari will only allow you to call this method if it was initiated by a user interaction (button click).

```ts
const credential = await navigator.credentials.create({
	publicKey: {
		attestation: "none",
		rp: { name: "My app" },
		user: {
			id: crypto.getRandomValues(new Uint8Array(32)),
			name: username,
			displayName: name,
		},
		pubKeyCredParams: [
			{
				type: "public-key",
				// ECDSA with SHA-256
				alg: -7,
			},
		],
		challenge,
		authenticatorSelection: {
			// See note below.
			userVerification: "required",
			residentKey: "required",
			requireResidentKey: true,
		},
		// list of existing credentials
		excludeCredentials: [
			{
				id: new Uint8Array(/*...*/),
				type: "public-key",
			},
		],
	},
});
if (!(credential instanceof PublicKeyCredential)) {
	throw new Error("Failed to create credential");
}
const response = credential.response;
if (!(response instanceof AuthenticatorAttestationResponse)) {
	throw new Error("Unexpected");
}

const clientDataJSON: ArrayBuffer = response.clientDataJSON;
const attestationObject: ArrayBuffer = response.attestationObject;
```

-   `rp.name`: Your application's name.
-   `user.id`: Random user ID for the authenticator. This can be different from the actual user ID your application uses.
-   `user.name`: A human-friendly user identifier (username, email).
-   `user.displayName`: A human-friendly display name (does not need to be unique).
-   `excludeCredentials`: A list of the user's credentials to avoid duplicate credentials.

The algorithm ID is from the [IANA COSE Algorithms registry](https://www.iana.org/assignments/cose/cose.xhtml). ECDSA with SHA-256 (ES256) is recommended as it is widely supported. You can also pass `-257` for RSASSA-PKCS1-v1.5 (RS256) to support a wider range of devices but devices that only support it are rare.

For most cases, `attestation` should be set to `"none"`. We don't need to verify of the authenticator and not all authenticators support it.

For passkeys, ensure the public key is a resident key and requires user verification.

```ts
const credential = await navigator.credentials.create({
	publicKey: {
		// ...
		authenticatorSelection: {
			userVerification: "required",
			residentKey: "required",
			requireResidentKey: true,
		},
	},
});
```

For security tokens, we can skip user verification and the credential doesn't need to be a resident key. We can limit the authenticator to security tokens by setting `authenticatorAttachment` to `cross-platform` as well.

```ts
const credential = await navigator.credentials.create({
	publicKey: {
		// ...
		authenticatorSelection: {
			userVerification: "discouraged",
			residentKey: "discouraged",
			requireResidentKey: false,
			authenticatorAttachment: "cross-platform",
		},
	},
});
```

The client data JSON and authenticator data are sent to the server for verification. A simple way to send binary data is by encoding it with base64. Another option is use schemes like CBOR that encode JSON-like data into binary.

The first step is to parse the attestation object, which is encoded with CBOR. This includes the attestation statement and authenticator data. You can use the attestation statement to verify the user's device if you required it. If you've set it to `"none"` in the client, verify that the statement format is `none`.

```go
var attestationObject AttestationObject

// Parse attestation object

if attestationObject.Fmt != "none" {
	return errors.New("invalid attestation statement format")
}

type AttestationObject  struct {
	Fmt                  string // "fmt"
	AttestationStatement AttestationStatement // "attStmt"
	AuthenticatorData    []byte // "authData"
}

type AttestationStatement struct {
	// see spec
}
```

Next is to parse the authenticator data.

-   Bytes 0-31: Relying party ID hash.
-   Byte 32: Flags:
    -   Bit 0 (least significant - rightmost): Use present.
    -   Bit 2: User verified.
    -   Bit 6: Includes credential data.
-   Bytes 33-36: Signature counter.
-   Variable bytes: Credential data (binary).

The relying party ID is the domain without the protocol or port and the authenticator data includes the SHA-256 hash of it. For localhost, the relying party ID is `localhost`. Check for the user presence flag and for the user verification flag if you required user verification. The signature counter is incremented each time the credential is used and can be used to detect forged devices. If your application is intended to be used with hardware security tokens, where credentials are bound to the token, you'd want to store the counter with the credential and ensure the counter value is larger than the previous attempt. However, since passkeys are meant to be shared across devices, this can be safely ignored.

Then, extract the credential ID and public key from the credential data.

-   Bytes 0-15: ID of the authenticator.
-   Bytes 16 and 17: Credential ID length.
-   Variable bytes: Credential ID.
-   Variable bytes: COSE public key.

The public key is a CBOR-encoded COSE key.

```go
import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"encoding/json"
	"errors"
)
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
	return errors.New("user not present")
}
// Check for the "user verified" flag if you need user verification.
if ((authenticatorData[32] >> 2) & 1) != 1 {
	return errors.New("user not verified")
}
if ((authenticatorData[32] >> 6) & 1) != 1 {
	return errors.New("missing credentials")
}

if (len(authenticatorData) < 55) {
	return errors.New("invalid authenticator data")
}
credentialIdSize:= binary.BigEndian.Uint16(authenticatorData[53 : 55])
if (len(authenticatorData) < 55 + credentialIdSize) {
	return errors.New("invalid authenticator data")
}
credentialId := authenticatorData[55 : 55+credentialIdSize]
coseKey := authenticatorData[55+credentialIdSize:]

// Parse COSE public key
```

The structure of the public key will depend on the algorithm used. Below is the public key for ECDSA, which uses (x, y) for public keys. Validate the algorithm and curve.

```
{
	1: 2 // EC2 key type
	3: -7 // Algorithm ID for ECDSA P-256 with SHA-256
	-1: 1 // Curve ID for P-256
	-2: 0x00...00 // x coordinate in bit string
	-3: 0x00...00 // y coordinate in bit string
}
```

Next, validate the client data, which is JSON-encoded. The origin is the domain of your application with the protocol and port. The challenge in the client data is base64url encoded with no padding.

```go
import (
	"bytes"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"errors"
)

var expectedChallenge []byte

// Verify the challenge and delete it from storage.

var credentialId string

var clientData ClientData

// Parse JSON

if clientData.Type != "webauthn.create" {
	return errors.New("invalid type")
}
if !verifyChallenge(clientData.Challenge) {
	return errors.New("invalid challenge")
}
if clientData.Origin != "https://example.com" {
	return errors.New("invalid origin")
}

type ClientData struct {
	Type	  string // "type"
	Challenge string // "challenge"
	Origin	  string // "origin"
}
```

Finally, create a new user with their public key and the credential ID.

## Authentication

During the authentication step, the authenticator creates a new signature using the private key.

Generate a challenge on the server and authenticate the user.

```ts
const credential = await navigator.credentials.get({
	publicKey: {
		challenge,
		userVerification: "required",
	},
});

if (!(credential instanceof PublicKeyCredential)) {
	throw new Error("Failed to create credential");
}
const response = credential.response;
if (!(response instanceof AuthenticatorAssertionResponse)) {
	throw new Error("Unexpected");
}

const clientDataJSON: ArrayBuffer = response.clientDataJSON);
const authenticatorData: ArrayBuffer = response.authenticatorData
const signature: ArrayBuffer = response.signature);
const credentialId: ArrayBuffer = publicKeyCredential.rawId;
```

For implementing 2FA with security tokens, pass a list of the user's credentials to `allowCredentials` to support non-resident keys.

```ts
const credential = await navigator.credentials.get({
	publicKey: {
		challenge,
		userVerification: "required",
		// list of user credentials
		allowCredentials: [
			{
				id: new Uint8Array(/*...*/),
				type: "public-key",
			},
		],
	},
});
```

The client data, authenticator data, signature, and credential ID are sent to the server. The challenge, the authenticator, and the client data are first verified. This part is nearly identical to the steps for verifying attestation expect that the client data type should be `webauthn.get`.

Another difference is that the credential portion of the authenticator is not included.

```go
if clientData.Type != "webauthn.get" {
	return errors.New("invalid type")
}
```

Finally, verify the signature. The signature is of the authenticator data and the SHA-256 hash of the client data JSON. For ECDSA, the signature is ASN.1 DER encoded.

```go
import (
	"crypto/ecdsa"
	"crypto/sha256"
)

clientDataJSONHash := sha256.Sum256(clientDataJSON)
// Concatenate the authenticator data with the hashed client data JSON.
data := append(authenticatorData, clientDataJSONHash[:]...)
hash := sha256.Sum256(data)
validSignature := ecdsa.VerifyASN1(publicKey, hash[:], signature)
```
