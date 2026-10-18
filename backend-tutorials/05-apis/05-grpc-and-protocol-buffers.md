# gRPC, Protocol Buffers, and service contracts

[Curriculum](../README.md) · Prerequisites: HTTP, service boundaries, and Python

## Purpose and tradeoffs

gRPC provides typed remote procedure calls, commonly described using Protocol Buffers. Generated clients can reduce hand-written transport code between services. It offers unary, client-streaming, server-streaming, and bidirectional-streaming calls. Browser integration, debugging tools, gateways, and operational support differ from ordinary JSON REST APIs. [gRPC core concepts](https://grpc.io/docs/what-is-grpc/core-concepts/).

A remote call still has network failures, deadlines, partial outcomes, and independent deployments. A generated method that looks like a local function does not make it one. Use gRPC when typed service contracts and streaming help; do not introduce it just to rename an existing HTTP request.

## Define a small contract

Create `tasks.proto` in an isolated protocol lab:

```proto
syntax = "proto3";
package tutorial.tasks.v1;

service TaskReader {
  rpc GetTask(GetTaskRequest) returns (TaskReply);
}

message GetTaskRequest {
  string id = 1;
}

message TaskReply {
  string id = 1;
  string title = 2;
  string status = 3;
  int32 version = 4;
}
```

Tenant identity is intentionally absent from the caller-controlled body. Resolve it from verified credentials and membership, or verify an explicit requested tenant against those credentials. Field numbers are wire identities; never recycle a deleted number for unrelated data. Reserve removed field numbers and names. Adding a field is often compatible, but required business meaning, enum handling, defaults, and client assumptions can still break behavior. [Protocol Buffers evolution](https://protobuf.dev/programming-guides/proto3/#updating).

## Generate Python stubs

In a disposable virtual environment, install the official Python packages, record their resolved versions, and generate code:

```powershell
python -m pip install grpcio grpcio-tools
python -m grpc_tools.protoc -I . --python_out=. --grpc_python_out=. tasks.proto
```

Expected output files are `tasks_pb2.py` and `tasks_pb2_grpc.py`. Keep the compiler and runtime versions compatible. Generated code is a transport adapter, not the domain implementation. The official Python quickstart documents the toolchain. [gRPC Python quickstart](https://grpc.io/docs/languages/python/quickstart/).

Implementation outline, explicitly pseudocode:

```text
GetTask(request, context):
    enforce deadline and authenticated service identity
    resolve authorized end-user/tenant context
    validate UUID
    call the same task-read service used by Django REST views
    return selected fields, or NOT_FOUND / PERMISSION_DENIED by policy
```

A standalone server importing Django models must initialize Django settings and `django.setup()` before model access, manage worker/thread database connections, and use a bounded executor. It is a separate deployment entry point; adding generated files does not make Django's WSGI server speak gRPC.

## Deadlines, cancellation, and retries

Set a deadline for every remote operation. Propagate the remaining budget rather than starting a fresh full timeout at every hop. Observe cancellation and stop unnecessary work where possible, while recognizing that cancellation cannot reverse a committed side effect.

Map transport statuses deliberately: invalid input, missing resource, denied permission, failed precondition, unavailable dependency, and deadline expiry have different meanings. Keep sensitive diagnostics out of status details.

Retry only operations whose side effects are safe under repetition, and use a total attempt/time budget. A timed-out mutation may have committed. Retrying requires idempotency or a reconciliation query, not optimism. Streaming reconnection also needs sequence positions and duplicate handling; a new stream is not automatically a continuation.

## Practical lab and failure tests

1. Compile the schema and inspect the generated request/response classes.
2. Implement a local read-only server using the official Python server pattern and a fake task service.
3. Replace the fake with a tenant-scoped Django service after initializing Django correctly.
4. Add an authentication interceptor before exposing it beyond loopback.
5. Test missing ID, invalid UUID, another tenant's task, short deadline, and unavailable database.
6. Add a response field and regenerate an updated client; verify an older client still performs its documented task.
7. Remove a field only after reserving its number/name and testing old/new combinations.

Expected outcome: failures have meaningful statuses, deadlines bound caller waiting, and wire evolution is tested across versions. The capstone itself remains an HTTP service; this is an explicit integration extension.

## Checklist

- [ ] Remote calls have deadlines and cancellation behavior.
- [ ] Generated transport types do not replace domain validation.
- [ ] Tenant identity is authenticated and authorized.
- [ ] Retry policy accounts for ambiguous mutations.
- [ ] Field numbers and schema evolution are reviewed.

Continue with [service communication](../08-architecture/02-microservices-and-service-communication.md).
