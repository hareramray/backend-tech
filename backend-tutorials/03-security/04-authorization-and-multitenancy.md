# Authorization and multitenancy

## Purpose and prerequisites

Authorization answers whether this authenticated principal may perform this action on this particular resource now. Multitenancy adds a partition boundary: one installation serves multiple organizations, each with its own data and memberships.

Prerequisites: [authentication](02-authentication-passwords-and-sessions.md), SQL transactions, and the [capstone schema](../09-capstone/01-task-service.md). The PostgreSQL examples use the capstone's `tenants` and `tasks` tables. Run role and policy experiments in a disposable database; production policy migrations require compatibility testing.

## Model the decision explicitly

Represent the decision as `allow(subject, action, resource, context)`. Subject attributes might include user ID and active membership. Resource attributes include tenant, owner, state, and sensitivity. Context can include authentication freshness or an approved support-access grant.

RBAC assigns permissions through roles. ABAC evaluates attributes. ReBAC evaluates relationships such as project membership or document sharing. They can coexist: a tenant editor may update a task only if the task belongs to their tenant and is not under a legal hold. Keep policy logic centralized enough to review, but enforce it at every relevant entry point.

Deny by default. Check each request and object, including bulk operations, exports, search, attachments, administrative endpoints, background jobs, and GraphQL resolvers. Unpredictable IDs reduce easy guessing but do not confer permission. [OWASP authorization guidance](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html).

Use an authorization matrix:

| Action | Viewer | Editor | Tenant administrator |
|---|---|---|---|
| Read task in own tenant | Allow | Allow | Allow |
| Update task in own tenant | Deny | Allow | Allow |
| Change membership | Deny | Deny | Allow with constraints |
| Read another tenant | Deny | Deny | Deny |
| Grant platform support access | Deny | Deny | Separate explicit workflow |

Tenant administration is not platform administration. Protect last-administrator removal, invitation acceptance, ownership transfer, and role escalation with business rules and fresh authentication when appropriate.

## Establish trusted tenant context

A tenant ID in a URL or header is a requested scope. Resolve it against verified current membership before constructing a trusted internal context. Never trust a body field named `tenant_id`, a client-supplied role, or a development identity header as a production identity proof.

This optional schema supports the lab:

```sql
CREATE TABLE tenant_memberships (
  tenant_id uuid NOT NULL REFERENCES tenants(id),
  user_id bigint NOT NULL,
  role text NOT NULL CHECK (role IN ('viewer', 'editor', 'admin')),
  revoked_at timestamptz,
  PRIMARY KEY (tenant_id, user_id)
);
```

`user_id` represents Django's default integer user primary key; use the correct type if your custom user differs. This SQL is a design illustration, not an instruction to bypass Django migrations. In the application, define a foreign key to `settings.AUTH_USER_MODEL`; the [forms and admin tutorial](../10-django/02-forms-templates-and-admin.md) provides a concrete membership model. Load active membership and then query the task inside the authorized tenant:

```sql
SELECT id, title, status, version
FROM tasks
WHERE tenant_id = $1 AND id = $2;

UPDATE tasks
SET title = $3, version = version + 1, updated_at = now()
WHERE tenant_id = $1 AND id = $2 AND version = $4
RETURNING id, title, status, version;
```

`$1` comes from server-established context, not an unchecked input. Parameterization prevents SQL injection; tenant predicates prevent cross-tenant access. They address separate threats. Updating zero rows can mean missing, unauthorized, or stale data. Choose consistent response semantics; avoid disclosing another tenant's object existence while still supporting conflict handling for authorized objects.

The SQL above shows PostgreSQL prepared-statement placeholders. Django cursor code uses unquoted `%s` placeholders with a separate parameter list, as in the next example; do not paste `$1` directly into a Django cursor call.

For immediately effective membership revocation, define transaction and locking semantics. A membership check followed by a later write can race with revocation. Sensitive actions may lock or otherwise coordinate the membership record with revocation, or evaluate permission atomically with the mutation. Document whether already-running work may complete.

## Row-level security as an additional boundary

