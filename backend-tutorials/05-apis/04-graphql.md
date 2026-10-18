# GraphQL schemas, resolvers, and query cost

[Curriculum](../README.md) · Prerequisites: API design, authorization, and SQL query planning

## When GraphQL fits

GraphQL lets a client select fields from a typed schema. It can help when multiple clients need different views over connected domain data. It adds schema governance, resolver execution, cost controls, and caching decisions. It does not replace the database or automatically make arbitrary client queries cheap.

A schema describes queries, mutations, object types, input types, enums, and nullability. Resolvers obtain field values from services or storage. The transport and authentication still need explicit configuration. The GraphQL project documents schema, execution, validation, and response behavior. [GraphQL learning guide](https://graphql.org/learn/).

## Design a task schema

This is a schema-design lab, not an endpoint installed by the capstone:

```graphql
enum TaskStatus { TODO DOING DONE }
type Task {
  id: ID!
  title: String!
  status: TaskStatus!
  version: Int!
}
type TaskPage {
  nodes: [Task!]!
  nextCursor: String
}
type Query {
  task(id: ID!): Task
  tasks(first: Int = 20, after: String): TaskPage!
}
input UpdateTaskInput {
  id: ID!
  status: TaskStatus!
  expectedVersion: Int!
}
type Mutation {
  updateTask(input: UpdateTaskInput!): Task!
}
```

Use explicit input types so clients cannot write fields intended only for output. Schema-level nonnull does not express every business rule; the resolver must enforce limits, transitions, and permissions. A nullable task can represent an absent/not-visible record under a documented policy.

## Resolver boundaries

Pseudocode for a Django-backed resolver:

```text
resolve task(id, context):
    require authenticated principal
    obtain tenant from verified membership context
    validate id
    query Task WHERE tenant_id = context.tenant AND id = supplied_id
    apply object/field policy
    return allowed representation or null
```

This uses the same application service as REST. Do not maintain a second version of task business rules inside resolvers. Authorization must cover nested fields and alternate lookup paths, not just the top-level query.

## N+1 queries and batching

If a page contains 50 tasks and each resolver fetches an assignee separately, one request can cause dozens of database queries. Use Django `select_related` for suitable single-valued relations, `prefetch_related` for collections, or request-scoped batching. A batch loader must preserve input/output correspondence and tenant policy.

Never share a user-specific resolver cache globally without permission-aware keys and invalidation. A request-scoped cache can deduplicate repeated reads during one query while preventing cross-user reuse.

## Cost and evolution

Bound page sizes, aliases, depth, total selected complexity, execution time, and request body size. Depth alone does not bound a wide query with many aliases. Persisted/allowlisted operations can make client behavior more predictable, but still need authentication and cost limits.

Nullable fields and partial errors require deliberate client behavior. Making an existing field nonnull or removing an enum variant can break clients. Deprecate fields, measure usage, and plan removal. Adding an enum value can also break clients with exhaustive switches even if a schema check calls it additive.

## Practical lab and expected outcomes

1. Choose and install a Django-compatible GraphQL integration in a disposable capstone branch, using its official version-specific setup guide.
2. Register the schema above and implement only the task read and bounded list resolvers first.
3. Reuse authenticated tenant context from the security tutorial; do not expose demo tenant selection publicly.
4. Query a task with only its ID/title, then query another tenant's ID and verify it is not visible.
5. Add a relationship in a lab model, measure query count, and remove an N+1 pattern.
6. Submit an oversized page, deeply nested query, and many aliases; verify the configured limits.
7. Implement the versioned mutation through the existing service and test a stale update.

The integration package is an explicit prerequisite; these schema fragments alone do not create a server. Record the chosen package and resolved version so the lab is reproducible.

## Checklist

- [ ] Schema fields express a public domain contract, not every database column.
- [ ] Every resolver path enforces tenant and field permission.
- [ ] Query cost and response size are bounded.
- [ ] Batching avoids N+1 without cross-user cache leakage.
- [ ] Clients can handle partial errors and schema evolution.

Next: compare [gRPC](05-grpc-and-protocol-buffers.md) and [REST](01-rest-and-resource-design.md) for the same operation.
