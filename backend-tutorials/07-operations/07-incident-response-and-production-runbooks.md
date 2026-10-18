# Incident response and production runbooks

[Tutorial index](../README.md) · Related: [observability](05-observability-and-slos.md) and [recovery](06-reliability-backups-and-disaster-recovery.md)

## Outcome and prerequisites

Respond to service failures with a clear command structure, evidence-driven decisions, and repeatable recovery steps. You need the deployment architecture, monitoring signals, and documented service ownership. All runnable commands here inspect a local learning environment. Production actions require the organization's established authority and environment selection; this tutorial does not send messages, page people, or change deployed systems.

An incident is an event requiring coordinated response because normal service objectives or critical business properties are threatened. Examples include widespread task-write failures, cross-tenant data exposure, severe latency, corrupted data, and loss of the ability to deploy an urgent fix. Severity reflects impact and urgency, not how dramatic a graph looks.

## Establish roles and a shared record

An incident commander coordinates priorities and decisions. An operations lead investigates and executes technical mitigations. A communications lead maintains stakeholder updates. A scribe records evidence and actions. Small teams may combine roles, but everyone should know who currently owns decisions and who is making changes.

Declare early when impact is uncertain but potentially serious. Name the incident, start a timeline with a consistent timezone, identify affected user journeys and regions, and record current hypotheses separately from verified facts. Assign explicit owners for investigations so several people do not unknowingly run conflicting changes. Google's incident-response guidance emphasizes coordination and clear responsibilities. [Google SRE incident response](https://sre.google/workbook/incident-response/)

Keep a durable record of command target, timestamp, operator, intent, and outcome. Avoid pasting secrets or raw customer data into incident channels. A useful update says what users experience, what is known, what is being attempted, and when the next update is expected. Do not promise a recovery time without evidence.

## Walkthrough: investigate local database-related failures

1. Establish scope. Is `/health/live/` successful while `/health/ready/` fails? Do all task operations fail or only writes? Did the problem begin with a deployment, configuration change, migration, traffic increase, or external event? Correlation narrows investigation but does not prove causation.

2. Inspect the local environment from the capstone's Compose directory:

```powershell
docker compose ps
docker compose logs --tail 100 db
curl.exe --silent --show-error --include http://127.0.0.1:3000/health/live/
curl.exe --silent --show-error --include http://127.0.0.1:3000/health/ready/
```

Expect useful differences between process availability and dependency readiness. Review log output before sharing it. The Django server may run on the host and therefore will not appear as a Compose service in the basic capstone.

3. If PostgreSQL is reachable, inspect activity using this read-only SQL through an authorized database client. It omits query text to reduce accidental exposure of application values.

```sql
SELECT pid, application_name, state, wait_event_type, wait_event,
       now() - query_start AS query_age,
       now() - xact_start AS transaction_age
FROM pg_stat_activity
WHERE datname = current_database()
ORDER BY xact_start NULLS LAST;
```

Look for many waiting connections, long-lived transactions, or a changed workload. Visibility depends on database privileges. A long transaction is evidence to investigate, not automatic permission to terminate it. PostgreSQL exposes activity and wait information through its monitoring views. [PostgreSQL monitoring statistics](https://www.postgresql.org/docs/17/monitoring-stats.html)

4. Form competing hypotheses: a connection limit, bad credentials, a locked table, missing schema, exhausted storage, or a network failure. Choose the cheapest read-only observation that distinguishes them. For example, successful direct SQL with failed application connections suggests application configuration, pool behavior, or network-path differences rather than complete database loss.

5. Select a bounded mitigation with a clear stop condition. Examples include disabling an expensive optional feature, reducing worker concurrency, pausing a problematic consumer, or rolling back a compatible application release. Record the expected signal change and verify it afterward. Do not apply every plausible fix simultaneously.

## A runbook should support decisions

Use this template for each actionable alert:

```text
Runbook: Task API sustained write failures
Owner and escalation: team/role and maintained contact reference
Trigger: precise alert condition and affected objective
Scope check: environment, region, deployment revision, tenant distribution
Initial evidence: dashboard, logs, traces, dependency health
Decision A: dependency unreachable -> connectivity/credential checks
Decision B: lock contention -> identify blocker and responsible workload
Decision C: regression after release -> verify rollback compatibility
Mitigation: bounded action, authorization, target, expected outcome
Stop/abort: conditions that make the action unsafe or ineffective
Recovery validation: reads, writes, consistency, latency, queue recovery
Communication: owner and next checkpoint
Follow-up: incident record, evidence preservation, repair owner
Last exercised: date and drill evidence
```

A command without a decision rule can be dangerous. Explain when to use it, what output means, and how to undo or stop the action where possible. Prefer scoped commands over global restarts or deletion. For a traffic rollback, name the exact artifact or route target instead of “deploy the previous version.”

## Mitigate without making recovery harder

Preserve evidence before ephemeral processes disappear when practical, but do not delay urgent mitigation merely to collect perfect diagnostics. Capture relevant metrics, deployment identity, and a small redacted log sample. Avoid repeated restarts that erase state and create connection storms.

For suspected data corruption, stop additional harmful writes where authorized and preserve recovery sources. Do not restore over the only remaining copy of the database. Recover into a separate environment, validate, and plan reconciliation. For security incidents, follow the organization's incident process, preserve appropriate evidence, and coordinate credential rotation and access containment.

If requests time out, remember that the operation may still have committed. Blindly retrying non-idempotent writes can duplicate effects. Check idempotency records or durable state before replaying work. Queue recovery should account for duplicate delivery, poison messages, and a backlog that can overload dependencies when consumers resume.

## Recovery, learning, and verification

Recovery means the user journey and data properties are healthy, not merely that the alert stopped. Verify writes persist, reads are correct, tenant boundaries hold, queues drain at a sustainable rate, and error/latency signals remain stable. Keep observation long enough to see ordinary traffic and scheduled work. Explicitly hand over remaining risk when ending active response.

Write a blameless postmortem covering impact, detection, timeline, contributing conditions, response decisions, and follow-up actions. Explain why reasonable people made the decisions they did with the information available. Assign actionable improvements to owners and dates; “be more careful” is not an engineering control. [Google SRE postmortem culture](https://sre.google/sre-book/postmortem-culture/)

Common mistakes are waiting for certainty before coordinating, changing several things at once, hiding uncertainty in updates, equating restart with repair, and closing the incident without validating data. Run a tabletop exercise for a failed migration and a practical local exercise for database unavailability. Have someone unfamiliar with the runbook execute it, then revise ambiguous steps and record the drill's evidence.
