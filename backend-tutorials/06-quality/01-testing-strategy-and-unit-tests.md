# Testing strategy and unit tests

[Tutorial index](../README.md) · Next: [integration and contract tests](02-integration-contract-and-end-to-end-tests.md)

## Outcome and prerequisites

Build a test strategy that detects important failures without making routine changes expensive. You should understand Python functions, exceptions, modules, and the Task Service domain. The runnable example requires the capstone Python 3.12+/Django 5.2 project and its activated virtual environment. It creates a separate learning module rather than assuming that the capstone already exports these functions.

A useful test answers a specific question: can another tenant read a task, can an invalid title reach storage, or can two writers silently overwrite one another? Passing tests provide evidence about the exercised conditions. They are never proof that every deployment, dependency, input, or interleaving is correct.

## Choose boundaries before choosing a framework

A unit test exercises a small behavior with controllable dependencies. An integration test verifies an actual boundary such as SQL, serialization, or Redis. A contract test checks an agreement between independently changing parties. An end-to-end test follows an important user journey through deployed components. These categories overlap; name the boundary and dependency assumptions instead of arguing about labels.

Use many fast tests for complicated business rules, real database tests for queries and transactions, and a smaller set of full-system journeys. The exact proportions depend on the system. A data-heavy application may gain more confidence from database integration tests than from hundreds of mocked repository calls.

For Task Service, create a risk table:

| Behavior | Best first test | Additional evidence |
| --- | --- | --- |
| Title normalization | Pure function test | HTTP invalid-input test |
| Tenant isolation | Real database integration | Authorized HTTP journey |
| Optimistic concurrency | Two competing database updates | HTTP conflict response |
| Shutdown | Process integration test | Deployment drain exercise |
| Recovery | Restore drill | Application reads restored rows |

Prioritize likelihood multiplied by impact, but include rare catastrophic failures. A typo in a label and a tenant data leak deserve different test depth.

## Walkthrough: deterministic business-rule tests

1. In a learning copy of the capstone, create `tasks/title_policy.py` with this complete module. This tutorial defines a maximum of 120 Unicode code points for its standalone exercise; adapt the number and character policy to the actual API contract before integrating it.

```python
def normalize_title(value: object) -> str:
    if not isinstance(value, str):
        raise TypeError("title must be text")
    title = value.strip()
    if not 1 <= len(title) <= 120:
        raise ValueError("title must contain 1 to 120 code points")
    return title


def next_version(current: int) -> int:
    # bool is an int subclass, so explicitly reject it at this boundary.
    if type(current) is not int or not 1 <= current < 2_147_483_647:
        raise ValueError("invalid PostgreSQL integer version")
    return current + 1
```

2. Create `tasks/test_title_policy.py`. `SimpleTestCase` is appropriate because these functions do not use a database.

```python
from django.test import SimpleTestCase
from tasks.title_policy import normalize_title, next_version


class TitlePolicyTests(SimpleTestCase):
    def test_normalizes_surrounding_whitespace(self):
        self.assertEqual(normalize_title("  Review  report  "), "Review  report")

    def test_accepts_boundary_and_rejects_next_code_point(self):
        self.assertEqual(len(normalize_title("x" * 120)), 120)
        with self.assertRaises(ValueError):
            normalize_title("x" * 121)

    def test_rejects_non_text_and_blank_titles(self):
        for value in [None, 12, {}, [], " ", "\n"]:
            with self.subTest(value=value):
                with self.assertRaises((TypeError, ValueError)):
                    normalize_title(value)

    def test_version_increases_and_rejects_invalid_values(self):
        self.assertEqual(next_version(7), 8)
        for value in [0, -1, 1.5, True, 2_147_483_647]:
            with self.subTest(value=value):
                with self.assertRaises(ValueError):
                    next_version(value)
```

3. Run `python manage.py test tasks.test_title_policy` from the capstone application directory. Expect four passing tests. Change `<= 120` to `< 120`, rerun, and observe a boundary failure. Restore the correct condition. This deliberate mutation checks whether the test can detect the defect it claims to cover.

Django builds on Python's `unittest` framework and discovers test modules through its test runner. Use focused test labels while developing, then run the relevant complete suite. [Django testing overview](https://docs.djangoproject.com/en/5.2/topics/testing/overview/)

## Control time, randomness, and side effects

Inject a clock when business behavior depends on time. A function that accepts an aware `datetime` is easier to test than one that reads the wall clock internally. Inject an ID generator when the generated value matters. Seed randomized generators and report the seed so failures can be reproduced.

A stub returns a chosen response. A spy records calls. A fake implements a simplified working dependency, such as an in-memory repository. A mock usually adds expectations about interactions. Use these techniques at unstable or slow boundaries, not to reproduce every private implementation detail. A mocked PostgreSQL client cannot demonstrate PostgreSQL locking or constraint behavior.

For a notification workflow, assert that a successful state change schedules the required notification, and that a failed transaction schedules none. Avoid asserting the exact sequence of five internal helper calls unless that sequence is itself the requirement. Such tests make safe refactoring unnecessarily painful.

Property-based testing complements examples: normalized titles should be idempotent, accepted versions should strictly increase, and serialization followed by parsing should preserve supported values. Generate malformed Unicode, huge numbers, empty collections, and boundary timestamps deliberately. Record the minimal failing example. Do not treat a random test run as exhaustive coverage.

## Verification, common mistakes, and exercises

Coverage shows what executed, not whether assertions were meaningful. Review untested branches around authorization, conflicts, retries, and cleanup. Mutation testing can reveal missing assertions, but costly or equivalent mutations require human interpretation. Keep explicit tests for important rules even when aggregate coverage looks high.

Flaky tests often expose shared state, real-time sleeps, uncontrolled scheduling, or leaked servers. Give tests independent fixtures, await cleanup, and use an observable completion condition with a deadline. A retry may collect diagnostic evidence; permanently hiding failures behind retries reduces confidence. Quarantined tests need an owner and a repair date.

Practice by adding an idempotence test for `normalize_title`, testing emoji length, and defining whether combining characters count as code points or visible graphemes. Then add tests for the chosen rule before changing implementation. Explain why SQL `varchar` limits and a frontend character counter might disagree.

Before merging, confirm that every test has a behavioral name, relevant boundary cases, reliable cleanup, and a useful failure message. Separate pure tests from infrastructure tests so developers can run the fast group immediately. A regression fix should usually include the smallest test that reproduces the original defect and remains stable after legitimate refactoring.
