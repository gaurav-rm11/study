# JSON Web Tokens (JWTs)

> **TL;DR** — A JWT is a compact, signed JSON token carrying claims about a subject (user id, scopes, expiry). Any service holding the verification key can validate it locally — no session-store lookup — which makes it the usual bearer-token format for APIs, microservices and OAuth/OIDC.

## How it works

```text
base64url(header) . base64url(payload) . base64url(signature)

header     {"alg":"RS256","typ":"JWT","kid":"2026-10"}
payload    {"iss":"https://auth.example.com","sub":"42","aud":"orders-api",
            "scope":"orders:read","iat":1792499100,"exp":1792500000}
signature  sign(privateKey, base64url(header) + "." + base64url(payload))

Client ──credentials──► Auth server ──signed JWT──► Client
Client ──Authorization: Bearer <jwt>──► API: verify signature + claims locally ✓
```

- **Encoded, not encrypted:** base64url is reversible — anyone holding the token can read the payload (paste it into jwt.io). The signature proves integrity and origin, not secrecy.
- **Registered claims (RFC 7519):** `iss` issuer · `sub` subject · `aud` audience (intended recipients — anyone else must reject it) · `exp` expiry · `nbf` not-before · `iat` issued-at · `jti` unique token id. All optional in the spec — your validation policy makes them mandatory. Roles, tenant and `scope` are custom claims.
- **Flow:** the client sends `Authorization: Bearer <jwt>` on every call; each service checks signature + claims with a cached key — no session lookup, no per-request call to the auth server.

## Signing algorithms

| `alg` | Scheme | Keys | Notes |
|---|---|---|---|
| HS256/384/512 | HMAC-SHA2 | one shared secret signs *and* verifies | every verifier can also **mint** tokens — only when issuer = verifier; weak secrets crack offline |
| RS256 | RSA PKCS#1 v1.5 | private key signs, public key verifies | most widely supported (OIDC default); 256-byte signature with 2048-bit keys |
| PS256 | RSA-PSS | same | modern RSA padding; prefer over RS256 when both sides support it |
| ES256 | ECDSA P-256 | same | 64-byte signature; a reused per-signature nonce leaks the private key |
| EdDSA | Ed25519 | same | fast, deterministic, small; narrower library/IdP support |

## Advanced

- **Key rotation (JWKS + `kid`):** the issuer publishes public keys as a JSON Web Key Set (OIDC: `jwks_uri` in `/.well-known/openid-configuration`); the header's `kid` picks the key. Rotate: publish the new key → wait for verifier caches to refresh → sign with it → drop the old key after the longest token lifetime. Verifiers cache the JWKS and refetch on an unknown `kid`, rate-limited so forged `kid`s can't cause fetch storms.
- **JWS vs JWE:** a "JWT" is nearly always a **JWS** (RFC 7515: signed, 3 parts, readable). **JWE** (RFC 7516) encrypts the claims (5 parts) when clients or intermediaries must not read them; nest sign-then-encrypt if you need both.
- **Validation checklist (RFC 7519 + 8725):** (1) verify the signature with an algorithm from a server-side allow-list bound to the key — never trust the header's `alg`; (2) reject `alg: none`; (3) check `exp`/`nbf` with a small clock-skew leeway (≤ 1–2 min); (4) `iss` = expected issuer; (5) `aud` includes *this* service; (6) explicit typing (`typ: at+jwt`, RFC 9068) so an ID token can't pass as an access token; (7) take keys only from the configured issuer — ignore `jku`/`x5u`/embedded `jwk` headers, and treat `kid` as untrusted input (path traversal, SQL injection in key lookups).
- **Algorithm confusion (RS256 → HS256):** if the library trusts the header's `alg`, an attacker signs a forged token with HS256 using the server's RSA *public* key as the HMAC secret; the server "verifies" with that same key and accepts it. Pinning the algorithm per key kills the attack.
- **Decode ≠ verify:** `jwt.decode()` (Node `jsonwebtoken`) and .NET's `JwtSecurityTokenHandler.ReadJwtToken` only parse. Classic bug: the gateway verifies, then internal services trust claims from anyone who can reach them directly.
- **Revocation:** a JWT stays valid until `exp` — logout, password reset or a ban doesn't kill it. Fixes: short-lived access tokens (5–15 min) + **refresh tokens** stored server-side, **rotated** on every use with **reuse detection** (an already-used refresh token reappears → revoke the whole token family); a **`jti` denylist** in Redis with TTL = remaining lifetime (stays tiny); a **per-user token version** claim checked against a cached user record — bump it to log out every device.
- **Browser storage:** `localStorage` is readable by any script → one XSS = stolen token. An `HttpOnly; Secure; SameSite` cookie hides it from JS but is sent automatically → add CSRF defences (SameSite=Lax/Strict, anti-CSRF token, `Origin` check). Strongest for SPAs: a **BFF** (backend-for-frontend) keeps tokens server-side and gives the browser only a session cookie.
- **Size & contents:** the token rides on every request; role/group lists can push it past proxy header limits (often ~8 KB). No secrets or PII — payloads end up in logs, caches and jwt.io.
- **JWT vs opaque tokens:** an opaque token is a random handle the API resolves via **introspection** (RFC 7662) → instant revocation and nothing readable client-side, but a network call per request (cache briefly). Hybrid "phantom token": clients hold opaque tokens; the gateway introspects and forwards a JWT internally.
- **OAuth 2.0 / OIDC:** OAuth 2.0 issues **access tokens** for calling APIs (format unspecified; JWT profile = RFC 9068). OpenID Connect adds the **ID token** — always a JWT, `aud` = the client app — telling the *client* who logged in. Never send an ID token to an API as a bearer token.
- **Service-to-service calls:** forwarding the user's token down the call chain forces a broad `aud` and lets any hop replay it anywhere; exchange it for a down-scoped token per downstream service (OAuth 2.0 Token Exchange, RFC 8693), and use client-credentials tokens when no user is involved.
- **Sender-constrained tokens:** a bearer token works for whoever holds it. **DPoP** (RFC 9449): the client signs a per-request proof JWT (HTTP method, URL, `jti`, `iat`) with its private key, and the access token carries that key's thumbprint (`cnf.jkt`). **mTLS** (RFC 8705) binds the token to the client certificate (`cnf.x5t#S256`). A stolen token is useless without the key.

