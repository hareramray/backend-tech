# Relational modeling and SQL

Prerequisites: [course map](../README.md), basic programming, and access to PostgreSQL through psql or a database editor. The [task-service capstone](../09-capstone/01-task-service.md) supplies the shared tenants and tasks tables used below. Run experiments in a disposable learning database.

## Objectives

Model entities and relationships, enforce invariants inside the database, write tenant-safe queries, and distinguish application validation from durable data integrity.

## Start with facts and invariants

A relational model represents facts as rows and relationships as keys. A task has one owning tenant. A task can have many comments. Each comment belongs to exactly one task within its own tenant. Write these statements before choosing an ORM or creating tables.

A primary key identifies a row. A candidate key is another combination that could uniquely identify it. A foreign key constrains a relationship. A natural key comes from the domain, while a surrogate key exists for identification. Email addresses, usernames, and external identifiers change more often than developers expect; an internal ID can keep relationships stable while a separate unique constraint enforces the relevant business rule.

Normalization separates facts that change independently. First remove repeating groups; then avoid storing attributes that depend on only part of a composite identity or on another non-key attribute. Store a tenant's name in tenants rather than copying it into every task. Denormalization is reasonable for measured read needs, but every duplicated fact requires an update mechanism and a clear authority.

Choose types deliberately. Use timestamps with time zone for instants, and separate a user's time-zone name when future local schedules matter. Represent currency with integer minor units plus a currency code, or an explicitly scaled decimal; binary floating point is inappropriate for exact monetary totals. A nullable field means an absent or unknown value, which is different from an empty string, zero, or an empty array.

