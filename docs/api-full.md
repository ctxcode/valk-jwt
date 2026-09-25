
# Documentation

Namespaces: [main](#main)

---

# main

## Errors for 'main'

```js
// Thrown when a token cannot be read or cannot be trusted.
+ error Error (syntax, signature, algorithm, expired, not_yet_valid, claim, key) payload { message: String }
```

### Error

Thrown when a token cannot be read or cannot be trusted.

- `syntax`: the token is not three parts separated by dots, or a part is not valid base64url
  or not valid JSON.
- `signature`: the signature does not belong to this token and this secret. A token that was
  changed by anyone reaches you as this.
- `algorithm`: the token was signed with another algorithm than the one asked for, `none`
  included.
- `expired`: `exp` has passed.
- `not_yet_valid`: `nbf` lies in the future.
- `claim`: a claim the caller asked for is missing or is not what it asked for, such as an
  audience or an issuer that does not match.
- `key`: no key to check the token with: a `KeySet` without the token's `kid`, or a key that
  does not fit the algorithm, such as an EC key for RS256. Also thrown by `sign` for a key
  that cannot sign with the algorithm.

Every one of these means the token must be refused; they are apart so that a program can
tell an expired session from a forged one.

## Enums for 'main'

```js
// The algorithms this package signs and verifies with.
+ enum Algorithm { hs256, hs384, hs512, rs256, rs384, rs512, ps256, ps384, ps512, es256, es384, es512, eddsa }
```

### Algorithm

The algorithms this package signs and verifies with.

The HMAC ones (`hs*`) sign and verify with one secret, through `encode` and `decode`. The
others sign with a private key and verify with its public key, through `sign` and
`verify`: RSA with PKCS#1 v1.5 (`rs*`) or PSS padding (`ps*`), ECDSA on P-256, P-384 and
P-521 (`es256`, `es384`, `es512`), and Ed25519 (`eddsa`).

## Functions for 'main'

```js
// Reads base64url, with or without the padding. Throws `syntax` when the text is not that.
+ fn base64url_decode(text: String) String !Error
// Writes bytes the way a token writes them: base64url, without the `=` padding.
+ fn base64url_encode(data: String) String
// Reads a token, checks its signature, and returns what it says.
+ fn decode(token: String, secret: String, options: Options (.{})) Claims !Error
// Reads a token into a class or struct of your own, checking it the way `decode` does.
+ fn decode_to[T](token: String, secret: String, options: Options (.{})) T !Error
// Reads what a token says without checking anything at all.
+ fn decode_unverified(token: String) Claims !Error
// Checks a token with the secret its `kid` names in `secrets`, as `decode` checks it with one secret.
+ fn decode_with_secrets(token: String, secrets: SecretSet, options: Options (.{})) Claims !Error
// Signs claims into a token.
+ fn encode(claims: Value, secret: String, algorithm: Algorithm (Algorithm.hs256), expires_in_seconds: uint (0), key_id: String ("")) String
// Signs a class or struct of your own into a token, as `json.from` would write it.
+ fn encode_of(claims: $T, secret: String, algorithm: Algorithm (Algorithm.hs256), expires_in_seconds: uint (0), key_id: String ("")) String
// Signs claims into a token with a private key, for the RSA, ECDSA and EdDSA algorithms.
+ fn sign(claims: Value, key: PrivateKey, algorithm: Algorithm, expires_in_seconds: uint (0), key_id: String ("")) String !Error
// Signs a class or struct of your own into a token with a private key; see `sign`.
+ fn sign_of(claims: $T, key: PrivateKey, algorithm: Algorithm, expires_in_seconds: uint (0), key_id: String ("")) String !Error
// Reads a token, checks its signature with a public key, and returns what it says.
+ fn verify(token: String, key: PublicKey, options: Options) Claims !Error
// Reads a token into a class or struct of your own, checking it the way `verify` does.
+ fn verify_to[T](token: String, key: PublicKey, options: Options) T !Error
// Checks a token with the key its `kid` names in `keys`, as `verify` checks it with one key.
+ fn verify_with_keys(token: String, keys: KeySet, options: Options) Claims !Error
```

### base64url_decode

Reads base64url, with or without the padding. Throws `syntax` when the text is not that.

### base64url_encode

Writes bytes the way a token writes them: base64url, without the `=` padding.

Every part of a token is written this way, so a program that builds a header of its own, or
reads a key out of a JWK, needs the same encoding.

### decode

Reads a token, checks its signature, and returns what it says.

The signature is checked against the algorithm the caller expects, not against the one the
token names: a token cannot choose how it is verified, which is the mistake `alg: none`
attacks are built on. `exp` and `nbf` are checked as well, and the audience and issuer when
`options` names them.

```valk
let claims = jwt.decode(token, secret) ! {
    if error_is(E.code, expired) : return "your session has ended"
    return "that token is not valid"
}
```

This checks the HMAC algorithms, signed with a secret; tokens signed with a private key are
checked by `verify`. An `options.algorithm` of another kind throws `algorithm`.

### decode_to

Reads a token into a class or struct of your own, checking it the way `decode` does.

### decode_unverified

Reads what a token says without checking anything at all.

Nothing that comes out of this may be trusted: anyone can write a token. It is for looking at
a token you already refused, or for reading the `iss` of one before you know which secret to
check it with.

### decode_with_secrets

Checks a token with the secret its `kid` names in `secrets`, as `decode` checks it with one
secret.

Throws `key` when the token has no `kid` or the set has no secret with it.

### encode

Signs claims into a token.

`expires_in_seconds` above 0 adds `exp` and `iat`, which is what a session token wants; a
token without `exp` is valid until the secret changes.

```valk
let claims = json.new_object()
let token = jwt.encode(json.from(.{ "sub" => "user-1" }), secret, jwt.Algorithm.hs256, 3600)
```

`key_id` becomes the header's `kid`, so a server that rotates its secrets can tell which one
signed the token; see `SecretSet`.

Only the HMAC algorithms sign with a secret: another algorithm panics, those sign with a
private key through `sign`.

### encode_of

Signs a class or struct of your own into a token, as `json.from` would write it.

```valk
class Session {
    sub: String
    role: String
}

let token = jwt.encode_of(Session { sub: "user-1", role: "admin" }, secret, jwt.Algorithm.hs256, 3600)
```

### sign

Signs claims into a token with a private key, for the RSA, ECDSA and EdDSA algorithms.

`key_id` becomes the header's `kid`, so a verifier that holds several keys, such as a
`KeySet`, can tell which one to use. Throws `algorithm` for an HMAC algorithm, which
`encode` signs with a secret, and `key` when the key does not fit the algorithm, such as an
EC key for RS256 or a P-384 key for ES256.

```valk
let key = crypto.PrivateKey.from_pem(pem) ! panic("not a key")
let token = jwt.sign(json.from(Map[String]{ "sub" => "user-1" }), key, jwt.Algorithm.es256, 3600, "key-2024") ! panic("%{E.message}")
```

### sign_of

Signs a class or struct of your own into a token with a private key; see `sign`.

### verify

Reads a token, checks its signature with a public key, and returns what it says.

For the RSA, ECDSA and EdDSA algorithms; `options.algorithm` says which one the token must
use, and the claims are checked as `decode` checks them. Throws `algorithm` for an HMAC
algorithm, which `decode` checks with a secret, and `key` when the key does not fit the
algorithm.

```valk
let key = crypto.PublicKey.from_pem(pem) ! panic("not a key")
let claims = jwt.verify(token, key, jwt.Options { algorithm: jwt.Algorithm.rs256 }) ! panic("%{E.message}")
```

### verify_to

Reads a token into a class or struct of your own, checking it the way `verify` does.

### verify_with_keys

Checks a token with the key its `kid` names in `keys`, as `verify` checks it with one key.

Throws `key` when the token has no `kid` or the set has no key with it, and `algorithm`
when the key's JWK is limited to another algorithm than `options.algorithm`.

## Classes for 'main'

```js
// What a token says.
+ class Claims {
    // `aud`: who the token is for; the first one when `aud` is a list.
    + audience: String
    // Every `aud` value: one for a plain `aud`, all of them for a list.
    + audiences: Array[String]
    // Everything the token says, the claims above included.
    + data: Value
    // `exp`: the second after which the token is no longer valid, or 0 when it never expires.
    + expires_at: uint
    // `jti`: the id of the token.
    + id: String
    // `iat`: the second the token was made, or 0.
    + issued_at: uint
    // `iss`: who made the token.
    + issuer: String
    // `nbf`: the second before which the token is not valid yet, or 0.
    + not_before: uint
    // `sub`: who the token is about.
    + subject: String

    // Returns a claim as a bool, or false.
    + fn bool(name: String) bool
    // Returns a claim by name, json null when it is not there.
    + fn get(name: String) Value
    // Returns whether the token says anything about this claim.
    + fn has(name: String) bool
    // Returns a claim as a number, or 0.
    + fn int(name: String) int
    // Returns whether the token is expired, allowing for `leeway_seconds` of clock difference.
    + fn is_expired(leeway_seconds: uint (0)) bool
    // Returns a claim as text, or "" when it is not there.
    + fn string(name: String) String
    // Reads the claims into a class or struct of your own, as `json.Value.to_type` does.
    + fn to_type[T]() T !Error
}
```

### Claims

What a token says.

The claims the standard names are read into their own fields; `data` holds the whole object,
including whatever else was put in it.

```valk
let claims = jwt.decode(token, secret) ! panic("%{E.message}")
println(claims.subject)                       // "sub"
println(claims.data["role"].string)           // a claim of your own
```

#### audience

`aud`: who the token is for; the first one when `aud` is a list.

#### audiences

Every `aud` value: one for a plain `aud`, all of them for a list.

#### data

Everything the token says, the claims above included.

#### expires_at

`exp`: the second after which the token is no longer valid, or 0 when it never expires.

#### id

`jti`: the id of the token.

#### issued_at

`iat`: the second the token was made, or 0.

#### issuer

`iss`: who made the token.

#### not_before

`nbf`: the second before which the token is not valid yet, or 0.

#### subject

`sub`: who the token is about.

#### bool

Returns a claim as a bool, or false.

#### get

Returns a claim by name, json null when it is not there.

#### has

Returns whether the token says anything about this claim.

#### int

Returns a claim as a number, or 0.

#### is_expired

Returns whether the token is expired, allowing for `leeway_seconds` of clock difference.

#### string

Returns a claim as text, or "" when it is not there.

#### to_type

Reads the claims into a class or struct of your own, as `json.Value.to_type` does.

```js
// Public keys by key id, as an OAuth or OpenID Connect provider publishes them in a JWKS document, for `verify_with_keys`.
+ class KeySet {
    // Adds `key` under `key_id`, replacing a key with that id.
    + fn add(key_id: String, key: PublicKey) void
    // Reads a JWKS document, `{"keys": [...]}`.
    + static fn from_jwks(text: String) KeySet !Error
    // Returns the key with id `key_id`, or null.
    + fn get(key_id: String) ?PublicKey
    // Returns the ids of the keys in the set.
    + fn key_ids() Array[String]
}
```

### KeySet

Public keys by key id, as an OAuth or OpenID Connect provider publishes them in a JWKS
document, for `verify_with_keys`.

```valk
let keys = jwt.KeySet.from_jwks(jwks_text) ! panic("%{E.message}")
let claims = jwt.verify_with_keys(token, keys, jwt.Options { algorithm: jwt.Algorithm.rs256 }) ! panic("%{E.message}")
```

#### add

Adds `key` under `key_id`, replacing a key with that id.

#### from_jwks

Reads a JWKS document, `{"keys": [...]}`.

Keys without a `kid`, keys for encryption (`use` other than `sig`) and keys of a type
`crypto.PublicKey.from_jwk` does not read are left out, so a document may list more than
this package uses. Throws `syntax` when the text is not a JWKS document.

#### get

Returns the key with id `key_id`, or null.

#### key_ids

Returns the ids of the keys in the set.

```js
// What a token has to satisfy to be accepted.
+ class Options {
    // The algorithm the token must be signed with.
    + algorithm: Algorithm
    // The audience the token must carry, or "" to accept any. When `aud` is a list, the audience has to be one of its values.
    + audience: String
    // The issuer the token must carry, or "" to accept any.
    + issuer: String
    // How much clock difference to allow when checking `exp` and `nbf`, in seconds.
    + leeway_seconds: uint
    // Whether a token without `exp` is refused.
    + require_expiry: bool
}
```

### Options

What a token has to satisfy to be accepted.

The algorithm is checked against what the caller expects rather than against what the token
says, which is what keeps a token from choosing how it is verified.

#### algorithm

The algorithm the token must be signed with.

#### audience

The audience the token must carry, or "" to accept any. When `aud` is a list, the
audience has to be one of its values.

#### issuer

The issuer the token must carry, or "" to accept any.

#### leeway_seconds

How much clock difference to allow when checking `exp` and `nbf`, in seconds.

#### require_expiry

Whether a token without `exp` is refused.

```js
// HMAC secrets by key id, for rotating the secret without refusing the tokens signed before.
+ class SecretSet {
    // Adds `secret` under `key_id`, replacing a secret with that id.
    + fn add(key_id: String, secret: String) void
    // Returns the secret with id `key_id`, or null.
    + fn get(key_id: String) ?String
    // Returns the ids of the secrets in the set.
    + fn key_ids() Array[String]
}
```

### SecretSet

HMAC secrets by key id, for rotating the secret without refusing the tokens signed before.

Sign new tokens with the newest secret and its id (the `key_id` of `encode`), and keep the
older secrets in the set until the tokens they signed have expired.

```valk
let secrets = jwt.SecretSet {}
secrets.add("2026-09", new_secret)
secrets.add("2026-06", old_secret)
let token = jwt.encode(claims, new_secret, jwt.Algorithm.hs256, 3600, "2026-09")
let checked = jwt.decode_with_secrets(token, secrets) ! panic("%{E.message}")
```

#### add

Adds `secret` under `key_id`, replacing a secret with that id.

#### get

Returns the secret with id `key_id`, or null.

#### key_ids

Returns the ids of the secrets in the set.
