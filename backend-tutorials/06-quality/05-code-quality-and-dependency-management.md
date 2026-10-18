# Code quality and dependency management

[Tutorial index](../README.md) · Related: [CI/CD](../07-operations/03-ci-cd-and-release-strategies.md)

## Outcome and prerequisites

Create a maintainable change process that combines readable boundaries, automated checks, and an explicit dependency lifecycle. You need Git fundamentals, the Python/Django capstone, and an activated virtual environment. This tutorial describes checks to add in a learning branch; it does not assume linting, formatting, or dependency-scanning tools are already installed.

Code quality is the ability to change behavior with understandable consequences. It includes correctness, clarity, testability, operational visibility, and control of coupling. A short function can still conceal a dangerous dependency; a longer function can be reasonable when it describes one cohesive transaction. Treat style rules as shared conventions and architectural rules as constraints that protect important properties.

## Make dependencies visible

Separate transport concerns, business decisions, persistence, and external integrations when the separation improves comprehension. A route should identify the caller, parse validated input, invoke behavior, and map results to the protocol. A repository should express persistence semantics. Keep transaction ownership clear: helpers should not silently commit or acquire unrelated connections halfway through a business operation.

Prefer explicit dependency parameters over global mutable clients when tests or lifecycle management benefit. Avoid making every function an interface merely to satisfy a pattern. Introduce an abstraction when there are meaningful alternative implementations or a stable boundary to protect.

Model invalid states deliberately. Python type annotations help readers and optional type checkers, but they do not automatically validate HTTP inputs or database contents. Check parsed JSON types explicitly. Remember that `bool` is a subclass of `int`, so a naive integer check may accept `true` as a version number. Django model choices improve model-level validation but database constraints are still needed where persistence must enforce the rule.

Use `python manage.py check` to run Django system checks and evaluate deployment-specific checks under actual production-like settings. They supplement review and tests; they cannot infer every business rule. [Django deployment checklist](https://docs.djangoproject.com/en/5.2/howto/deployment/checklist/)

## Walkthrough: establish a repeatable verification lane

1. Inspect the capstone's requirements and settings. Record the supported Python release and package-management tooling. The lab uses Django `~=5.2.0` and `psycopg[binary]>=3.2,<4`; those ranges are teaching inputs, not a fully reproducible release lock. Generate a reviewed exact dependency lock with your chosen tool before building release artifacts.

For one concrete optional workflow, install a reviewed pinned `pip-tools` version in a development environment, copy the intended direct requirements into `requirements.in`, and run `python -m piptools compile --generate-hashes --output-file=requirements.lock requirements.in`. Add a reviewed Gunicorn requirement when preparing the Linux production image. Compile and verify for the target Python/platform, inspect the resulting pins, and commit both input and lock. A Windows-only resolution can differ from Linux because of environment markers. [pip-tools locking workflow](https://pip-tools.pypa.io/en/stable/)

2. In a clean disposable checkout and new virtual environment, use the following commands after installing the capstone requirements. Database tests require the local PostgreSQL service and its documented settings.

```powershell
python -m pip check
python manage.py check
python manage.py makemigrations --check --dry-run
python manage.py test
```

These should finish successfully when configuration, migrations, and required services are present. `pip check` detects incompatible installed dependency requirements; it is not a vulnerability scanner. For a release, install from a complete pinned hash-checked file using `python -m pip install --require-hashes -r requirements.lock`. That command requires every applicable dependency and hash to be present; the lab's ranged requirements file does not satisfy it. [pip secure installs](https://pip.pypa.io/en/stable/topics/secure-installs/)

3. Change a model field in a learning branch without creating its migration. Confirm the migration check fails. Restore it. Add a regression test that fails on a known behavior change, then restore correct behavior. If using an optional type checker, introduce a type mismatch and verify that its separate check fails; Python does not enforce annotations by itself.

4. If adopting Ruff, select a version deliberately and commit its configuration and dependency pin. `ruff check .` and `ruff format --check .` are check-only CI commands; automatic fixing is a separate developer action. Teach rules that prevent actual defects and explain local exceptions rather than disabling a whole tool. Ruff is optional and must be installed in the development environment first. [Ruff documentation](https://docs.astral.sh/ruff/)

5. Review a sample change for authorization boundaries, transaction behavior, resource cleanup, backward compatibility, and observability. Automated style checks free reviewers to focus on these higher-impact questions.

## Understand the dependency graph

Direct dependencies appear in your input requirements or project metadata. Transitive dependencies arrive through those dependencies. Development dependencies run during builds and tests, so they can still affect artifact integrity even when omitted from the runtime image. Optional extras, wheels, and native extensions may vary by platform; verify the target operating system, Python ABI, and architecture used in deployment.

Version ranges express permitted selections, not a guarantee that an upgrade is harmless. A lockfile records resolved versions and, when supported, integrity metadata, improving reproducibility. It does not prove that a package is trustworthy or that building a source distribution is safe. Build backends can execute code; review the package's role and source before adding it.

Use `python -m pip list` and `python -m pip show Django` to inspect installed packages and dependency metadata. A dependency-tree tool is optional and requires explicit installation. Check maintenance activity, licensing fit, release practices, reachable functionality, and whether a small local implementation would be clearer. Avoid replacing a mature cryptographic library with custom cryptography just to reduce package count.

For each vulnerability finding, identify the affected artifact, vulnerable version range, exploit preconditions, reachable code path, available fix, and compensating controls. Prioritize exposed and exploitable paths while still tracking build-system issues. Record accepted risk with an owner and expiry date. An optional Python advisory scanner must be installed and configured separately; a clean report against known advisory data is not proof that all dependencies are safe.

## Supply chain and update workflow

Update dependencies in reviewable batches. Separate major migrations from routine patch updates so failures remain attributable. Read upstream release notes, run compatibility tests, inspect generated lockfile changes, and test the built artifact rather than only the source tree. Pin container releases by verified digest and CI actions by reviewed commit SHA when creating a hardened release pipeline.

Generate a software bill of materials for the release artifact and retain provenance connecting it to source, builder, and inputs. Verify attestations at the consuming boundary; producing a signature nobody checks gives limited protection. Keep registry permissions narrow and use short-lived credentials where supported. GitHub recommends immutable action pinning and careful treatment of untrusted workflow input. [GitHub Actions secure use](https://docs.github.com/en/actions/reference/security/secure-use)

Do not expose release credentials to untrusted pull-request code. Separate testing from publishing, require the appropriate review gates, and verify the exact artifact promoted to production. Rebuilding separately for each environment can accidentally create different artifacts from the same nominal source revision.

## Exercises and review checklist

Trace one runtime dependency through its transitive graph and explain why it exists. Perform a small update in a learning branch and document release-note changes, lockfile differences, and test evidence. Then design a policy for an advisory with no fix: assess exposure, reduce risk, track ownership, and set a revisit date.

Before merging a change, check that names reveal intent, errors preserve useful context without leaking secrets, asynchronous work is awaited, clients are released, and new abstractions justify their cost. Before releasing, check that the artifact is reproducible, dependencies are inventoried, runtime configuration is documented, and rollback compatibility is understood. Quality emerges from these habits together; no single lint score represents it.
