
# Documentation

Namespaces: [main](#main)

---

# main

## Errors for 'main'

```js
// Thrown when a token cannot be read or cannot be trusted.
+ error Error (syntax, signature, algorithm, expired, not_yet_valid, claim, key) payload { message: String }
```

## Enums for 'main'

```js
// The algorithms this package signs and verifies with.
+ enum Algorithm { hs256, hs384, hs512, rs256, rs384, rs512, ps256, ps384, ps512, es256, es384, es512, eddsa }
```

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
