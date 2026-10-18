# Data formats, identifiers, time, numbers, and text

[Curriculum](../README.md) · Prerequisite: JSON and basic database types

## Why representation is a backend technique

A valid JSON object can still lose money precision, shift a calendar date, or create duplicate identifiers. Correctness requires an explicit representation at every boundary: client, application runtime, database, message, cache, and exported file. This tutorial builds a representation contract for the Task Service and common extensions.

## JSON and schema boundaries

JSON represents objects, arrays, strings, numbers, booleans, and null. It has no built-in date, decimal, binary, undefined, or bigint type. JavaScript `JSON.stringify` omits undefined object properties, while null is explicit. A PATCH contract must distinguish an omitted property from one intentionally set to null.

Choose what unknown fields mean. Rejecting them catches spelling mistakes and mass assignment; accepting them can aid compatibility for a permissive envelope. Apply the decision separately to requests and responses. Clients generally need to tolerate additive response fields.

Limit request size, nesting depth where the parser supports it, collection lengths, string lengths, and expensive validation. A schema that allows an unbounded array is an operational decision, even when every item is valid.

## Exact numbers and money

JavaScript numbers cannot exactly represent every integer beyond `Number.MAX_SAFE_INTEGER`, and binary floating-point cannot exactly represent many decimal fractions. Do not use a floating-point total as an accounting source of truth.

For a fixed-currency payment, store integer minor units with an explicit currency. For calculations requiring decimal precision, use database `numeric` and a decimal library with a documented scale and rounding mode. Different currencies and business instruments can use different scales. Do not blindly multiply every currency by 100.

```json
{"amount_minor":"125000","currency":"USD","scale":2}
```

This example uses a string to preserve large integers across clients. The contract says it represents 1,250.00 units with scale 2. An actual payment integration must use its provider's currency rules. Never infer arithmetic meaning merely from the displayed string.

Practical experiment in Python:

```python
from decimal import Decimal

print(0.1 + 0.2 == 0.3)  # False
print(Decimal("0.1") + Decimal("0.2") == Decimal("0.3"))  # True
print(125000 + 250)  # 125250
```

Python integers have arbitrary precision, but a JavaScript client may lose that precision when parsing a large JSON number. Decimal arithmetic still needs a rounding policy. Construct Decimal values from exact strings, not already-rounded floating-point numbers.

## Instants, dates, local times, and durations

These are different domain types:

| Meaning | Example | Suitable representation |
| --- | --- | --- |
| Event instant | Task created at a moment | UTC instant / PostgreSQL `timestamptz` |
| Calendar date | Birthday or due date without a time | `YYYY-MM-DD` / SQL `date` |
| Local appointment | 09:00 in a named city | Local fields plus IANA zone and resolution policy |
| Elapsed duration | Request took 42 ms | Monotonic elapsed-time measurement |
| Recurrence | Every weekday at 09:00 | Schedule plus zone, not fixed 24-hour addition |

`timestamptz` represents an instant; it does not retain the original named timezone. Store the zone separately when that name is business data. PostgreSQL distinguishes timestamp types and converts timezone-aware values for display using session settings. [PostgreSQL date/time types](https://www.postgresql.org/docs/17/datatype-datetime.html).

Daylight-saving transitions can make a local time nonexistent or ambiguous. Define whether the scheduler skips, shifts, or chooses an occurrence. UTC storage alone does not resolve the meaning of “9 AM tomorrow in the user's city.”

Use a monotonic clock for latency, because wall time can jump. Use database or application wall-clock instants for records, but avoid relying on two machines' clocks as a globally ordered event sequence. Sequence numbers or log offsets can carry ordering explicitly.

## Precision across boundaries

PostgreSQL and Python datetime can preserve microsecond timestamp precision while a JavaScript Date client uses milliseconds. A pagination cursor that truncates precision can skip rows. Preserve the server-generated cursor unchanged on the client. The Django capstone encodes the Python datetime's full ISO string and parses it back without routing it through a browser Date.

Identifiers should be opaque to clients. A random UUID is not an authorization mechanism. Sequential IDs are compact and index-friendly but expose ordering; random IDs distribute key locality differently. Never choose an ID scheme as a substitute for checking tenant ownership.

## Unicode and files

A string's character count, UTF-16 code-unit count, grapheme count, and UTF-8 byte count can differ. Decide whether a limit protects storage bytes or user-visible characters. Usernames and email-like identifiers need explicit case and normalization policy; silently changing arbitrary user text can corrupt meaning.

Treat uploaded filenames as display metadata. Generate storage keys server-side and prevent path traversal. CSV exports need quoting and an explicit spreadsheet-formula policy when untrusted text is opened by spreadsheet software. Binary data usually belongs in object storage or a binary transport; base64 adds size and is not encryption.

## Practical contract lab

1. Write schemas for task title, status, ID, created instant, and optional due date.
2. Test null versus omission and an unknown `is_admin` property.
3. Round-trip a timestamp through SQL, JSON, and the application; compare precision.
4. Round-trip a large integer as a JSON number and then as a string.
5. Test accented text, emoji, combining marks, empty strings, and whitespace-only titles.
6. For a reminder extension, record the timezone and explicitly resolve an ambiguous local time.

Expected result: every boundary either preserves the value or rejects it with an intentional error. Silent coercion is a defect when it changes domain meaning.

## Checklist and exercises

- [ ] Numbers have a range, precision, scale, and rounding policy.
- [ ] Every time value is an instant, date, local time, or duration by contract.
- [ ] Cursors preserve sort-key precision.
- [ ] Null and omitted fields have distinct documented behavior.
- [ ] IDs and filenames never bypass authorization or path controls.

Exercise: design a billing-line schema and a recurring-reminder schema. Explain their behavior across runtime languages and timezone changes. Continue with [relational modeling](../02-data/01-relational-modeling-and-sql.md).
