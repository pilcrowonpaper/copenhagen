---
title: "Elliptic curve digital signature algorithm (ECDSA)"
---

# Elliptic curve digital signature algorithm (ECDSA)

ECDSA is a digital signature algorithm using elliptic-curve cryptography. A private key is used to sign a message and a public key is used verify the signature.

The message is hashed with algorithms like SHA-256 before signing.

```go
import (
	"crypto/ecdsa"
	"crypto/rand"
	"crypto/sha256"
)

msg := "Hello world!"
hash := sha256.Sum256([]byte(msg))
signature, err := ecdsa.SignASN1(rand.Reader, privateKey, hash[:])
```

## Signatures

ECDSA signatures are represented using a pair of positive integers, (r, s).

### IEEE P1363

In the IEEE P1363 format, the signature is the concatenation of r and s. The values are encoded as big-endian bytes with a size equivalent to the curve size. For example, P-256 is 256 bits or 32 bytes in size.

```ts
r || s;
```

### PKIX

In [RFC 5480](https://datatracker.ietf.org/doc/html/rfc5480) by the PKIX working group, the signature is ASN.1 DER encoded sequence of r and s.

```
SEQUENCE {
    r     INTEGER,
    s     INTEGER
}
```

## Public keys

ECDSA public keys are represented as a pair of positive integers, (x, y).

### SEC1

In SEC 1, which defines ECDSA, public keys can either be uncompressed or compressed. Uncompressed keys are the concatenation of x and y, with a leading `0x04` byte. The values are encoded as big-endian bytes with a size equivalent to the curve size. For example, P-256 is 256 bits or 32 bytes in size.

```
0x04 || x || y
```

Compressed keys are the x value with a leading `0x02` byte if x is even or `0x03` byte if x is odd. The y value can be derived from x and the curve.

```
0x02 || x
0x03 || x
```

### PKIX

In [RFC 5480](https://datatracker.ietf.org/doc/html/rfc5480) by the PKIX working group, the public key is represented as a `SubjectPublicKeyInfo` ASN.1 sequence. The `subjectPublicKey` is either the compressed or uncompressed SEC1 public key.

```
SubjectPublicKeyInfo := SEQUENCE {
    algorithm           AlgorithmIdentifier,
    subjectPublicKey    BIT STRING
}
```

The `AlgorithmIdentifier` for ECDSA is an ASN.1 sequence with the ECDSA object identifier (`1.2.840.10045.2.1`) and the curve (e.g. `1.2.840.10045.3.1.7` for P-256 curve)

```
AlgorithmIdentifier := SEQUENCE {
    algorithm   OBJECT IDENTIFIER
    namedCurve  OBJECT IDENTIFIER
}
```
