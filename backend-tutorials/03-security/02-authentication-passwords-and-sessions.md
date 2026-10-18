# Authentication, passwords, and sessions with Django

## Purpose and prerequisites

Authentication establishes a principal; authorization decides what that principal may do. Login, enrollment, session renewal, password reset, factor replacement, logout, and account recovery are all part of the authentication boundary.

Prerequisites: [security foundations](01-security-foundations-and-threat-modeling.md), HTTPS and cookies, and the [Django task-service capstone](../09-capstone/01-task-service.md). Use Python 3.12+ and Django 5.2.x. The capstone's loopback-only demo tenant is not authentication. Before production, remove demo behavior, require authenticated membership, and enable CSRF protection on cookie-authenticated mutation routes.

Use Django's maintained authentication and session framework rather than constructing a custom password database for this tutorial. Configure the standard auth, contenttypes, and sessions applications and their migrations. Keep `SessionMiddleware`, `CsrfViewMiddleware`, and `AuthenticationMiddleware` in the required order, with session middleware before authentication.

## Password storage and validation

A password is a shared secret with uneven entropy. Store it through `set_password()` or the user manager's `create_user()`, never by assigning plaintext directly to `user.password`. Django's configured hashers handle encoded algorithm parameters and verification. The first configured hasher creates new hashes; retain supported older hashers during migration. Django can upgrade hashes on successful authentication. [Django password management](https://docs.djangoproject.com/en/5.2/topics/auth/passwords/).

Inside `python manage.py shell`, this demonstrates hashing without creating an account:

```python
from django.contrib.auth.hashers import make_password, check_password

encoded = make_password("tutorial-only-passphrase-not-a-real-account")
assert check_password("tutorial-only-passphrase-not-a-real-account", encoded)
assert not check_password("wrong-password", encoded)
assert encoded != "tutorial-only-passphrase-not-a-real-account"
```

Keep Django's supported default PBKDF2 configuration for the baseline lab. Argon2 is an optional choice requiring its supported dependency and deliberate `PASSWORD_HASHERS` configuration; benchmark it before changing production. OWASP currently recommends Argon2id with at least 19 MiB memory, two iterations, and one degree of parallelism, or an appropriate alternative. Do not copy those settings blindly into a library with different units. [OWASP password storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html).

Password validators and password hashers solve different problems. A custom registration or reset API must invoke password validation explicitly; calling `create_user()` alone does not apply every configured validator. Allow password managers and long passphrases, reject known compromised choices where supported, bound input size, and avoid silent truncation.

## Establishing a Django session

Use `AuthenticationForm` and built-in authentication views where they fit. If you need a custom JSON login endpoint, compose Django's authentication functions while preserving CSRF, input validation, generic errors, rate limits, and secure error handling. The essential operation is:

```python
from django.contrib.auth import authenticate, login

def establish_session(request, username, password):
    # A validated, CSRF-protected caller supplies these values.
    user = authenticate(request, username=username, password=password)
    if user is None:
        return False
    login(request, user)
    return True
```

This helper is not a complete endpoint. `authenticate()` delegates to configured backends; review backend policy for inactive users and custom credentials. `login()` updates the session through framework logic. `logout()` clears the current session. Django's groups and model permissions can support authorization, but tenant/object checks remain your application's responsibility. [Django authentication framework](https://docs.djangoproject.com/en/5.2/topics/auth/default/).

For an HTTPS deployment using database sessions:

```python
SESSION_ENGINE = "django.contrib.sessions.backends.db"
SESSION_COOKIE_NAME = "__Host-session"
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = "Lax"
SESSION_COOKIE_PATH = "/"
SESSION_COOKIE_DOMAIN = None
CSRF_COOKIE_SECURE = True
```

These settings assume HTTPS and a properly configured trusted proxy. They are not a reason to weaken production cookies to make an HTTP development server work. Use local HTTPS for browser cookie-security testing or a separate clearly local configuration.

The database session backend stores a session key and encoded session data; it does not implement a custom hashed-session-token scheme. Protect that database accordingly. The signed-cookie backend has different properties: content is signed, not encrypted, and a stolen old cookie is not revoked by server-side logout in the same way as database sessions. [Django sessions](https://docs.djangoproject.com/en/5.2/topics/http/sessions/).

## Expiration, revocation, and recovery

Specify absolute and idle expiration separately. Reading a session does not necessarily extend its expiry, and writing it can change expiration behavior. `SESSION_COOKIE_AGE` is not by itself a complete custom idle-time policy. If strict idle expiry is required, track activity with bounded write frequency and verify the policy on the server. Schedule expired-session cleanup with `python manage.py clearsessions` for backends that require it.

“Log out all devices” needs a deliberate user-to-session index, account session version, or another revocation design; deleting the current cookie is insufficient. Password changes interact with Django's session authentication hash. Use framework password-change helpers and decide whether the current authenticated session should survive a legitimate password change. Test compromised-account recovery separately.

Use Django's built-in password-reset flow or a reviewed identity provider where practical. Configure trusted host/proxy behavior so generated reset URLs cannot be poisoned. Return generic reset-request responses, limit issuance, and avoid exposing reset URLs in logs. Django's reset-token validity changes when relevant user state changes, including the password; this is different from a database row explicitly consumed under a lock.

For custom one-time tokens, generate high-entropy values with Python `secrets`, store digests, bind purpose and account, set expiry, and consume atomically. If strict single-consumption under concurrent reset requests is required, design and test transaction-level coordination rather than assuming a prior validation check is atomic.

## MFA and passkeys

Two passwords are not two factors. TOTP enrollment, code replay prevention, recovery codes, factor replacement, and account recovery need explicit policy. Recovery should not bypass the protections imposed on normal login. Django's core authentication framework does not provide a complete passkey or MFA product; use a maintained integration or identity provider.

Passkeys use public-key credentials bound to a relying party. A WebAuthn verifier must check challenge freshness, origin, relying-party identity, signature, and required user verification. Registration must bind the new credential to the correct authenticated account. Counter behavior varies, especially with synchronized credentials; do not invent a universal counter rule. [WebAuthn specification](https://www.w3.org/TR/webauthn-2/).

## Lab: verify the lifecycle

1. Enable the standard framework components and run `python manage.py migrate` in a disposable capstone copy.
2. Create a local test account using Django's user manager and a validated password. Keep real credentials out of fixtures.
3. Add a CSRF-protected login view and a protected task view requiring both `request.user.is_authenticated` and active tenant membership.
4. Log in over local HTTPS. Verify cookie flags, session rotation, and that the database contains no plaintext password.
5. Copy the session cookie to a second test client, then log out using the first. With database sessions, both clients should lose the copied session's access.
6. Expire the database session and repeat the request. A browser retaining an old cookie must not restore access.
7. Disable the account and test existing sessions under your configured backend. Confirm the result matches policy.
8. Test password reset, password change, lost factors, and session-store failure. Protected operations must fail safely when identity cannot be established.
9. Use `Client(enforce_csrf_checks=True)` for CSRF-specific Django tests; the ordinary test client's relaxed behavior can otherwise hide missing protection.

## Exercises and completion checklist

Design a device/session management screen without exposing raw session keys. Add rate limits before expensive password verification, and test generic failure behavior for nonexistent and incorrect accounts.

- [ ] Passwords use Django's supported hashing and validation APIs.
- [ ] Demo tenant access is removed from production paths.
- [ ] Sessions rotate, expire, revoke, and fail safely.
- [ ] Reset and factor-recovery workflows have explicit policies.
- [ ] Cookie mutation routes enforce CSRF protection.
- [ ] Tenant permissions are checked after authentication.

Continue with [OAuth/OIDC](03-oauth-oidc-and-tokens.md), [authorization](04-authorization-and-multitenancy.md), and [web security](05-web-security-and-input-attacks.md).
