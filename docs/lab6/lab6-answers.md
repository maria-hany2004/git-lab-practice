# Lab 6 – Answers

## Task A1 – Coverage
`python -m pytest --cov=registration --cov=invoice --cov-report=term-missing` → **100 %** on both modules.
The one missed line was `registration.py:26` (dropping a course that is not registered);
closed by `test_cannot_drop_unregistered_course`.

**One-sentence answer:** 100 % coverage only proves every line was executed by a test,
not that the assertions are meaningful or that every behaviour/edge case is correct.

## Task B1 – BDD
`features/registration.feature` now has the drop scenarios
("Drop a registered course", "Refuse to drop a course that is not registered").
`python -m behave` → 8 scenarios, 31 steps, all passed.

## Task C1 – Smells in `legacy_invoice.py`

| # | Smell | Where | Why it hurts | Fix |
|---|-------|-------|--------------|-----|
| 1 | Long method / many responsibilities | whole `make_invoice` | Calculates fees, formats text, writes a file and prints | Split: policy computes, `make_invoice` orchestrates, writer does I/O |
| 2 | Conditional chain on `kind` (switch smell) | `if/elif` in the loop + extra `if` after it | Every new student type edits this function (violates Open/Closed) | Replace Conditional with Polymorphism → `*Policy` classes + `POLICIES` dict |
| 3 | Magic numbers | `1200`, `0.5`, `2000`, `5000` | Meaning hidden, duplicated | Named constants (`FEE_PER_HOUR`, `RATE`, `SERVICE_FEE`) |
| 4 | Hard-wired I/O (hidden dependency) | `open(...)`, `print(...)` | Untestable without touching disk/stdout | Inject a `writer` (default `FileWriter`) |
| 5 | Resource leak risk | `open` / `close` without `with` | File stays open if `write` raises | `with open(...)` |
| 6 | Primitive obsession / stringly-typed `kind` | `kind == "regular"` | Typo or unknown kind silently gives total 0 | Lookup in `POLICIES` (unknown kind raises `KeyError`) |
| 7 | Feature envy | loops over `student.courses` | Re-implements `student.total_hours` | Use `student.total_hours` |
| 8 | Scattered related logic | international rate in loop, service fee outside | One rule split in two places | Both live in `InternationalPolicy.fee` |
| 9 | Poor naming / string concatenation | `h`, `f`, `"..." + str(...)` | Hard to read | Clear names, f-string |

## Task C2 – `SiblingPolicy`
Done in TDD, one commit per step:
- **Red:** `test_invoice_total[sibling-6480]` failed with `KeyError: 'sibling'`.
- **Green:** added `SiblingPolicy` (10 % discount on the regular fee) and the `"sibling"` entry in `POLICIES`; all 10 tests pass.
- **Refactor:** added a docstring documenting the 10 % sibling discount; tests still pass.

No change to `make_invoice` was needed — that is the point of the policy refactor.

## Part D – Gherkin for the team story
Story: *As a student, I want to mark my attendance by scanning the lecture's QR code,
so that my presence is recorded without paper sheets.*

```gherkin
Feature: QR attendance
  Background:
    Given a lecture "SET312" with an active QR code
    And a student with ID "2023001" enrolled in "SET312"

  Scenario: Student scans a valid QR code
    When the student scans the QR code during the lecture
    Then the attendance is recorded as "present"

  Scenario: Student scans an expired QR code
    Given the QR code expired 5 minutes ago
    When the student scans the QR code
    Then the scan is rejected with "QR code has expired"
    And no attendance is recorded

  Scenario: Student scans the same QR code twice
    Given the student has already scanned the QR code
    When the student scans the QR code again
    Then the scan is rejected with "attendance already recorded"
    And the student still has exactly 1 attendance record for the lecture
```