## Trade-offs

- ✅ Stateless verification: no shared session store or auth-server round-trip; scales across services, regions and organisations (federation via OIDC).
- ✅ Standard, self-contained format with mature libraries everywhere; identity and scopes travel with each request.
- ❌ Revocation and stale claims (a removed role keeps working) until expiry, unless you add state back.
- ❌ Larger than a session id on every request, and readable by anyone who holds it.
- ❌ Many footguns (`none`, algorithm confusion, missing `aud`, decode-only) — use a vetted library or IdP, never hand-rolled crypto.

## In the wild

Auth0, Okta, Microsoft Entra ID, AWS Cognito, Keycloak, Firebase Auth issue JWT access/ID tokens · Kubernetes service-account tokens are JWTs · Edge validation: Kong JWT plugin, AWS API Gateway JWT authorizers, Envoy `jwt_authn` · .NET: `Microsoft.AspNetCore.Authentication.JwtBearer` (Microsoft.IdentityModel), Duende IdentityServer, OpenIddict · Node: `jose`, `jsonwebtoken`, `passport-jwt`.

## Gotchas

- **ASP.NET Core's default `ClockSkew` is 5 minutes** — a 5-minute token effectively lives ~10. Lower it explicitly.
- **No `aud` check** → a token minted for service A is accepted by service B from the same issuer.
- **Weak HS256 secrets** can be brute-forced offline from one captured token (hashcat); use ≥ 256 random bits or asymmetric keys.
- **Tokens in URLs** leak into logs, browser history and `Referer`; send them in the `Authorization` header (the WebSocket handshake is the usual exception — keep those tokens very short-lived).
- **`exp`/`nbf`/`iat` are seconds since the epoch** (NumericDate), not milliseconds: a hand-built `exp: Date.now() + 3600` in Node mints a token that effectively never expires.

## Related

[APIs](01-apis.md) · [API Gateways](03-api-gateways.md) · [Stateful vs Stateless](../01-foundations/04-stateful-vs-stateless.md) · [Webhooks](05-webhooks.md) · [Long Polling vs WebSockets](06-long-polling-vs-websockets.md) · [Service Mesh](../08-microservices-and-reliability/02-service-mesh.md)

## Read more

- [RFC 7519 — JSON Web Token (JWT)](https://www.rfc-editor.org/rfc/rfc7519)
- [RFC 8725 — JSON Web Token Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725)
- [JWT attacks — PortSwigger Web Security Academy](https://portswigger.net/web-security/jwt)

<!-- nav -->

---

[← API Gateways](03-api-gateways.md) · [Index](../README.md) · [Webhooks →](05-webhooks.md)
