# Documentation, Git, and the engineering workflow

[Curriculum](../README.md) · Prerequisite: familiarity with files and a terminal

## Goals

Make changes reviewable, reproducible, and recoverable. Documentation belongs to the engineering workflow because APIs, schemas, operational decisions, and deployment procedures are part of the system's behavior.

## Useful documentation types

| Artifact | Question it answers |
| --- | --- |
| README | What is this, and how do I run it? |
| Tutorial | How do I learn a technique by doing it? |
| Reference | What exactly are the fields, options, errors, and limits? |
| Architecture decision record | Why did we choose this approach? |
| Runbook | What should an operator do for a concrete symptom? |
| API contract | What can clients send and expect? |
| Migration plan | How do old and new versions coexist? |
| Incident review | What happened and which changes reduce recurrence? |

Separating these purposes helps readers. A tutorial should explain steps and expected observations; a production runbook should prioritize precise actions and verification. A design diagram without ownership or failure behavior is incomplete.

## Git basics without losing work

Git records snapshots as commits. A branch is a movable reference; it does not create an independent copy of external databases or secrets. A remote is a configured repository endpoint, and fetching does not itself merge remote work.

In a new practice directory you own, use PowerShell:

```powershell
git init
git status
git add README.md
git diff --cached
git commit -m "Document local setup"
```

Create the README before staging it. Configure your identity if Git requests it. The commands are an exercise, not a statement that this workspace is already a repository. `git diff --cached` lets you inspect exactly what will be committed.

For an existing team project, read its contribution instructions first. Start from an updated base, make a focused branch, commit a coherent change, and open a review with the problem, resulting behavior, and validation. Prefer an explicit file list when staging. Check for secrets and generated files before committing.

Git's official book explains staging, commits, branches, and conflict resolution. [Pro Git](https://git-scm.com/book/en/v2).

## A repeatable feature loop

1. Define one observable outcome, such as “a stale task update returns a conflict.”
2. Write accepted input, permission rules, invariants, and failure responses.
3. Inspect existing code and naming conventions.
4. Identify schema and compatibility changes before editing handlers.
5. Implement the smallest complete behavior.
6. Verify the behavior at its real boundary: pure function, database, HTTP, or integration.
7. Update the contract and operational instructions affected by the change.
8. Review the diff for accidental files, secrets, and unrelated modifications.
9. Release through the established process and check outcome metrics.

Tests support confidence; they are not an excuse to skip reading a diff. A change can pass tests while exposing a token in a log or making a migration lock a busy table.

## Write a compact decision record

```markdown
# ADR: use conditional versions for task updates

Status: accepted
Context: two editors can overwrite each other's changes.
Decision: require expected_version and update only that version.
Consequences: clients must handle conflicts and refetch before retrying.
Alternatives: last-write-wins; pessimistic locking; field-level merging.
Verification: two concurrent updates from version 1 produce one success.
Revisit when: offline collaborative editing becomes a requirement.
```

Record alternatives honestly. A decision is justified by requirements and constraints, not by popularity. Keep the record short enough to remain useful; link implementation details elsewhere.

## Dependencies and reproducibility

A manifest describes allowed dependencies; a lockfile records a concrete resolution. Commit the lockfile and use a clean install command in CI. Avoid relying on an unrecorded global package. Runtime versions, operating-system assumptions, database versions, and required services also affect reproducibility.

Treat dependency upgrades as changes with release notes, compatibility checks, and rollback planning. An audit finding needs applicability and reachability assessment, not blind dismissal or indiscriminate upgrades. The [dependency management tutorial](../06-quality/05-code-quality-and-dependency-management.md) expands this workflow.

## Practical lab: review your own capstone change

Add a maximum title length rule to a disposable capstone copy if it does not already exist, or change the limit consistently in application validation, SQL, tests, and documentation. Before editing, find every place the contract is represented. A mismatch between an API limit of 200 and a database limit of 100 causes avoidable server failures.

Prepare a review note with the trigger, before/after behavior, tests run, and remaining limits. Then simulate an invalid title and confirm the response is the documented client error. Do not claim an integration test passed if only a parser unit test ran.

## Conflict resolution and recovery

A merge conflict means Git cannot choose the correct combined text. Read both intended behaviors and resolve the semantic conflict, not just the marker lines. A migration filename collision may require ordering decisions even after the textual conflict is gone.

For shared history, a revert usually provides a reviewable corrective commit. Rewriting published history requires coordination. Database rollback is a different problem: restoring code does not recover deleted data or undo a schema change automatically.

## Exercises and completion checklist

Write a README setup section that a second person could follow without guessing directories, ports, credentials, commands, or expected output. Write a runbook for “readiness fails while liveness succeeds.” Link it to an actual diagnostic command.

- [ ] Setup states prerequisites and working directories.
- [ ] Example configuration contains no real secrets.
- [ ] Review notes distinguish checks run from checks still needed.
- [ ] API and schema changes include compatibility considerations.
- [ ] Documentation links resolve and code blocks identify their language.

Continue with [contract documentation](../05-apis/08-openapi-contracts-and-evolution.md) and [CI/CD](../07-operations/03-ci-cd-and-release-strategies.md).
