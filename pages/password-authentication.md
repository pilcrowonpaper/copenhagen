---
title: "Password authentication"
---

# Password authentication

## Table of contents

- [Input validation](#input-validation)
	- [Checking for compromised passwords](#checking-for-compromised-passwords)
- [Password storage](#password-storage)
	- [Argon2id](#argon2id)
	- [Scrypt](#scrypt)
	- [Bcrypt](#bcrypt)
- [Brute-force attacks](#brute-force-attacks)
- [Error handling](#error-handling)
- [Other considerations](#other-considerations)

## Input validation

- Passwords must be at least 8 characters long.
- Do not set the maximum password length too low. Anywhere around 64-256 characters is a good maximum.
- Do not silently modify or truncate the input.
- All valid Unicode characters should be allowed, including whitespace.
- Use libraries like [`zxcvbn`](https://github.com/dropbox/zxcvbn) to check for weak passwords.
- Detect leaked passwords with APIs such as [haveibeenpwned](https://haveibeenpwned.com/API/v3).

### Checking for compromised passwords

A free service called [haveibeenpwned](https://haveibeenpwned.com/API/v3) can be used to check a password against past leaks. Hash the password with SHA-1 (hex encoded) and send the first 5 characters.

```
GET https://api.pwnedpasswords.com/range/12345
```

The API will provide a list of hashed password suffixes beginning with the provided 5 characters

```
ec68dea7966a1ea2ba9408be4dcc409884f
248b2dddf14a111b9d08b906d06224a0a79
f10a49ecd2ada17a120dc359f162b84e12c
```

## Password storage

Passwords must be salted and hashed before storage. We recommend using [Argon2id](#argon2id) with salting.

In the most basic form, hashing is a one way process to generate a unique representation of the input. The same input should result in the same hash. Unlike encryption, it is not reversible - you can't get the original data from the hash. Popular examples include MD5, SHA-1, and SHA-256 - **DO NOT USE THESE FOR PASSWORDS**.

Hashing ensures that if you suffer a data breach, hackers won't be able to get the original password. This is especially important if the breach was limited in scope. Even if they were only able to read the user table, they'll effectively have access to everything once they get hold of user passwords. More importantly, however, it protects your users from further harm. Users often reuse passwords. With leaked passwords, hackers can gain access to user accounts in other applications as well.

However, a big issue with passwords is that they're aren't truly random. Technically there are 62^8 possible 8 character alphanumeric passwords, but reality is that most passwords use common words and names with maybe some numbers at the end. This significantly reduces the number of combinations to test when brute-forcing passwords. 

As such, slow hashing algorithms specifically designed for passwords are used. Common hashing algorithms like SHA-256 are designed to be fast as possible.

Even when using a slow algorithm, a table of precomputed hashes of common passwords called a rainbow table can be used. Salting is a common technique to prevent these attacks by adding random values to each password before hashing. The salt must be generated using a cryptographically-secure random generator and it should have at least 120 bits of entropy.

```
salt = randomValues()
hash = hashPassword(password + salt) + salt
```

Another option is peppering where you use a secret key when hashing the password. Whereas in salts are stored alongside the hashes, the secret key is stored in a separate location. Rolling your own hashing mechanism can be a bad idea so this should only be done if the algorithm you use supports it.

When comparing password hashes, use constant time comparison instead of `==`. This ensures your application is not vulnerable to timing-based attacks, where an attacker can extract information using how long it took to compare the password with the hash.

```go
import (
	"crypto/subtle"
	"golang.org/x/crypto/argon2"
)

var storedHash []byte
var password []byte
hash := argon2.IDKey(password, salt, 2, 19*1024, 1, 32)

if (subtle.ConstantTimeCompare(hash, storedHash)) {
	// Valid password.
}
```

Argon2id should be your first choice, followed by Scrypt, and then Bcrypt for legacy systems.

Password hashing is resource intensive and is vulnerable to denial-of-service (DoS) attacks.

### Argon2id

Argon2 was the winner of the 2013 Password Hashing Competition and has 3 versions: Argon2i, Argon2d, and Argon2id. Argon2id should be your default option as it provides a good balance between resisting both side-channel and GPU-based attacks. Recommended minimum parameters:

- `memorySize`: 19456 (19 MB)
- `iterations`: 2
- `parallelism`: 1

Optionally use the `secret` parameter to pepper your hashes. [See OWASP for details](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html#argon2id).

### Scrypt

Recommended minimum parameters:

- `N`: 16384
- `P`: 16
- `r`: 1
- `dkLen`: 64

[See OWASP for details](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html#scrypt).

### Bcrypt

The work factor should be at minimum 10.

Bcrypt has a maximum input length of 72 bytes, and some implementation may have a limit as low as 50 bytes. Pre-hashing the password with algorithms like SHA-256/512 is not recommended as some implementations of Bcrypt are not built to handle null bytes. Do not attempt to implement peppering by using HMAC either. Use algorithms like [Argon2id](#argon2id) or [Scrypt](#scrypt) instead if you need to support longer passwords.

[See OWASP for details](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html#bcrypt).

## Brute-force attacks

Passwords are susceptible to brute-force attacks. There are mainly 2 approaches to brute-forcing:

1. The attacker tries a bunch of common passwords.
2. The attacker targets specific accounts using leaked passwords (credential stuffing).

[Multi-factor authentication (MFA)](/mfa.md) is the best defense against brute-force attacks. While it doesn't prevent brute-force attacks themselves, it does make it near pointless to do. Users should be recommended to enable MFA and it should be required for security-critical applications.

IP-based throttling should always be implemented. A basic example is to block all attempts from an IP address for 10 minutes after they fail 10 consecutive attempts. Other ideas include increasing the lockout period on each lockout, and gradually allowing new attempts at a set interval after a lockout. This also prevents DOS attacks as password hashing is resource-intensive. An identifier-based throttling can also be implemented on top of IP-based throttling, though this can introduce DoS vulnerabilities (see [device cookies](https://owasp.org/www-community/Slow_Down_Online_Guessing_Attacks_with_Device_Cookies)).

Another layer of security you can implement is bot detection using tests like Captchas.

Finally, ensure a certain strength of passwords for users. Make sure passwords aren't weak and that they haven't been part of previous leaks. See the [Password validation](#password-validation) section.

## Error handling

As a good rule of thumb, error messages should be vague and generic. For example, a login form should display "Incorrect username or password" instead of "Incorrect username" or "Incorrect password." However, if your users' usernames or emails are public knowledge, it may make more sense to return a more specific error message. In most cases, anyone can already determine if an email address or username is valid by checking it in the registration form.

Even when returning a generic message, it may be possible to determine if a user exists or not by checking the response times of the login form. For example, if you only validate the password when the username is valid.

## Other considerations

- Do not prevent users from copy-pasting passwords as it discourages users from using password managers.
- Ask for the current password when a user attempts to change their password.
- [Open redirect](/open-redirect.md).