# Web security and input attacks

## Purpose and prerequisites

Every boundary accepts data that may be malformed, misleading, oversized, or intentionally hostile. Validation checks whether data fits your contract. Parameterization prevents data becoming query syntax. Output encoding prevents data becoming browser markup. Authorization determines whether a valid operation is permitted. None substitutes for the others.

Prerequisites: [authorization](04-authorization-and-multitenancy.md), HTTP, JSON, cookies, and SQL. Use only the local [task-service capstone](../09-capstone/01-task-service.md) and disposable fixtures for experiments. Examples describe defenses; they do not form a production security middleware package.

## Define a strict input contract

Validate types, lengths, formats, ranges, object depth, collection counts, and permitted fields. Reject ambiguous representations: a single-value parameter sent twice, conflicting content lengths, or a JSON object where a string was expected. Coordinate parsing rules with the proxy and framework.

For task creation, accept only `title` and a permitted initial state. Set `tenant_id`, ownership, timestamps, and privileged flags on the server. Avoid copying an entire request body into an ORM model. This is the mass-assignment boundary.

Normalize only according to domain rules. Trimming a task title may be appropriate; silently altering a password is not. Validate Unicode and byte limits where the database or downstream service imposes byte limits. Bound request bodies before loading them into memory, and set header, body-read, and total-request timeouts.

## Injection families and specific defenses

| Boundary | Failure mechanism | Defense |
|---|---|---|
| SQL | Data becomes query syntax | Bound values; allowlist dynamic identifiers |
| Shell | Input becomes command syntax | Avoid shells; fixed executable and argument arrays |
| HTML | Stored text becomes executable markup | Context-aware escaping and safe rendering |
| NoSQL query | User controls operators or query structure | Typed query construction and operator allowlists |
| Template engine | Untrusted template executes expressions | Fixed trusted templates; data-only input |
| XML/deserialization | Parser resolves entities or creates dangerous objects | Safe parsers, disabled external entities, constrained formats |
| File paths | Input escapes intended directory | Generated names and validated storage identifiers |
| Object merge | Special keys alter object behavior | Schema validation and safe data structures |

Values can be bound in SQL; column names and sort directions generally require explicit server-side choices:

```python
allowed_sort = {"newest": "-created_at", "oldest": "created_at"}
order = allowed_sort.get(requested_sort)
if order is None:
    raise ValueError("Unsupported sort")
rows = list(
    Task.objects.filter(tenant_id=tenant_id)
    .order_by(order, "id")
    .values("id", "title")[:page_size]
)
```

The sort expression comes only from the fixed mapping. Import the capstone's `Task` model; `tenant_id`, `requested_sort`, validated `page_size`, and a surrounding error handler are prerequisites. Django's ORM binds ordinary values, but raw SQL and unsafe expression construction can still reintroduce injection. Do not interpolate user input into raw query text.

XSS can originate in backend-stored data. Treat task titles as text in the frontend; encode for the actual HTML, attribute, URL, or JavaScript context. A Content Security Policy reduces some consequences but cannot make unsafe rendering correct. Avoid returning stack traces, SQL text, credentials, or raw vendor errors to clients.

## Cookies, CSRF, and CORS

CSRF occurs when a browser sends ambient credentials with an attacker-induced request. Cookie-authenticated mutations need an explicit defense: framework CSRF protection, a session-bound synchronizer token, or an appropriately signed double-submit design, supported by strict Origin checks where suitable. Do not mutate state with GET. Protect login and sensitive logout flows according to their risks as well.

SameSite cookies add protection, but “same-site” differs from “same-origin”; sibling subdomains can matter. `SameSite=None` is needed for some cross-site uses and requires `Secure` in modern browsers. A bearer token deliberately added to an Authorization header is not automatically attached by a cross-site form, but a refresh endpoint using cookies may still need CSRF protection. XSS can undermine many CSRF defenses. [OWASP CSRF guidance](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html).