PostgreSQL row-level security, or RLS, can constrain rows seen by the application role. It does not replace identity verification or business authorization. Table owners normally bypass RLS; superusers and `BYPASSRLS` roles bypass it regardless of ordinary policy. Use a restricted non-owner runtime role and a separate migration role. [PostgreSQL row-security documentation](https://www.postgresql.org/docs/current/ddl-rowsecurity.html).

Run this setup as a database administrator in the disposable lab:

```sql
CREATE ROLE task_app NOLOGIN NOSUPERUSER NOBYPASSRLS;
GRANT USAGE ON SCHEMA public TO task_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON tasks TO task_app;

ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;
ALTER TABLE tasks FORCE ROW LEVEL SECURITY;

CREATE POLICY task_tenant_scope ON tasks TO task_app
USING (
  tenant_id = nullif(current_setting('app.tenant_id', true), '')::uuid
)
WITH CHECK (
  tenant_id = nullif(current_setting('app.tenant_id', true), '')::uuid
);
```

`task_app` is a non-login privilege role. Use `SET ROLE task_app` from an authorized administrator session for the SQL experiment, or grant it to a separate restricted runtime login. Do not connect Django using the capstone's elevated setup role and conclude that RLS works: a superuser bypasses these policies. Role creation, grants, and runtime connection configuration are prerequisites to the Django snippet below.

`USING` filters existing rows; `WITH CHECK` constrains new row values. Missing tenant context evaluates to no match. This policy isolates tenants but does not distinguish viewers and editors; application authorization still must.

Set transaction-local context on one checked-out connection:

```python
from django.db import connection, transaction

def list_tenant_tasks(tenant_id):
    # tenant_id must already be authorized for the authenticated user.
    with transaction.atomic():
        with connection.cursor() as cursor:
            cursor.execute(
                "SELECT set_config('app.tenant_id', %s, true)",
                [str(tenant_id)],
            )
            cursor.execute(
                "SELECT id, title FROM tasks WHERE tenant_id = %s",
                [tenant_id],
            )
            return cursor.fetchall()
```

The cursor and `transaction.atomic()` use the same Django database alias. Evaluate ORM querysets inside the transaction; returning a lazy queryset can execute after tenant context has reset. Session-wide tenant settings can leak across pooled requests. Transaction-local settings reset on commit or rollback. The application can set this custom configuration value, so it is not a defense against arbitrary SQL execution or a fully compromised runtime credential. Preserve parameterization and least privilege. Django documents transaction boundaries and rollback behavior in its [transaction guide](https://docs.djangoproject.com/en/5.2/topics/db/transactions/).

Django's default model permissions and groups do not automatically implement tenant isolation or arbitrary object permissions. Use explicit membership checks and `Task.objects.filter(tenant_id=trusted_tenant_id)` on every relevant query. Bind custom membership models to `settings.AUTH_USER_MODEL`, and use migrations rather than mixing unmanaged tables into a real Django application.

RLS applies per table. Add corresponding policies to every tenant-bearing table; review views, privileged functions, partitions, and administrative operations. Foreign-key and uniqueness behavior can expose information through error differences, so return controlled errors. Do not grant the runtime role `TRUNCATE`, ownership, or policy-editing privileges.

## Lab: attack your own tenant boundary

1. Create two fixture tenants and one task in each. Add one user with membership only in tenant A and another with membership only in tenant B.
2. Authenticate as A and request A's task. Expect success.
3. Substitute B's task ID, tenant header, URL tenant, and body tenant separately. Expect denial or the chosen not-found response, with no mutation.
4. Test a bulk update containing one A task and one B task. Define whether the operation is atomic or explicitly reports per-item results; never silently update B.
5. As the restricted database role, begin a transaction, set A's context, and run `SELECT * FROM tasks` without a tenant predicate. RLS should expose only A's rows.
6. Commit and start a new transaction without context. Expect no task rows. Repeat after rollback to expose pooled-connection leaks.
7. Try changing a task's tenant to B under A's context. Expect the policy to reject it.
8. Revoke A's membership, invalidate applicable caches, and test subsequent reads, downloads, jobs, and websocket subscriptions.

## Pitfalls, exercises, and completion checklist

Cache keys need tenant and authorization scope where results differ by permission. Signed download URLs are bearer capabilities with their own lifetime and revocation tradeoffs. Shared search indexes, analytics queries, and support tools frequently bypass otherwise careful application isolation.

Exercise: add project-level sharing without making every tenant member an editor. Then write a table of tests for a user belonging to two tenants with different roles.

- [ ] Every resource path enforces action and object permission.
- [ ] Tenant context derives from verified identity and membership.
- [ ] Runtime database roles cannot bypass or edit policies.
- [ ] Pool reuse cannot retain another tenant's context.
- [ ] Jobs, caches, exports, and revocation follow the same policy.

Related: [input security](05-web-security-and-input-attacks.md) and [modular domain design](../08-architecture/01-modular-monoliths-and-domain-design.md).
