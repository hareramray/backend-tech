# OpenAPI, API documentation, and compatible evolution

[Curriculum](../README.md) · Prerequisites: REST, validation, and testing

## What a contract needs

An API contract describes paths, methods, request/response schemas, authentication, error behavior, limits, examples, and compatibility rules. Documentation becomes useful when it matches running behavior. Generated schemas can expose structure, but usually need explicit explanations of permission, retries, idempotency, concurrency, and pagination.

OpenAPI provides a machine-readable description for HTTP APIs. OpenAPI 3.1 uses a JSON Schema dialect for schema objects. Choose an explicit specification version supported by your tooling rather than assuming every generator supports every feature. [OpenAPI 3.1.1](https://spec.openapis.org/oas/v3.1.1.html).

## A complete small operation document

Save this as `openapi.yaml` in a disposable capstone copy. It documents only readiness, not the whole service:

```yaml
openapi: 3.1.1
info:
  title: Task Service readiness example
  version: 1.0.0
servers:
  - url: http://127.0.0.1:3000
paths:
  /health/ready/:
    get:
      operationId: readReadiness
      summary: Check local database reachability
      responses:
        '200':
          description: Application can query the database
          content:
            application/json:
              schema:
                type: object
                required: [status]
                properties:
                  status:
                    type: string
                    const: ready
        '503':
          description: Database is unavailable
          content:
            application/json:
              schema:
                type: object
                required: [error, request_id]
                properties:
                  error:
                    type: object
                    required: [code]
                    properties:
                      code:
                        type: string
                        const: database_unavailable
                  request_id:
                    type: string
                    format: uuid
```

For an authenticated endpoint, define the actual security scheme and operation requirements. Do not label an unauthenticated local demo as bearer-secured in its contract. Security declarations document behavior; they do not install authentication middleware.

## Extend the contract systematically

1. Add task creation with the exact accepted fields and maximum title length.
2. Add list responses including `items`, nullable `next_cursor`, default/max limits, and cursor expiration.
3. Add item retrieval and PATCH with required `expected_version` and at least one change field.
4. Document error codes for 400, 404, 409, 413, 415, 405, and 503 where applicable.
5. Include Location, Allow, and X-Request-Id response headers.
6. Explain tenant derivation and the local-demo limitation.
7. Validate the document with a compatible parser and compare actual responses against it.

Examples must use fake values, never real tokens or customer data. A valid example should satisfy its schema. Multiple documents should not independently maintain conflicting definitions of the same title length or status enum.

## Compatibility is behavioral

Adding an optional response property is usually safer than removing a field, but clients may still reject unknown fields. Adding an enum value can break exhaustive clients. Tightening validation, changing sort order, shortening retention, changing pagination semantics, or making an operation slower can be breaking even without a schema change.

Versioning can use path versions, headers, or media types. Choose a strategy based on clients and operational cost. A `/v2/` path does not solve data migration, dual-client support, or deprecation communication. Keep old and new code compatible with the database during rolling deployments.

## Contract tests and schema generation

For Django REST Framework, choose a maintained schema integration compatible with your DRF/OpenAPI versions, record it in dependencies, and annotate business rules that introspection cannot infer. Check the generated output into review or publish it from a reproducible build.

Provider tests verify actual responses against the provider contract. Consumer-driven contracts capture what specific consumers rely on. Neither replaces tests for authorization, concurrency, or provider outages. Test examples and SDKs against a local/staging service before treating them as working integration documentation.

## Practical failure lab

Deliberately change the implementation's error field name while leaving the contract unchanged. The contract check should fail. Restore it, then add a new optional field and verify an old client still works. Finally simulate a rollout with old and new application versions reading the same schema.

Record the deprecation owner, supported period, usage signal, communication channel, and removal criteria for one old field. Avoid removing it merely because no one reported a problem.

## Checklist

- [ ] Schemas, examples, limits, and actual responses agree.
- [ ] Authentication documentation matches enforced behavior.
- [ ] Errors and retries are part of the contract.
- [ ] Compatibility review includes behavior and data, not just JSON shape.
- [ ] Deprecation has observable usage and an owner.

Continue with [integration/contract tests](../06-quality/02-integration-contract-and-end-to-end-tests.md) and [release strategies](../07-operations/03-ci-cd-and-release-strategies.md).