Database constraints protect writes from every client, including scripts and background jobs. A CHECK alone does not reject NULL when its expression evaluates to unknown; use NOT NULL when absence is invalid. Composite foreign keys can enforce relationships within a tenant. [PostgreSQL constraint reference](https://www.postgresql.org/docs/current/ddl-constraints.html)

## Lab: add comments without weakening tenant isolation

1. Initialize the capstone schema. In `python manage.py shell`, create the shared tutorial records below. Django supplies the Python field defaults; equivalent raw INSERT statements must supply version and timestamps explicitly.

~~~python
from tasks.models import Tenant, Task

tenant, _ = Tenant.objects.get_or_create(
    id="11111111-1111-4111-8111-111111111111",
    defaults={"name": "Local demo"},
)
task, _ = Task.objects.get_or_create(
    id="22222222-2222-4222-8222-222222222222",
    defaults={"tenant": tenant, "title": "Prepare launch", "status": "todo"},
)
assert task.tenant_id == tenant.id
~~~

2. Create this extension table once. All statements in this section are executable PostgreSQL SQL.

~~~sql
CREATE TABLE task_comments (
  id uuid PRIMARY KEY,
  tenant_id uuid NOT NULL,
  task_id uuid NOT NULL,
  body text NOT NULL CHECK (length(btrim(body)) BETWEEN 1 AND 4000),
  created_at timestamptz NOT NULL DEFAULT now(),
  FOREIGN KEY (tenant_id, task_id) REFERENCES tasks(tenant_id, id)
);
CREATE INDEX task_comments_task_time_idx
  ON task_comments(tenant_id, task_id, created_at, id);

INSERT INTO task_comments(id, tenant_id, task_id, body)
VALUES (
  '33333333-3333-4333-8333-333333333333',
  '11111111-1111-4111-8111-111111111111',
  '22222222-2222-4222-8222-222222222222',
  'Write the acceptance criteria'
);
~~~

The composite reference is intentional. A foreign key on task_id alone would prove that a task exists, but would not prove the comment's tenant matches that task's tenant. Referential integrity supplements authorization; the API must still derive the tenant from an authenticated principal rather than trusting a request body.

3. Query tasks with comment counts, including tasks that have none.

~~~sql
SELECT t.id, t.title, count(c.id) AS comment_count
FROM tasks AS t
LEFT JOIN task_comments AS c
  ON c.tenant_id = t.tenant_id AND c.task_id = t.id
WHERE t.tenant_id = '11111111-1111-4111-8111-111111111111'
GROUP BY t.id, t.title
ORDER BY t.id;
~~~

COUNT(c.id) counts matched comments; COUNT(*) would count the placeholder row produced by the left join. Putting a comment-only predicate in WHERE can accidentally remove tasks without comments. Put that predicate in the join condition when preserving unmatched tasks is the intention.

4. Use an existence query when only a yes/no answer is needed.

~~~sql
SELECT t.id, t.title
FROM tasks AS t
WHERE t.tenant_id = '11111111-1111-4111-8111-111111111111'
  AND EXISTS (
    SELECT 1 FROM task_comments AS c
    WHERE c.tenant_id = t.tenant_id AND c.task_id = t.id
  );
~~~

5. Parameterize values in application code. Run this Python fragment inside the initialized Django project's `python manage.py shell`. In a request handler, obtain tenant identity from the authenticated principal.

~~~python
from django.db import connection

authenticated_tenant_id = "11111111-1111-4111-8111-111111111111"
with connection.cursor() as cursor:
    cursor.execute(
        "SELECT id, title FROM tasks WHERE tenant_id = %s AND status = %s",
        [authenticated_tenant_id, "todo"],
    )
    rows = cursor.fetchall()
print(rows)
~~~

Django's cursor accepts unquoted `%s` placeholders with a separate parameter collection; do not apply Python string interpolation yourself. [Django raw SQL documentation](https://docs.djangoproject.com/en/5.2/topics/db/sql/) Parameters represent values, not table names or sort directions. Select dynamic identifiers from a small server-owned allowlist; never concatenate arbitrary client input into SQL. Keep query results bounded and project only the columns needed.

## Queries beyond basic CRUD

Joins combine related rows; aggregates reduce groups; window functions annotate rows without collapsing them. Common table expressions organize complex queries, but are not a promise of faster execution. UNION removes duplicates while UNION ALL preserves them. SQL's three-valued logic makes `column = NULL` incorrect; use IS NULL. A NOT IN subquery containing NULL can surprise you; NOT EXISTS often expresses the intended anti-join more clearly.

An ORM can manage mapping, parameterization, and migrations, but it does not remove the need to understand SQL. Inspect generated statements for N+1 queries, unbounded eager loading, ambiguous deletes, missing tenant filters, and implicit transactions. A repository layer should expose domain operations rather than an unrestricted query builder to every route.

## Expected outcomes and failure tests

The count query should include the seeded task with a count of one. Try inserting a blank body: the check must reject it. Try inserting a comment with another tenant ID but the seeded task ID: the composite foreign key must reject it. Execute these deliberately failing statements separately so an aborted transaction does not hide later tests.

Create a second task with no comments and verify it remains in the left-join result. Request another tenant's task through the API and verify that access is denied even though the task exists. Record whether the API deliberately returns 403 or 404; keep the policy consistent.

## Common mistakes

- Encoding relationships as comma-separated IDs or unchecked JSON arrays.
- Using application prechecks as the only uniqueness protection.
- Assuming globally unique IDs replace authorization.
- Storing derived totals without defining how they remain correct.
- Returning database error details directly to clients.
- Choosing cascade deletion without reviewing the domain consequences.

## Exercises and completion checklist

Add labels with a tenant-scoped unique name and a task-label join table. Explain which relationships require composite foreign keys. Write a query returning each tenant's task counts by status, then a window query ranking tasks by creation time within a tenant.

- [ ] Every table has an intentional identity and ownership model.
- [ ] Required fields, uniqueness, and relationships are enforced.
- [ ] Query values are parameterized and result sets bounded.
- [ ] Empty, missing, duplicate, and cross-tenant cases were tested.
- [ ] You can explain every join and its expected cardinality.

Continue with [transactions and concurrency](02-transactions-and-concurrency.md) and [query performance](03-indexing-and-query-performance.md).
