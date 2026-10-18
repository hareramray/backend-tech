# OAuth, OpenID Connect, and token lifecycle

## Purpose and prerequisites

OAuth is a framework for delegated authorization. OpenID Connect, or OIDC, adds an identity layer. A JWT is a token format. These solve different problems: an OAuth access token need not be a JWT, and possession of an arbitrary JWT does not establish that it was issued for your API.

Prerequisites: [authentication and sessions](02-authentication-passwords-and-sessions.md), HTTPS, redirects, and a basic understanding of public-key signatures. The lab requires a test identity-provider tenant and a maintained OIDC client library. It is a configuration and verification exercise, not a complete identity provider implementation.

## Actors and credentials

The resource owner grants access; the client requests it; the authorization server issues credentials; the resource server hosts the protected API. One organization may operate several of these roles, but their trust relationships remain distinct.

| Item | Intended consumer | Purpose |
|---|---|---|
| Authorization code | Client's token exchange | Short-lived result of authorization |
| ID token | OIDC client | Claims about an authentication event |
| Access token | Intended resource server | Permission to call a resource |
| Refresh token | Authorization server | Obtain replacement access tokens |
| Application session cookie | Your own application | Maintain a browser session |

Never send an ID token as an API access token merely because both are JWTs. Do not accept an access token intended for a different resource. Use `(issuer, subject)` as the stable external identity key; email addresses can change and are not universally unique across identity providers. OIDC defines ID-token claims and client validation rules. [OIDC Core](https://openid.net/specs/openid-connect-core-1_0.html).

## Authorization code flow with PKCE

For a typical browser application:

1. Generate a fresh login transaction with an unpredictable `state`, OIDC `nonce`, and PKCE verifier. Store them securely with a short expiration and bind them to the initiating browser.
2. Derive the `S256` PKCE challenge and redirect to the configured authorization endpoint. Request narrowly required scopes and a preregistered redirect URI.
3. On callback, reject mismatched, expired, or reused login transactions. Validate the returned `state` and the expected authorization-server identity according to the library's flow.
4. Exchange the one-time code at the configured token endpoint using the verifier. A confidential backend also authenticates itself using its registered method.
5. Have the OIDC library validate the ID token and the expected nonce. Resolve the internal user from the verified issuer and subject.
6. Create a new application session. Keep tokens server-side when the application architecture permits this.

PKCE binds code redemption to the initiating client instance. It does not make an arbitrary redirect URI safe, prevent every form of XSS, or replace the complete OIDC transaction checks. Current OAuth security guidance recommends modern code flows, restricts redirect handling, and addresses refresh-token replay; avoid building new integrations around the implicit or password grant. [RFC 9700](https://www.rfc-editor.org/rfc/rfc9700.html).

## JWT verification is more than decoding

A JWT payload is readable encoding, not encrypted storage. A valid signature only proves that the key signed those bytes. Your API must still verify that the signer and claims are appropriate for this token type and operation.

Configure a maintained verifier with:

- An explicit allowed algorithm set appropriate to the issuer, rather than trusting `alg` from the token.
- Trusted issuer and expected audience; reject absent or mismatched required values.
- Signature verification using keys obtained from a preconfigured trusted issuer/JWKS location.
- Expiration, `nbf` when present, bounded clock tolerance, and any required issuance-time or maximum-age policy.
- Expected token type/profile, required claims, subject format, and permitted scopes.

Do not follow a token-supplied `jku` URL or treat `kid` as a file path. `kid` selects among trusted keys; it does not authorize a new key source. Separate validation rules for ID tokens and access tokens to prevent substitution. [RFC 8725](https://www.rfc-editor.org/rfc/rfc8725.html).

Conceptual adapter contract, deliberately library-neutral:

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class VerifiedPrincipal:
    issuer: str
    subject: str
    audience: tuple[str, ...]
    scopes: frozenset[str]

def authenticate_access_token(raw: str) -> VerifiedPrincipal:
    # Implement with a maintained verifier, never decode-only parsing.
    verified = configured_verifier.verify(raw)
    # The verifier enforces signature, algorithm, issuer, audience,
    # expiration, token profile, and required claims before returning.
    return map_verified_claims(verified)
```

`configured_verifier` and `map_verified_claims` are interfaces to implement; this snippet is not executable authentication middleware. Django core does not turn a decoded JWT into a trusted user automatically. Integrate a maintained OIDC library with Django's session/authentication boundary. After verification, load tenant membership and enforce object-level permissions. A token's `tasks:write` scope does not mean every task belongs to its holder.

## Key rotation, introspection, and revocation

Cache issuer signing keys with a bounded policy. On an unknown key ID, a controlled refresh may be appropriate; cap refresh frequency to resist request amplification. During normal rotation, issuers may overlap old and new keys until valid old tokens expire. An incident can require more immediate rejection.

Self-contained tokens usually trade immediate revocation for offline verification. Options include short access-token lifetimes, a revocation store, token introspection, or a backend session layer. Every option has availability and latency costs. State the actual logout guarantee: revoking a refresh token does not necessarily invalidate already-issued access tokens.

For machine-to-machine access, prefer workload identity or an appropriate confidential-client flow, narrow audiences and scopes, and short lifetimes. User delegation and service identity should remain distinguishable in audit events.

## Refresh rotation and concurrency

With refresh-token rotation, a successful exchange replaces the previous token. The server tracks a token family and detects use of an invalidated predecessor. Store refresh secrets securely; a browser cookie still needs CSRF protection if the browser automatically sends it.

Concurrency creates a subtle failure: two tabs redeem the same old token; the first succeeds and the second resembles replay. Coordinate refresh in the client or backend with a single in-flight operation. Authorization servers should update token state atomically. If a provider supports a narrow retry/grace mechanism, follow its documented semantics; accepting arbitrary predecessor tokens indefinitely defeats replay detection.

A lost refresh response is also ambiguous. Repeating the request may invalidate the family depending on provider behavior. Document a bounded recovery path that can require reauthentication. Never solve the problem by logging refresh tokens or suppressing every reuse alarm.

## Lab: prove your verifier rejects wrong credentials

1. Register a local test client with an exact redirect URI. Use separate development and production registrations.
2. Configure the issuer from trusted application configuration and complete a successful code/PKCE login.
3. Inspect the issued claims locally without publishing the token. Record issuer, audience, expiration, and token type.
4. Test a token for another API audience. Expect authentication rejection before any task query.
5. Test a modified payload, expired token, unknown issuer, missing required expiration, and disallowed algorithm. Each must fail closed.
6. Present an ID token to the API. Expect rejection even if the identity provider signs both token types.
7. Use a provider test facility to rotate keys. Verify valid overlap behavior and bounded unknown-key refreshes.
8. Trigger simultaneous refresh requests and a lost-response simulation. Confirm the client implements the provider's recovery semantics without an infinite loop.

Use test-issued tokens and a local service. Do not weaken production verification flags to construct negative cases.

## Exercises and completion checklist

Design an account-linking flow that requires proof of both identities instead of matching email alone. Compare server sessions, opaque introspected tokens, and JWT access tokens for your latency and revocation requirements.

- [ ] Issuer, audience, algorithm, time, and token profile are enforced.
- [ ] Login transactions bind state, nonce, PKCE, browser, and expiration.
- [ ] Access and ID tokens cannot be substituted.
- [ ] Key rotation and unavailable-key behavior are tested.
- [ ] Refresh races and stolen-token reuse have explicit responses.
- [ ] [Tenant authorization](04-authorization-and-multitenancy.md) is enforced after authentication.
