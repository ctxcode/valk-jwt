
# Documentation

Namespaces: [main](#main)

---

# main

## Errors for 'main'

```js
// Thrown when a token cannot be read or cannot be trusted.
+ error Error (syntax, signature, algorithm, expired, not_yet_valid, claim) payload { message: String }
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

Every one of these means the token must be refused; they are apart so that a program can
tell an expired session from a forged one.

## Enums for 'main'

```js
// The algorithms this package signs and verifies with.
+ enum Algorithm { hs256, hs384, hs512 }
```

### Algorithm

The algorithms this package signs and verifies with.

These are the HMAC ones, where signing and verifying use the same secret. The RSA and ECDSA
algorithms, where a public key verifies what a private key signed, are not supported yet.

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

### decode_to

Reads a token into a class or struct of your own, checking it the way `decode` does.

### decode_unverified

Reads what a token says without checking anything at all.

Nothing that comes out of this may be trusted: anyone can write a token. It is for looking at
a token you already refused, or for reading the `iss` of one before you know which secret to
check it with.

### encode

Signs claims into a token.

`expires_in_seconds` above 0 adds `exp` and `iat`, which is what a session token wants; a
token without `exp` is valid until the secret changes.

```valk
let claims = json.new_object()
let token = jwt.encode(json.from(.{ "sub" => "user-1" }), secret, jwt.Algorithm.hs256, 3600)
```

### encode_of

Signs a class or struct of your own into a token, as `json.from` would write it.

```valk
class Session {
    sub: String
    role: String
}

let token = jwt.encode_of(Session { sub: "user-1", role: "admin" }, secret, jwt.Algorithm.hs256, 3600)
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

`aud`: who the token is for.

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

### Options

What a token has to satisfy to be accepted.

The algorithm is checked against what the caller expects rather than against what the token
says, which is what keeps a token from choosing how it is verified.

#### algorithm

The algorithm the token must be signed with.

#### audience

The audience the token must carry, or "" to accept any.

#### issuer

The issuer the token must carry, or "" to accept any.

#### leeway_seconds

How much clock difference to allow when checking `exp` and `nbf`, in seconds.

#### require_expiry

Whether a token without `exp` is refused.
