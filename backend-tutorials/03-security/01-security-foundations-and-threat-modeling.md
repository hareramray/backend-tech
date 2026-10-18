# Security foundations and threat modeling

## Purpose and prerequisites

Security is a property of the complete system: application code, identities, infrastructure, dependencies, operators, and recovery procedures. A parameterized query cannot compensate for a public database backup. A strong login mechanism cannot compensate for an endpoint that returns another tenant's records.

Prerequisites: HTTP request/response basics, a simple CRUD service, and the [task-service capstone](../09-capstone/01-task-service.md). This tutorial uses a fictional task application with tenant isolation. Its example risks and priorities are teaching decisions, not an assessment of your real organization.

By the end, produce a threat model, a prioritized control backlog, and a verification plan. These artifacts should change when your system changes.

## Start with assets and guarantees

Confidentiality means unauthorized parties cannot read protected information. Integrity means unauthorized parties cannot alter it. Availability means intended users can obtain the service when needed. Authenticity establishes who or what produced a request; accountability supports investigation. Privacy additionally concerns whether collection and use are appropriate, even when access is technically authorized.

Write business invariants before selecting security products:

- A member of tenant A cannot read, edit, export, or infer private tasks in tenant B.
- A completed task change is attributable to an authenticated actor or explicitly identified service.
- Password recovery cannot bypass account ownership verification.
- A notification provider outage cannot erase accepted task changes.
- Losing one application instance does not destroy the only copy of accepted data.

An asset inventory should include database rows, session credentials, signing keys, backups, audit records, recovery codes, CI credentials, and administrator access. Classify each by sensitivity, owner, retention, and permitted processors. A task title might contain customer information even if the schema calls it ordinary text.

## Map trust boundaries

A trust boundary is where assumptions about identity, integrity, or authority change. Typical boundaries include browser to API, API to database, deployment pipeline to production, and application to an email vendor. Internal networks are still boundaries: a compromised worker must not automatically become a database administrator.

Create a compact flow inventory:

| Flow | Authentication | Data | Boundary question |
|---|---|---|---|
| Browser to API | Session cookie | Task title, task ID | Can another site cause a mutation? |
| API to database | Restricted application role | Tenant-scoped rows | Can a missing filter expose another tenant? |
| Worker to email vendor | Vendor credential | Recipient, message | What information leaves our system? |
| CI to deployment | Workload identity | Application artifact | Can an untrusted contribution deploy? |
| Operator to backup store | Separate recovery role | Entire database | Can restore access bypass normal controls? |

Mark external input even when it arrives indirectly: stored titles, queue messages, imported CSV fields, webhook payloads, and vendor responses remain untrusted. A successful TLS connection establishes transport protection, not business correctness.

OWASP describes threat modeling as an iterative process of modeling a system, identifying threats, deciding responses, and validating them. Use it during design and whenever boundaries change. [OWASP threat modeling guidance](https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html).

## Threat enumeration and prioritization

STRIDE is a useful prompt set: spoofing, tampering, repudiation, information disclosure, denial of service, and elevation of privilege. It is a brainstorming aid, not proof of completeness. Add business abuse: creating thousands of free trials, exhausting an export quota, or exploiting task ownership transfers.

Describe a threat with an actor, entry point, precondition, action, and consequence. “SQL injection” is too vague. “An unauthenticated search caller supplies SQL syntax that becomes executable query text and reads other tenants' task titles” is actionable.

Prioritize using impact, exposure, exploit difficulty, and existing controls. A numerical score can help sorting but does not replace reasoning. Record uncertainty explicitly: an undocumented vendor retention policy is an investigation item, not evidence that retention is safe. Assign an owner and review date for accepted risks.

Controls serve different purposes:

- Preventive: constrained permissions, authentication, parameter binding, schema validation.
- Detective: suspicious-login alerts, audit events, dependency monitoring.
- Corrective: token revocation, credential rotation, data restoration, rollback.

Layer controls around a concrete failure. For tenant isolation, combine request authorization, tenant-scoped queries, database constraints, optional row security, and cross-tenant tests. Five controls that all trust the same spoofable header are not five independent barriers.

## Lab: threat-model the task export feature

This is a design lab; no public target or attack tooling is involved.

1. Define `POST /exports`: an authenticated member requests an export of their tenant's tasks. A worker generates a private object and an authorized download endpoint returns it.
2. Draw or list the API, queue, worker, database, object storage, and browser flows. Include the queue's producer identity and the storage object's tenant metadata.
3. Specify the authorization decision at creation, execution, and download. Decide whether membership revocation cancels a queued export or only prevents download. Either policy requires explicit implementation.
4. Record at least six threats. Use the following as a starting hypothesis table.

| Threat | Proposed control | Verification evidence |
|---|---|---|
| Forged tenant in export request | Tenant resolved from authenticated membership | Tenant B request denied for A-only member |
| Queue message changes object destination | Worker constructs storage key from validated job | Supplied destination never used directly |
| Replayed job creates excessive work | Durable job identity and bounded attempts | Duplicate delivery produces one logical result |
| Public export object | Private bucket and authorized download | Anonymous fetch rejected |
| Ex-member uses old download link | Short expiration plus documented revocation policy | Removal test matches promised behavior |
| Giant export exhausts memory | Pagination, streaming, quota, cancellation | Large fixture stays within memory budget |

5. Write one negative test specification per row. Include exact setup, request, expected status, state changes, and evidence to inspect.
6. Conduct a tabletop incident: a storage credential leaks. List immediate containment, rotation, object-access review, notification ownership, and restoration steps.
7. Review the model after changing the export to a third-party reporting provider. Add data transfer, deletion, subprocessor, and outage assumptions.

Expected result: the model produces implementation tasks and test cases, not merely a diagram. A failure is a threat marked “handled” without a named control, an owner, or a reproducible check.

## Secure delivery and common mistakes

Keep an inventory of runtime dependencies and build inputs. Review dependency updates, protect release credentials, pin reproducible inputs where practical, scan committed secrets, and restrict CI permissions. A scanner finding needs triage based on deployed reachability and impact; a clean scan does not establish secure business logic.

Include security checks in normal review: authorization boundaries, sensitive logging, unsafe deserialization, resource limits, migration privileges, and failure behavior. Separate production administration from ordinary application identities. Document emergency access and record its use.

Common mistakes include treating a penetration test as a permanent certificate, confusing obscured IDs with access control, trusting everything behind a gateway, and postponing recovery design until an incident. Another mistake is installing every possible control without understanding operating cost: a fragile dependency can reduce availability without materially improving security.

## Exercises

1. Threat-model an invitation workflow, including expired invitations and ownership transfer.
2. Add a compromised employee laptop to the export model. Identify which guarantees still hold.
3. Choose one accepted risk and specify what new evidence would change the decision.
4. Review [authorization and multitenancy](04-authorization-and-multitenancy.md) and turn two controls into executable tests.

## Completion checklist

- [ ] Assets, owners, actors, flows, and trust boundaries are documented.
- [ ] Business invariants include abuse and recovery scenarios.
- [ ] High-impact threats map to controls and negative tests.
- [ ] Accepted risks have owners, rationale, and review dates.
- [ ] Third-party and build-pipeline boundaries are included.
- [ ] A change to the system has triggered a model update.

Continue with [authentication and sessions](02-authentication-passwords-and-sessions.md), [input attacks](05-web-security-and-input-attacks.md), and [abuse prevention](07-abuse-prevention-and-audit.md).
