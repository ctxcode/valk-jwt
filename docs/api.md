
# Documentation

Namespaces: [main](#main)

---

# main

## Errors for 'main'

```js
// Thrown when a token cannot be read or cannot be trusted.
+ error Error (syntax, signature, algorithm, expired, not_yet_valid, claim) payload { message: String }
```

## Enums for 'main'

```js
// The algorithms this package signs and verifies with.
+ enum Algorithm { hs256, hs384, hs512 }
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
// Signs claims into a token.
+ fn encode(claims: Value, secret: String, algorithm: Algorithm (Algorithm.hs256), expires_in_seconds: uint (0)) String
// Signs a class or struct of your own into a token, as `json.from` would write it.
+ fn encode_of(claims: $T, secret: String, algorithm: Algorithm (Algorithm.hs256), expires_in_seconds: uint (0)) String
```

## Classes for 'main'

```js
// What a token says.
+ class Claims {
    // `aud`: who the token is for.
    + audience: String
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
// What a token has to satisfy to be accepted.
+ class Options {
    // The algorithm the token must be signed with.
    + algorithm: Algorithm
    // The audience the token must carry, or "" to accept any.
    + audience: String
    // The issuer the token must carry, or "" to accept any.
    + issuer: String
    // How much clock difference to allow when checking `exp` and `nbf`, in seconds.
    + leeway_seconds: uint
    // Whether a token without `exp` is refused.
    + require_expiry: bool
}
```