CORS is a browser read-access policy, not API authorization. A server-to-server client ignores it. Some cross-origin “simple” requests reach the server without preflight even if the browser cannot read the response. For credentialed browser access, allow only exact trusted origins, handle preflights deliberately, and return `Vary: Origin` when responses vary by origin. Never reflect any submitted origin with credentials enabled.

In Django 5.2, retain `CsrfViewMiddleware`, render `{% csrf_token %}` in forms, and send the supported CSRF header for same-origin AJAX mutations. Configure `CSRF_TRUSTED_ORIGINS` narrowly; it is separate from CORS. Remove the capstone's loopback-demo exemption before adding real cookie-authenticated access. Django validates request origin/referer where applicable and rotates the CSRF secret on login. [Django CSRF reference](https://docs.djangoproject.com/en/5.2/ref/csrf/).

## SSRF, uploads, and webhooks

Server-side URL fetching creates an SSRF boundary. Prefer fixed destinations or strict allowlists. Validate schemes, hostnames, ports, resolved addresses, and redirects. Reject private, loopback, link-local, and cloud metadata destinations when they are not required. DNS can change between validation and connection; use a networking layer that enforces destination restrictions at connection time and network egress controls. Put time and byte limits on responses.

Uploads need size and count limits, generated storage names, private storage, content validation appropriate to the format, and authorization for retrieval. MIME types and extensions supplied by clients are claims. Store untrusted HTML or SVG outside the application's privileged origin or serve it safely as a download. Image processing and archive extraction need decompression, path, and resource limits.

Webhook verification should use the provider's documented signature scheme over the exact raw bytes, a bounded timestamp window where supported, and replay/deduplication controls. Parse the payload only after obtaining the bytes needed for verification. Re-fetch authoritative provider state when the business workflow requires it; a valid signature does not make every event actionable forever.

## Lab: build a negative-input matrix

1. Define valid task creation: a bounded nonempty string title and no client-selected tenant or privilege fields.
2. Send a number, null, array, deeply nested object, oversized title, and unexpected `tenant_id`. Expect a controlled client error and no database write.
3. Store a title containing quotes and angle brackets. Confirm it remains literal text, can be read back, and does not alter SQL execution or execute in the UI.
4. Try an unknown sort option. Expect rejection before query execution; do not fall back to interpolating it.
5. With cookie authentication enabled, submit a mutation without a valid CSRF token from a different origin. Expect rejection and no state change. Repeat using a same-site sibling origin if your deployment has one.
6. Test permitted and forbidden CORS origins separately. Confirm that application authorization still rejects an unauthenticated non-browser HTTP client.
7. In a stub URL-fetch feature, reject loopback and metadata destinations without actually contacting sensitive services. Test a redirect from an allowed host to a forbidden host.
8. Upload an oversized fixture and a file with mismatched claimed type. Confirm bounded resource usage and safe error handling.

Expected results include both rejection status and absence of side effects. A response of 400 is not sufficient if the database was modified first.

## Pitfalls, exercises, and completion checklist

Avoid global “sanitize everything” functions: encoding depends on the destination, while validation depends on the field. A web application firewall cannot know all business invariants. A regex-only URL filter misses parser and DNS edge cases. Resource limits must also apply after decompression and during downstream processing.

Exercise: design a CSV export that does not turn untrusted cells into spreadsheet formulas when opened in common spreadsheet software. Document the format and compatibility tradeoffs rather than silently changing stored values.

- [ ] Each input has type, size, depth, and field constraints.
- [ ] Queries and commands keep code separate from data.
- [ ] Stored text is safely rendered in its output context.
- [ ] Cookie mutations have tested CSRF protection.
- [ ] CORS never substitutes for authentication or authorization.
- [ ] Uploads, URL fetches, and webhooks have their own boundaries.

Continue with [secrets and privacy](06-secrets-cryptography-and-privacy.md) and [abuse prevention](07-abuse-prevention-and-audit.md).
