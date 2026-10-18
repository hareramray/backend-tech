# Python, Django, synchronous requests, and asynchronous work

[Curriculum](../README.md) · Prerequisite: basic functions, classes, and exceptions

## Goals and the selected stack

The curriculum uses Python, Django, and PostgreSQL. The hands-on baseline is Python 3.12 or 3.13 and Django 5.2; examples are written for that version rather than relying on whichever release happens to be newest. Django supplies URL routing, views, ORM models, migrations, templates, forms, authentication, middleware, admin, and testing tools. A framework integrates those mechanisms; it does not decide your business invariants or tenant policy.

Create a virtual environment for each project. This isolates Python packages from other projects; it does not isolate network access, processes, or operating-system dependencies. In PowerShell:

```powershell
py -m venv .venv
.\.venv\Scripts\python.exe -m pip install "Django~=5.2.0" "psycopg[binary]>=3.2,<4"
.\.venv\Scripts\python.exe -m django --version
```

Use explicit interpreter paths if activation is unavailable. Record resolved dependency versions for reproducibility. Do not copy this environment into a different operating system; recreate it from dependency records.

## Python objects and runtime validation

Type hints help readers and static checkers, but Python does not automatically validate request JSON against a hinted class. Check types and ranges at the external boundary. In Python, `bool` is a subclass of `int`, so use an exact type check when a boolean is invalid as a numeric version.

```python
def parse_version(value: object) -> int:
    if type(value) is not int or not 1 <= value <= 2_147_483_646:
        raise ValueError("expected_version must be a positive integer")
    return value
```

Avoid mutable default parameters such as `items=[]`; that list is shared across calls. Avoid module-level mutable request state, which leaks across threads and users. Use context managers to close resources and `try/finally` for lifetimes not covered by a context manager.

Exceptions should describe actionable failure categories. Catch the narrow exception you can handle. A broad `except Exception` followed by success hides a broken operation. Log unexpected failures through a central boundary with redaction and a request identifier.

## Django request execution

A server passes a request into the Django application. Middleware surrounds the view, the URL resolver selects a view, the view invokes domain/ORM behavior, and the response passes back through middleware. Keep imports free of database queries and expensive side effects: importing a module occurs in management commands and test setup too.

A project supplies configuration and entry points; an app groups a domain or reusable feature. `config` can be the project package, while `tasks` is an app. A Django app is not necessarily a separately deployed service. The official tutorial documents the project/app distinction and startup commands. [Django first application](https://docs.djangoproject.com/en/5.2/intro/tutorial01/).

## WSGI, ASGI, and async

WSGI is a synchronous server interface. ASGI supports asynchronous application execution and protocols used by long-lived connections. Merely writing `async def` does not make blocking work nonblocking or make a whole Django middleware stack async-compatible.

Django has asynchronous ORM methods for supported operations, but the transaction workflow in Django 5.2 should remain in a synchronous function when used from async code. Use the documented adapters when crossing the sync/async boundary. Avoid calling synchronous ORM operations directly inside an async view. [Django asynchronous support](https://docs.djangoproject.com/en/5.2/topics/async/).

Start with synchronous views for ordinary database CRUD. Use async when concurrent I/O or long-lived connections justify its complexity. A thread or worker process still needs a database-connection budget. Adding workers can overload PostgreSQL even while increasing application capacity.

## Structured concurrency in Python

This complete standalone example shows bounded concurrent asynchronous waiting; it does not query Django:

```python
import asyncio

async def main():
    limit = asyncio.Semaphore(3)

    async def work(item):
        async with limit:
            await asyncio.sleep(0.05)
            return item * 2

    async with asyncio.TaskGroup() as group:
        jobs = [group.create_task(work(i)) for i in range(8)]
    print([job.result() for job in jobs])

asyncio.run(main())
```

Expected output is `[0, 2, 4, 6, 8, 10, 12, 14]`. The semaphore bounds work inside its region, but creating millions of tasks would still consume memory. For large streams, use a bounded producer/consumer queue. `TaskGroup` gives related tasks a shared lifetime and propagates failures; cancellation still does not roll back external side effects. [Python asyncio tasks](https://docs.python.org/3/library/asyncio-task.html).

CPU-heavy Python work may need process-based parallelism, native libraries, or a separate worker service. Interpreter builds and extension behavior affect parallel execution; measure the actual runtime instead of assuming async or threads will accelerate CPU work.

## Database and background work ownership

Keep a transaction short and on the correct connection. Do not send email or call a payment provider while holding row locks unless there is a specific justified requirement. A commit callback can defer a side effect until commit, but it does not durably guarantee delivery after a process crash. Use a transactional outbox when that guarantee matters.

Do not launch untracked threads inside a request for durable jobs. A deployment restart can kill them after the response. Use an explicit worker system with acknowledgement, retry, and deduplication rules.

## Practical lab and verification

1. Run the capstone and identify its settings, root URLs, app URLs, models, and views.
2. Send invalid JSON, a boolean version, and an unknown property; observe deliberate rejection.
3. Run the standalone asyncio example and measure elapsed time.
4. Replace its asynchronous sleep with blocking `time.sleep` in a disposable copy; observe lost concurrency.
5. Explain where a durable email task should be enqueued and what happens if the process dies after commit.

- [ ] I can explain project versus app, view versus model, and WSGI versus ASGI.
- [ ] Type hints are complemented by runtime validation.
- [ ] Background work has an owner and a durable acceptance boundary.
- [ ] Connections, transactions, and concurrent tasks have bounded lifetimes.

Continue with [project structure](04-project-structure-and-configuration.md) and [jobs](../04-async/01-jobs-and-scheduling.md).
