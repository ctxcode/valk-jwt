
# valk-jwt

JSON Web Tokens for [Valk](https://valk-lang.dev): signing, and verifying that refuses everything
it should. Purely written in Valk, with no os-package dependencies.

Requires Valk 0.7.5 or newer.

## Install

```
vman install github.com/ctxcode/valk-jwt
```

## Example

```rust
use jwt
use valk.core

class Session {
    sub: String
    role: String
}

let secret = core.getenv("SESSION_SECRET") !? ""

// A token for an hour
let token = jwt.encode_of(Session { sub: "user-1", role: "admin" }, secret, jwt.Algorithm.hs256, 3600)

// Reading one, which checks the signature, the algorithm and the time
let session = jwt.decode_to[Session](token, secret) ! {
    if error_is(E.code, expired) : return "your session has ended"
    return "that token is not valid"
}
println(session.role)
```

`jwt.encode` and `jwt.decode` do the same with a `json.Value` and a `Claims`, for claims that are
not a fixed shape.

## What it checks

`decode` refuses a token unless all of this holds, and says which one failed:

| code | meaning |
| --- | --- |
| `signature` | the signature does not belong to this token and this secret |
| `algorithm` | the token was signed with another algorithm than the one expected |
| `expired` | `exp` has passed |
| `not_yet_valid` | `nbf` is still in the future |
| `claim` | the audience, the issuer or a required `exp` is not what the caller asked for |
| `syntax` | not three parts, not base64url, or not JSON |

**The algorithm is taken from the caller, not from the token.** A token that says
`{"alg":"none"}`, or that was re-signed with a weaker algorithm, is refused with `algorithm`
rather than believed — that is the oldest hole in JWT libraries, and this package does not have
it. Signatures are compared in constant time.

## Options

```rust
jwt.decode(token, secret, .{
    algorithm: jwt.Algorithm.hs512   // the algorithm the token must use
    leeway_seconds: 30               // clock difference to forgive on exp and nbf
    audience: "the-api"              // aud must be this, or a list holding it
    issuer: "the-login-service"      // iss must be this
    require_expiry: true             // refuse a token that never expires
}) ! panic("%{E.message}")
```

## Claims

```rust
claims.subject      // sub
claims.issuer       // iss
claims.audience     // aud; the first one when aud is a list
claims.audiences    // every aud value
claims.id           // jti
claims.expires_at   // exp, in seconds since the epoch
claims.not_before   // nbf
claims.issued_at    // iat
claims.string("role")   // a claim of your own; also int, bool, get and has
claims.data             // the whole object as a json.Value
claims.to_type[T]() !>  // read into a class of your own
```

`jwt.decode_unverified(token)` reads a token without checking anything, for looking at one you
already refused or for reading `iss` before you know which secret it needs. Nothing it returns
may be trusted.

## Algorithms

`HS256`, `HS384` and `HS512`: the secret that signs is the secret that verifies, which suits a
program that signs its own sessions. These go through `encode` and `decode`.

Keep the secret out of the source and out of the repository: read it from the environment or
from a file the program is given. A secret shorter than the hash it feeds (32 bytes for HS256)
weakens the signature.

To change the secret without refusing every token it signed, give each secret an id. `encode`
writes it into the token as `kid`, and `decode_with_secrets` checks a token with the secret its
`kid` names. Keep an old secret in the set until its tokens have expired:

```rust
let secrets = jwt.SecretSet {}
secrets.add("2026-09", new_secret)
secrets.add("2026-06", old_secret)

let token = jwt.encode(claims, new_secret, jwt.Algorithm.hs256, 3600, "2026-09")
let checked = jwt.decode_with_secrets(token, secrets) ! panic("%{E.message}")
```

`RS256` to `RS512`, `PS256` to `PS512`, `ES256` to `ES512` and `EdDSA`: a private key signs and
its public key verifies, so a token can be checked by a program that could never make one. These
go through `sign` and `verify`, with the keys of `valk.crypto`:

```rust
use jwt
use valk.json
use valk.crypto

let private = crypto.PrivateKey.from_pem(private_pem) ! panic("not a private key")
let token = jwt.sign(json.from(Map[String]{ "sub" => "user-1" }), private, jwt.Algorithm.es256, 3600, "key-1") ! panic("%{E.message}")

let public = crypto.PublicKey.from_pem(public_pem) ! panic("not a public key")
let claims = jwt.verify(token, public, jwt.Options { algorithm: jwt.Algorithm.es256 }) ! panic("%{E.message}")
```

A token from an OAuth or OpenID Connect provider is checked with the keys it publishes as a JWKS
document; `verify_with_keys` picks the key the token's `kid` names:

```rust
let keys = jwt.KeySet.from_jwks(jwks_text) ! panic("%{E.message}")
let claims = jwt.verify_with_keys(token, keys, jwt.Options { algorithm: jwt.Algorithm.rs256, issuer: "https://accounts.google.com" }) ! panic("%{E.message}")
```

A secret and a key never stand in for each other: `decode` refuses the key algorithms and
`verify` the HMAC ones, which rules out the attack where a public key is used as an HMAC secret.

## Development

`make test` runs the suite, `make example` runs the example, `make lint` checks the sources and
`make docs` regenerates the API documentation. The tokens the suite writes, with every algorithm,
were also checked against an independent implementation of the standard, and it reads tokens that
implementation signed.
