# Networking, DNS, TLS, and HTTP

[Curriculum](../README.md) · Prerequisite: [request lifecycle](01-client-server-and-request-lifecycle.md)

## Goals

Understand which network layer can fail, inspect a raw exchange, and configure an API around real HTTP semantics. This chapter uses local commands; public DNS inspection is read-only.

## Names, addresses, ports, and connections

A hostname identifies a service name; DNS maps names to records such as addresses or aliases. A DNS TTL influences cache freshness, but a deployment cannot assume every client immediately re-resolves a changed record. Existing connections can continue using the previous endpoint.

An IP address identifies a network interface endpoint; a port selects a listening transport endpoint. `127.0.0.1` is IPv4 loopback: reachable from the same host, not a public deployment address. Inside a container, loopback refers to that container's network namespace. Therefore `localhost` in an application container does not identify a separate database container.

TCP provides an ordered byte stream. Application messages still require framing. HTTP/1.1 expresses request boundaries using protocol rules including body lengths and transfer coding. HTTP/2 multiplexes streams over a connection; HTTP/3 uses QUIC. Their exact negotiation and intermediaries matter when debugging, but your resource semantics should not depend on a browser choosing one wire version.

TLS authenticates endpoints with certificates and protects traffic in transit. Check the certificate hostname, chain, validity, and trust root. Turning off certificate verification removes essential authentication. Terminating TLS at a load balancer creates a separate trust decision for the connection from that balancer to your application.

## An HTTP exchange

```http
GET /v1/tasks/?limit=20 HTTP/1.1
Host: localhost:3000
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Cache-Control: no-store
X-Request-Id: example-request-id

{"items":[],"next_cursor":null}
```

The target includes the path and query; headers carry protocol metadata; the body carries a representation. `Content-Type` describes the sent body. `Accept` expresses acceptable response formats. A JSON response does not make a non-JSON request valid.

Protocol properties have precise meanings: safe methods request no state change as their intended purpose; idempotent methods have the same intended effect when repeated. Logging a GET does not violate safety. Two DELETE calls may return different statuses while remaining idempotent. POST and PATCH are not inherently idempotent. [HTTP method semantics](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2).

## Status and caching decisions

| Status | Typical API use |
| --- | --- |
| 200 | Successful operation with a representation |
| 201 | Resource created; usually include its location |
| 202 | Accepted for deferred processing; expose a status resource |
| 204 | Success without a response body |
| 304 | Conditional cache validation; not a normal JSON body response |
| 400 | Malformed request or this API's invalid-input policy |
| 401 | Authentication required/invalid; include the applicable challenge |
| 403 | Recognized caller lacks permission |
| 404 | No visible resource at this target |
| 409 | Conflict with current application state |
| 412 | A supplied precondition failed |
| 413 / 415 | Body too large / unsupported media type |
| 429 | Rate policy exceeded |
| 500 / 503 | Unexpected server failure / temporary unavailability |

Choose and document how 400 versus 422 is used; consistency matters. Avoid returning 200 with an embedded `error` for ordinary REST failures. Distinguish a database outage from a nonexistent row.

`Cache-Control: no-store` asks caches not to store a response; `no-cache` allows storage but requires revalidation before reuse. An ETag identifies a representation version. `If-None-Match` supports conditional reads; `If-Match` supports preconditioned writes. Private authenticated resources need careful cache partitioning and policy. [HTTP caching, RFC 9111](https://www.rfc-editor.org/rfc/rfc9111.html).

## Practical lab

With the [capstone](../09-capstone/01-task-service.md) running, use PowerShell:

```powershell
Resolve-DnsName example.com
Test-NetConnection 127.0.0.1 -Port 3000
curl.exe -i http://127.0.0.1:3000/health/live/
curl.exe -i http://127.0.0.1:3000/route-that-does-not-exist
```

Use `curl.exe` explicitly on Windows to avoid older PowerShell aliases. First inspect the status line, then headers, then the body. A successful TCP connection proves neither correct routing nor database health.

Next send a POST with `Content-Type: text/plain`. The capstone should return 415. Send JSON containing an empty title and expect 400. Compare an application 404 with a connection-refused failure after stopping the server: the latter has no HTTP response at all.

## Timeouts, proxies, and trust

Give each operation a time budget: DNS lookup, connect, TLS, request upload, upstream response, and total operation are different phases. A server's request-receiving timeout is not automatically a deadline for database execution. A client giving up does not guarantee the server stopped doing work.

Forwarded headers can describe the original address and scheme when a trusted proxy rewrites them. Do not globally trust arbitrary `X-Forwarded-For` input. Configure the actual proxy topology, otherwise rate limiting, secure-cookie decisions, and audit records can be spoofed.

Keep request framing unambiguous across intermediaries. Enforce body/header limits and supported transfer behaviors consistently; conflicting interpretations can cause request smuggling. This is a proxy and server configuration concern as well as an application concern.

## Mistakes, exercises, and verification

Common mistakes include hardcoding an IP instead of respecting service discovery; confusing TLS with user authentication; using GET for state changes; retrying a POST blindly; and assuming CORS applies to nonbrowser clients.

Exercise: prepare a troubleshooting matrix for DNS failure, refused connection, TLS failure, 404, 429, and 503. Record which layer emitted each failure and whether retrying could help. Then design caching headers for public documentation versus a private billing response.

- [ ] I can distinguish network failures from HTTP failures.
- [ ] I can explain safety, idempotency, and caching separately.
- [ ] I know which proxy headers the application is allowed to trust.
- [ ] I can inspect a request without printing credentials.

Continue with [REST](../05-apis/01-rest-and-resource-design.md) and [web security](../03-security/05-web-security-and-input-attacks.md).
