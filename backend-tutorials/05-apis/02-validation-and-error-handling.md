# Validation, serialization, and error handling

[Curriculum](../README.md) · Prerequisites: Python, HTTP, and the [capstone](../09-capstone/01-task-service.md)

## Validate at the correct boundaries

Validation asks whether input has an accepted shape and meaning. Authorization asks whether this caller may perform the operation. Database constraints protect persistent invariants under concurrency. All three are necessary, and none replaces the others.

Use layered checks: transport size/media type, syntax, field shape, cross-field rules, business state, permission, and database invariants. A title must be a string of an accepted length; a status transition may depend on the existing status; a tenant relationship must be authorized; a uniqueness rule must still survive concurrent requests.

For Django, use Forms for HTML form input, serializers for DRF API input, and database constraints for invariants. A normal `model.save()` does not automatically call `full_clean()`. ORM convenience must not be mistaken for automatic validation of every write path. [Django model validation](https://docs.djangoproject.com/en/5.2/ref/models/instances/#validating-objects).

## Define coercion intentionally

Should `"5"` count as integer 5? Should a blank string become null? Should unknown fields be rejected? Those are API contract decisions. Silent coercion can hide client bugs. Python considers booleans integers for subclass checks; use `type(value) is int` when `true` must not become version 1.

Normalize once where the domain requires it. Trimming a task title is reasonable; lowercasing arbitrary prose or modifying a signed webhook body is not. Do not sanitize by removing suspicious SQL characters. Parameterize queries and retain legitimate input according to the domain contract.

## Error categories and response design

| Category | Example | Appropriate behavior |
| --- | --- | --- |
| Input | Unsupported status | Stable client error and field guidance |
| Identity | Expired credential | Authentication failure without credential leakage |
| Permission | Another tenant's object | Consistent forbidden/not-visible policy |
| Conflict | Stale version | Conflict and safe recovery guidance |
| Dependency | Database connection refused | Temporary failure, bounded retries by policy |
| Defect | Unexpected invariant break | Internal error, restricted diagnostics, alert if needed |

The capstone uses a small custom error envelope. For a standardized design, Problem Details defines fields such as `type`, `title`, `status`, `detail`, and `instance` and uses `application/problem+json`. Do not present a custom `{"error": ...}` envelope as that standard. [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html).

Example alternative contract, not a capstone response:

```json
{
  "type": "https://api.example.test/problems/version-conflict",
  "title": "Task version conflict",
  "status": 409,
  "detail": "Read the task again before submitting a new change.",
  "request_id": "a-server-generated-correlation-id"
}
```

The example URL represents a documentation location you would control; it is not an existing service. Keep machine codes stable while allowing human explanations to improve. Never echo secrets, raw SQL, connection URLs, or stack traces into public errors.

## Practical lab: build a rejection matrix

Use the capstone test client or PowerShell to submit:

1. Invalid JSON and duplicate JSON properties.
2. An array where an object is required.
3. An empty title, whitespace-only title, and title beyond 200 characters.
4. An unsupported status and a boolean version.
5. An unexpected `tenant_id` or `is_admin` field.
6. A valid task with an old expected version.
7. A body exceeding the configured size limit.

Record the expected status, public code, and whether any row changed. Invalid and conflicting writes must leave durable state unchanged. Test middleware-level failures separately: malformed hosts, CSRF failure, and unmatched paths may bypass view-level handlers.

## Centralize mapping without hiding causes

Catch domain failures at the HTTP boundary and map them consistently. Keep business functions independent of HTTP when other adapters need them. Distinguish retryable failures from permanent failures. An integrity exception caused by a known unique constraint can become a conflict; an arbitrary database exception must not automatically become “bad input.”

Log a request identifier and restricted diagnostic context. Metrics should use bounded error categories, not user-provided strings as labels. In asynchronous jobs, record the failed attempt and retry decision; returning an HTTP error is no longer the right mechanism.

## Exercises and checklist

Add JSON error handlers for unmatched API paths and CSRF failures without altering ordinary HTML admin behavior. Test with real CSRF checks enabled. Extend the title rules and update the API contract, serializer/form validation, and database constraint where applicable.

- [ ] Untrusted values become domain inputs only after validation.
- [ ] Unknown fields and null/omission behavior are documented.
- [ ] Public errors are stable and do not expose secrets.
- [ ] Failure responses agree with transaction outcomes.
- [ ] Tests cover failures outside the view function.

Continue with [DRF](../10-django/03-django-rest-framework.md) and [web security](../03-security/05-web-security-and-input-attacks.md).
