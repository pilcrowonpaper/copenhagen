---
title: "Generating random values"
---

# Generating random values

## Table of contents

- [Overview](#overview)
- [Random strings](#random-strings)
  - [Custom character set](#custom-character-set)
- [Random integers](#random-integers)
- [Random floating-point numbers between 0 and 1](#random-floating-point-numbers-between-0-and-1)
- [Biases](#biases)

## Overview

Pseudo-random generators often provided by the standard math package are fast but predictable. When dealing with cryptography, having access to a strong random generator is essential.

This page describes how to generate random strings and numbers from randomly generated bits.

## Random strings

The easiest and safest way to generate random strings is to generate random bytes and encode it with base16 (hex), base32, or base64 encoding schemes.

```go
import (
	"crypto/rand"
	"encoding/base32"
)

func generateRandomString() string {
	bytes := make([]byte, 12)
	rand.Read(bytes)
	return base32.StdEncoding.EncodeToString(bytes)
}
```

### Custom character set

If the character set length fits with an existing encoding scheme (e.g. base64), you can customize the characters used.

```go
import (
	"crypto/rand"
	"encoding/base32"
)

var customEncoding = base32.NewEncoding("0123456789ABCDEFGHJKMNPQRSTVWXYZ")

func generateRandomString() string {
	bytes := make([]byte, 12)
	rand.Read(bytes)
	return customEncoding.EncodeToString(bytes)
}
```

If not, you would need a high-quality [random number generator](#random-integers) to generate an integer within a custom range.

```go
const alphabet = "abcdefg"

func generateRandomString() string {
	var result string
	for i := 0; i < 12; i ++ {
		result += string(alphabet[generateRandomInt(0, len(alphabet))])
	}
	return result
}
```

## Random integers

If the range is a power of 2 (2, 4, 8, 16 etc), a simple bit masking will do the trick.

```go
bytes := make([]byte, 1)
rand.Read(bytes)
value := bytes[0] & 0x03 // random value between [0, 3]
```

For a custom range, a simple approach is to generate a very large random number compared to the maximum and use the modulo operator. Since this introduces [a modulo bias](#biases), the random integer must be sufficiently large. For example, if the maximum was 10 and we generate 32 random bits, the bias will be around 1/250,000,000 - which can be good enough for most use cases.

```go
import (
	"crypto/rand"
	"encoding/binary"
)

// Generates a random integer between [0, max).
// `max` should not be a very large number.
func generateRandomUint32(max uint32): uint32 {
	var max uint32 = 10
	bytes := make([]byte, 4)
	rand.Read(bytes)
	randUint32 := binary.BigEndian.Uint32(bytes) // Convert bytes to uint32
	return randUint32 % max
}
```

Another common approach is to multiply our maximum with a random float. This can also [introduce a bias](#biases) but it can be fine if the maximum is small enough and, unlike our first approach, the bias is spread out.

```go
func generateRandomUint32(max uint32): uint32 {
	var max uint32 = 10
	return uint32(max * generateRandomFloat32())
}
```

The safest approach then is to use rejection sampling, where a random value is repeatedly generated until its under the maximum. To increase the likelihood the random value is under the maximum, we can only generate the maximum number bits required to represent the maximum. For example, if the maximum is 10, we would only have to generate 4 bits. In the code below, we're generating a random byte and then masking the 4 leading bits to get 4 random bits (8-4=4).

```go
import (
	"crypto/rand"
	"math/big"
)

func generateRandomUint64(max *big.Int) uint64 {
	randVal := new(big.Int)
	shift := max.BitLen() % 8
	bytes := make([]byte, (max.BitLen() / 8) + 1)
	rand.Read(bytes)
	if shift != 0 {
		bytes[0] &= (1 << shift) - 1
	}
	randVal.SetBytes(bytes)
	for randVal.Cmp(max) >= 0 {
		rand.Read(bytes)
		if shift != 0 {
			bytes[0] &= (1 << shift) - 1
		}
		randVal.SetBytes(bytes)
	}
	return randVal.Uint64()
}
```

### Random floating-point numbers between 0 and 1

A common approach is to generate a random integer and divide it by a very big number. When doing this, it is crucial that the denominator is large enough, and that the denominator is a power of 2 to be accurately represented by float64.

```go
func generateRandomFloat64() float64 {
	return float64(generateRandomInteger(1<<53)) / (1 << 53)
}
```

Another approach is to generate 52 random bits for the mantissa (float64) and convert that into a float within [0, 1). This will be generally faster since it avoids division.

```go
import (
	"crypto/rand"
	"math"
)

func generateRandomFloat64() float64 {
	bytes := make([]byte, 7)
	rand.Read(bytes)
	bytes = append(make([]byte, 1), bytes...)
	// set exponent part to 0b01111111111
	bytes[0] = 0x3f
	bytes[1] |= 0xf0
	return math.Float64frombits(binary.BigEndian.Uint64(bytes)) - 1
}
```

## Biases

A very common bias seen in the wild is the modulo bias. For example, if `RANDOM_INT` is an integer in [0, 10), some numbers will appear 3 times (0, 1) while others will appear 2 times (2, 3).

```
RANDOM_INT % 4
```

To calculate the approximate bias, we can use the formula below.

```
1 / ( RANDOM_BITS - LOG2(MAX) )
```

For example, if we use 8 random bits and the maximum is 100, the approximate bias would be 0.6:

```
1 / ( 8 - LOG2(100) ) ≈ 1 / (8-6.4) ≈ 0.6
```

Multiplying the maximum with a random floating point number can introduce bias as well. In this example, `RANDOM_FLOAT` is within [0, 1) so the output will be [0, 5).

```
FLOOR( RANDOM_FLOAT * 5 )
```

Let's say `RANDOM_FLOAT` can be one of 8 numbers: 0, 0.125, 0.25, ..., 0.875. In this case, (0, 1, 3) will appear 1/4 times while (2, 4) will only appear 1/8 times. While this is an extreme example, it is essential that the random float provides enough "randomness" compared to the maximum.
