# REST, resources, routes, and HTTP behavior

[Curriculum](../README.md) · Prerequisite: [HTTP](../01-foundations/02-networking-dns-tls-and-http.md)

## Goals and resource modeling

Design an API around stable domain concepts and predictable operations. A resource is an identifiable concept; its JSON representation is one view of that concept. A database table and an API resource do not need a one-to-one relationship. Exposing your schema directly makes internal changes become public breaking changes.

For a task service, use a collection `/v1/tasks/` and an item `/v1/tasks/{id}/`. The capstone implements creation, listing, retrieval, and partial updates. A complete product might add comments, assignments, attachments, and activity resources; each requires ownership rules.

| Operation | Example | Contract decision |
| --- | --- | --- |
| List | GET collection | Bounded page, deterministic order |
| Create | POST collection | 201 plus Location and representation |
| Read | GET item | 200 or no visible resource |
| Replace | PUT item | Define complete replacement and creation policy |
| Partial update | PATCH item | Define omitted/null behavior and concurrency |
| Delete | DELETE item | Define retention, references, and retry behavior |

HTTP's standardized methods carry safety and idempotency semantics. REST also involves architectural constraints such as a uniform interface and stateless interactions; a JSON CRUD API is not automatically a complete implementation of every REST constraint. [HTTP methods](https://www.rfc-editor.org/rfc/rfc9110.html#section-9).

## Business operations need a model

Some actions do not fit generic field updates. “Approve expense” may require policy checks, an immutable decision record, and notifications. Model it as a transition with an explicit contract, for example creating an approval resource, rather than allowing arbitrary `status='approved'` writes.

Avoid deeply nested URLs unless they communicate a stable ownership relationship. `/tenants/{tenant}/projects/{project}/tasks/{task}` repeats identifiers whose consistency must be checked. A shorter task URL still needs tenant authorization. Path complexity does not enforce security.

## Requests, responses, and concurrency

Expose only allowed response fields. Internal notes, password hashes, integration credentials, and soft-deletion metadata should not leak just because a serializer can see them. Define timestamps, enums, monetary values, and identifiers independently of Python's internal types.

For updates, choose last-write-wins, expected-version checks, ETag preconditions, or a domain merge policy. The capstone requires `expected_version`; a stale update returns 409. If using `If-Match` instead, a failed precondition maps naturally to 412. Do not silently retry an edit against a new version without deciding whether the user's intent is still valid.

Bulk operations need a maximum batch size, authorization per item, atomicity rules, and partial-failure representation. A batch endpoint that checks only the first item's tenant is unsafe. A large transaction can hold locks far longer than a series of bounded chunks.

## Practical lab

1. Run the capstone and create a task through POST.
2. Inspect Location and retrieve that exact resource.
3. Update using version 1 and then repeat the same version-1 request.
4. Verify the second update fails and the record stays at version 2.
5. Submit a client-chosen tenant or ID field; the create validator should reject it.
6. Design a delete contract on paper before implementing it: hard deletion, retention, foreign keys, and response after a repeated delete.

Write a contract table containing successful and failed outcomes for each method. Include wrong media type, unknown route, unknown object, invalid enum, unauthenticated caller, forbidden caller, conflict, and unavailable dependency. The unauthenticated capstone does not implement the identity rows; complete them in the security extension.

## Common design mistakes

Using GET to trigger a destructive action lets crawlers, prefetchers, and retries accidentally invoke it. Returning 200 for every outcome prevents generic clients and monitoring from understanding failures. Treating POST as safe to retry without deduplication can create duplicate side effects. Exposing stack traces helps attackers and couples clients to implementation details.

Consistency is valuable, but do not force every domain action into a universal repository API. A well-named application service can preserve business meaning while multiple HTTP, worker, and admin adapters call it.

## Exercises and completion checklist

Design a comment resource and an archive-task operation. For each, state authorization, transaction boundary, idempotency, concurrency policy, and observable result after a timeout. Then implement one using the existing service pattern and add an integration test.

- [ ] Resource identity is stable and independent of presentation.
- [ ] Methods and statuses agree with documented behavior.
- [ ] Every list and bulk operation is bounded.
- [ ] Updates have an explicit lost-update policy.
- [ ] Client-visible fields are intentionally selected.

Next: [validation](02-validation-and-error-handling.md), [pagination](03-pagination-filtering-and-sorting.md), and [contracts](08-openapi-contracts-and-evolution.md).
