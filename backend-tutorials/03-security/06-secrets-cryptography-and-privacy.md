# Secrets, cryptography, and privacy

## Purpose and prerequisites

A secret grants capability: a database password, signing key, vendor API key, recovery token, or cloud credential can authorize actions. Good cryptography cannot rescue a key placed in a public repository, and encryption does not authorize the application to collect unnecessary personal data.

Prerequisites: [threat modeling](01-security-foundations-and-threat-modeling.md) and [authentication](02-authentication-passwords-and-sessions.md). This is an operational design tutorial with a local rotation simulation. Do not rotate real credentials merely to follow the exercise.

## Secret lifecycle

Track creation, distribution, use, renewal, revocation, and destruction. Every secret needs an owner, permitted identities, purpose, environment, expiration or rotation policy, and incident procedure. Distinguish configuration from secrets: a public API URL is configuration, while a credential embedded in that URL is a secret.

Prefer workload identities and short-lived credentials when supported. Otherwise, retrieve secrets from a controlled secret manager with narrowly scoped access. Environment variables can be practical for local development, but they can leak through process inspection, crash reports, debugging, or copied deployment definitions. They are a transport mechanism, not a complete secret-management system. OWASP emphasizes centralized lifecycle controls, restricted access, rotation, and auditing. [OWASP secrets management](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html).

Never commit real secrets to source control, container layers, example files, frontend bundles, or logs. A `.gitignore` entry does not remove an already tracked secret. If a credential has leaked, revoke or rotate it first, then address repository history and downstream copies according to the incident plan.

## Choose the correct primitive

| Requirement | Appropriate family | Important distinction |
|---|---|---|
| Verify a password | Adaptive password hashing | Human secrets need resistance to guessing |
| Identify random session tokens in storage | Cryptographic digest or keyed digest | High entropy changes the guessing risk |
| Hide recoverable data | Authenticated encryption | Requires secure key and nonce management |
| Authenticate data with a shared key | MAC, such as HMAC | Verifiers can also create authentic messages |
| Authenticate data with public verification | Digital signature | Signing and verification permissions differ |
| Protect transport | TLS with certificate verification | Does not protect plaintext at endpoints |

Use maintained libraries and established protocols. Do not design a cipher, invent password encryption, reuse an AEAD nonce with the same key, or replace certificate validation with “trust every certificate.” Encryption and authentication are distinct requirements; authenticated-encryption constructions address both for the protected message. [OWASP cryptographic storage](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html).

Random identifiers should come from a cryptographically secure random generator when unpredictability matters. Wall-clock timestamps, ordinary pseudo-random helpers, or sequential counters are not suitable session credentials. Constant-time comparison APIs help compare fixed-length authenticators, but surrounding parsing and error behavior still require review.

## Envelope encryption and key boundaries

Envelope encryption uses a data-encryption key for data and a separate key-encryption key to protect the data key. A managed KMS can control access to the wrapping key. Store ciphertext with the metadata needed to decrypt it: algorithm/version, nonce, authentication tag where applicable, wrapped data key, and key reference. The exact representation should come from the chosen library or service.

Bind relevant context as authenticated associated data where supported, such as tenant identity and record purpose. This can detect ciphertext moved between contexts, provided decryption supplies the expected trusted context rather than accepting whatever metadata arrived with the ciphertext.

Disk or database-at-rest encryption protects some storage-loss scenarios. It usually does not stop an application identity from issuing an authorized database read. Application-layer encryption changes that boundary but complicates search, indexing, rotation, debugging, and recovery. Select it based on a documented threat rather than assuming “more encryption” automatically means a better system.

Keys and encrypted backups must have compatible retention. Deleting the only decryption key can make every retained backup useless. Conversely, retaining an old key indefinitely may prevent an intended cryptographic erasure guarantee. Test both restoration and destruction policies.

## Rotation without avoidable downtime

Rotation is a transition, not a single string replacement:

1. Create the replacement credential with the required scope.
2. Deploy readers or verifiers that can accept the overlap if the protocol supports it.
3. Switch writers or clients to the new credential.
4. Observe successful use and identify remaining old consumers.
5. Revoke the old credential after the required overlap, or immediately when incident containment requires it.
6. Verify rejection of the old credential and update inventory records.

Signing keys, database passwords, API keys, and encryption keys have different rotation semantics. Replacing a signing key may invalidate tokens; changing a wrapping key may allow rewrapping without rewriting every plaintext value. Do not apply one universal script to every secret type.

## Privacy engineering

Map each collected field to a purpose. Minimize what you collect, who receives it, and how long it remains. Use synthetic fixtures in development. Redact authorization headers, cookies, reset links, payment details, and unnecessary personal content from logs and traces.

Pseudonymization replaces direct identifiers but can remain linkable; it is not automatically anonymization. Hashing an email address without a secret is often guessable from a list of likely addresses. Separate analytics identifiers from authentication credentials and make access to reidentification mappings explicit.

Deletion is a workflow across primary data, replicas, caches, indexes, exports, queues, vendors, and backups. Backups may expire on a separate schedule; record how a restored backup re-applies deletion records. Jurisdiction-specific legal duties require a separate reviewed policy. This tutorial explains engineering mechanisms rather than asserting universal retention periods or legal compliance.

## Lab: rehearse a fake credential rotation

Use two locally generated dummy values, never real service credentials.

1. Create a local stub verifier accepting credential version `v1`. Keep values in untracked local configuration and put only variable names in example configuration.
2. Add `v2` to the accepted set and change the test client to use it. Record credential version in diagnostics, never the credential value.
3. Run an old client and a new client simultaneously. Both should work only during the deliberate overlap.
4. Remove `v1` from the accepted set. The old client must fail with a controlled authentication error; the new client must continue working.
5. Simulate one forgotten worker using `v1`. Verify monitoring identifies the consumer without printing its secret.
6. Rehearse an emergency rotation with no overlap. Document expected failures and how operators restore legitimate clients.
7. Review one fixture record through a deletion workflow. Identify every derived copy, including an export and a log entry.
8. Restore a disposable backup and apply the deletion ledger. Confirm the deleted fixture does not become active again.

Expected result: rotation can be explained and demonstrated without exposing secret material. A failure includes “successful rotation” while a forgotten consumer still works with the revoked value.

## Exercises and completion checklist

Design a secrets inventory for the task service, including CI, database, object storage, email provider, and observability exporter. For each, specify the smallest useful scope and the consequence of compromise. Compare storage encryption with field-level encryption for sensitive task attachments.

- [ ] Secrets have owners, scopes, and revocation procedures.
- [ ] Production credentials never appear in examples or client bundles.
- [ ] Cryptography comes from maintained libraries and protocols.
- [ ] Rotation includes verification of old-credential rejection.
- [ ] Recovery includes both encrypted data and required keys.
- [ ] Collection, retention, deletion, and third-party copies are documented.

Related: [audit logging](07-abuse-prevention-and-audit.md) and [system design](../08-architecture/05-system-design-and-capacity-planning.md).
