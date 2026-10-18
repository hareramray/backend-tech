# Project structure, configuration, and dependency boundaries

[Curriculum](../README.md) · Prerequisite: [Python, Django, and async](03-python-django-and-async.md)

## Goals

Build a structure that makes behavior easy to locate, configuration easy to validate, and infrastructure easy to replace in tests. Start with the [capstone](../09-capstone/01-task-service.md); its small file count is deliberate, and this tutorial explains how to grow it.

## Organize around responsibilities

For a small application, separate startup, HTTP handling, validation, and data access. As the application grows, group related behavior by domain module rather than building enormous global controller and service folders.

```text
manage.py
config/
  settings.py             # configuration and installed apps
  urls.py                 # root routing
  wsgi.py                 # synchronous deployment entry
  asgi.py                 # asynchronous deployment entry
tasks/
  urls.py                 # task routes
  views.py                # HTTP translation
  services.py             # use cases and transaction boundaries
  models.py               # ORM models and database constraints
  validation.py           # external input parsing
  migrations/             # reviewed schema history
  tests/                  # behavior and integration tests
```

A directory is not a boundary unless dependencies respect it. Avoid a service importing another module's internal tables or private implementation classes. Export a small public interface. For a tiny CRUD route, a service layer may add no useful decision; add boundaries when they clarify rules, transactions, or testing.

Dependency injection means supplying a dependency instead of creating it invisibly inside business code. It need not require a framework:

```python
# Design fragment: caller supplies an implementation of the persistence function.
def complete_task(tenant_id, task_id, expected_version, *, mark_done):
    updated = mark_done(tenant_id, task_id, expected_version)
    return "completed" if updated else "conflict-or-missing"
```

The caller must still establish authorization. A small interface makes a fake useful for focused policy tests, while database integration tests verify actual SQL behavior. An interface alone does not make an implementation correct.

## Configuration is external input

Environment variables arrive as strings or are absent. In Python, `bool('false')` is true. Parse exact accepted forms and validate ranges. Fail startup before accepting requests if required configuration is invalid.

```python
def parse_port(raw: str | None) -> int:
    text = "3000" if raw is None else raw
    if not text.isascii() or not text.isdigit():
        raise ValueError("PORT must contain ASCII digits")
    port = int(text)
    if not 1 <= port <= 65535:
        raise ValueError("PORT must be between 1 and 65535")
    return port
```

Document configuration names, purpose, allowed forms, default, sensitivity, and whether changes require restart. Keep `.env.example` with placeholders; ignore `.env` and real credentials in version control. Production secrets belong in the deployment's secret delivery mechanism, with restricted access and rotation ownership.

Configuration and feature flags differ. Configuration describes infrastructure and operating parameters. Feature flags select behavior, possibly per user or tenant. Flags need an owner, expiration plan, fallback value, and telemetry; otherwise old branches become permanent complexity. Avoid downloading a remote flag synchronously on every request without caching and failure policy.

## Startup and shutdown order

1. Parse configuration and initialize logging redaction.
2. Create dependency clients with connection and operation limits.
3. Check essential startup assumptions without making destructive changes.
4. Build the application using explicit dependencies.
5. Start listening and expose meaningful readiness.
6. On shutdown, stop accepting work and mark the process unready.
7. Drain requests/jobs, close clients, and exit before a hard deadline.

Schema migrations should be a controlled release step, not an uncontrolled race among every replica at startup. See [migrations](../02-data/04-migrations-and-data-lifecycle.md).

## Practical refactoring lab

1. Run the capstone and save its current endpoint behavior as a checklist.
2. Move configuration parsing into a settings helper without changing accepted values.
3. Move pure input parsers into their own module; the capstone already demonstrates this separation.
4. Extract a repository only after writing down its tenant and concurrency requirements.
5. Inject the repository into a task service that implements one meaningful rule, such as forbidding a status transition unless the current version matches.
6. Keep the HTTP layer responsible for mapping service outcomes to statuses.
7. Run the same create/read/update/conflict scenarios before and after the refactor.

Expected outcome: behavior remains stable, dependencies become visible, and policy can be tested without starting HTTP. Do not call a large file split a successful architecture change if hidden global state remains.

## Failure cases and tradeoffs

A circular import can expose partially initialized values. A singleton with mutable test state can make tests order-dependent. A generic repository that accepts arbitrary table names can erase domain restrictions. Excessive abstraction can make simple SQL impossible to understand or tune.

Prefer local clarity over speculative flexibility. Extract duplicated behavior when its meaning is actually shared; two similar functions in different domains may evolve differently. Inject clocks and random-ID generators when determinism helps, but do not mock every standard library call.

## Exercises and completion checklist

Test missing database configuration, invalid `PORT`, and an unknown feature flag value. The process should fail clearly without exposing credentials. Write an architecture decision record for one boundary and explain its cost.

- [ ] Startup validates required configuration before listening.
- [ ] Secrets are absent from source and ordinary logs.
- [ ] Transaction ownership is explicit.
- [ ] Module dependencies do not form accidental cycles.
- [ ] A new teammate can locate a use case from its route.

Django documents settings loading and environment-specific configuration. [Django settings](https://docs.djangoproject.com/en/5.2/topics/settings/). Continue with [domain design](../08-architecture/01-modular-monoliths-and-domain-design.md).
