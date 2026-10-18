# CI/CD and release strategies

[Tutorial index](../README.md) · Related: [dependency management](../06-quality/05-code-quality-and-dependency-management.md)

## Outcome and prerequisites

Design a delivery pipeline that turns reviewed source into an identifiable artifact and releases it with measurable safety. You need Git, the capstone build and tests, and familiarity with containers. The pipeline examples are design templates; no repository secrets, hosted runner, registry, or deployment account is assumed to exist.

Continuous integration verifies changes frequently against shared expectations. Continuous delivery keeps a tested artifact ready for release, potentially with a human release gate. Continuous deployment automatically releases qualifying changes. The terminology matters less than identifying exactly which checks, approvals, and credentials separate an untrusted change from production.

## Separate verification from publication

Run inexpensive checks early: requirements/lock consistency, Django system checks, optional static typing, formatting, and pure tests. Follow with real PostgreSQL integration tests, migration drift checks, upgrade tests, contract checks, and a build of the actual runtime image. Scan that artifact and create an inventory of its components. Smoke-test the built artifact so source-only tests cannot conceal a missing runtime file.

Pull-request jobs often execute contributor-controlled code. Do not expose release credentials or privileged infrastructure merely because the workflow definition looks trusted. Dependency install scripts, test scripts, and build scripts can execute arbitrary code. Keep publishing in a separate trusted context with narrow permissions. GitHub's guidance covers immutable action references, least privilege, and untrusted input handling. [GitHub Actions secure use](https://docs.github.com/en/actions/reference/security/secure-use)

Cache downloaded dependencies to improve speed, but do not make correctness depend on a cache hit. Key caches by relevant lockfile, platform, and toolchain identity. Never put secrets or credentials in shared caches. Build outputs need a stronger provenance story than “this directory happened to exist on a runner.”

## Walkthrough: specify a reviewable pipeline

1. Write the following stages as a pipeline specification. This is pseudocode, not a vendor-specific YAML file:

```text
on reviewed source revision:
  create isolated workspace
  select recorded Python and package-management versions
  install locked dependencies
  run Django system and migration drift checks
  run pure tests
  start disposable PostgreSQL and apply migrations
  run integration and contract tests
  build runtime image once
  smoke-test image against disposable dependencies
  scan image and produce SBOM
  publish immutable image digest and provenance
  deploy that digest to staging
  run bounded smoke and compatibility checks
  evaluate release policy
  progressively deploy same digest to production
  observe error, latency, and business correctness signals
  record completion or execute defined rollback/roll-forward
```

2. Make each stage produce evidence: source revision, test results, migration revision, image digest, scanner policy result, provenance reference, and environment configuration revision. Do not store raw credentials in this evidence bundle.

3. In a clean virtual environment, emulate the verification core with `python -m pip install --require-hashes -r requirements.lock`, `python manage.py check`, `python manage.py makemigrations --check --dry-run`, and `python manage.py test`, after preparing the local test database service. The first command requires a complete pinned hash-checked lock generated from reviewed requirements; the capstone's ranged lab requirements are insufficient. [pip secure installs](https://pip.pypa.io/en/stable/topics/secure-installs/)

4. Deliberately introduce a failing assertion in a learning branch. Verify publication cannot occur. Then simulate an unavailable registry or failed staging health check and ensure deployment stops with a clear reason. Pipeline failure paths deserve tests because a workflow that only works on success is not a reliable release mechanism.

5. Define a release record: owner, source revision, artifact digest, migration effects, flags changed, rollout steps, stop conditions, rollback target, and monitoring links. The record should make the next operational decision possible without reading chat history.

## Choose a rollout strategy

| Strategy | How it works | Main tradeoff |
| --- | --- | --- |
| Replace | Stop old version, start new | Simple, but usually downtime |
| Rolling | Replace instances gradually | Requires mixed-version compatibility |
| Blue/green | Prepare a second environment, switch traffic | Extra capacity and state coordination |
| Canary | Expose a small controlled traffic share first | Needs reliable comparative signals |
| Feature flag | Deploy code, enable behavior separately | Requires flag lifecycle and safe defaults |

Rolling updates require new and old instances to understand the same database and message formats. Kubernetes Deployments support rolling-update controls, but those controls do not guarantee application compatibility or zero dropped requests. [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

Canary analysis should compare like populations. Routing only internal staff to a canary may miss large tenants or geographic conditions. Measure request success, latency, and meaningful outcomes such as task writes persisting correctly. A low-error canary can still be wrong if it silently omits data.

Feature flags need ownership, an expiry plan, observable state, and authorization around changes. Distinguish a release flag from a permanent tenant entitlement. Test both flag states and transitions where they affect storage or events. A kill switch must work when the affected dependency is degraded.

## Database changes and rollback

Use expand-and-contract migrations for incompatible schema changes. Add a compatible structure, deploy code that can handle both representations, backfill with bounded batches, switch reads, verify correctness, and remove the old structure only after old readers and writers are gone. Track backfill progress and make restarts safe.

Rolling back application code is not the same as rolling back data. A previous binary may fail after a column is removed or may misinterpret newly written enum values. A destructive down migration may lose information. Decide before deployment whether the safe response is code rollback, feature disablement, traffic shift, or a forward fix.

Run migrations once through a controlled job with locking and timeout policy, not opportunistically from every replica at startup. Test lock duration on representative data. A migration that completes instantly on ten rows can block a large table for an unacceptable time.

## Verification, common mistakes, and exercises

After deployment, verify the running digest, readiness, synthetic critical journeys, error rate, latency distribution, and queue lag. Keep an observation window long enough to encounter relevant traffic, while defining immediate stop conditions for severe errors. Record success only after these checks, not merely after the deployment command exits.

Common mistakes are rebuilding separately for each environment, deploying `latest`, allowing parallel releases to race, granting all jobs production credentials, and treating a green health endpoint as sufficient business validation. Another is maintaining a rollback command that nobody has exercised.

Exercise: write a rollout for adding a new nullable task field, then a separate rollout for making it required. Show which versions can run together and how a partially completed backfill resumes. Finally simulate a canary with increased conflict responses and explain whether that signal represents a code regression, a traffic difference, or a change in caller behavior.
