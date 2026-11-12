---
name: code-review
description: Review git diffs, pull requests, branches, or commits for bugs and correctness problems, and prove each issue with a failing unit test instead of just describing it. Use this skill whenever the user asks to review a PR, diff, branch, commit, or "my changes", asks "is this change safe", "what could break", "find bugs in this", or wants test coverage checked on changed code, even if they never say the words "code review". The skill never fixes the bug; it writes a test that fails now and passes only once the bug is fixed.
---

# Code Review (bugs proven by failing tests)

The goal is a review where every finding is undeniable. A comment like "this might overflow" is easy to dismiss or misjudge; a failing test is not. So each reported issue ships with a unit test that fails today and will pass only when the issue is genuinely fixed.

Two rules shape everything else:

1. **Do not fix the bug.** Never edit source code, and don't include patches or rewritten implementations. The author owns the fix. The only files you create or change are test files. Describing the expected behavior is fine and necessary; supplying the corrected code is not.
2. **No test, no finding.** If you cannot make a test fail for the reason you claim, you don't have a bug, you have a suspicion. Drop it, or list it separately as unproven (see Output).

The review focuses on **bugs/correctness** and **test coverage**. Style, naming, and formatting opinions are out of scope unless they cause incorrect behavior. Skipping them keeps the signal high.

## Workflow

### 1. Get the diff

Use whatever the user gave you; otherwise find it yourself:

- Pull request: `gh pr diff <n>` (and `gh pr view <n>` for the description)
- Branch: `git diff <base>...HEAD` (find base with `git merge-base` or default branch)
- Commit(s): `git show <sha>` / `git diff <sha1>..<sha2>`
- Uncommitted work: `git diff` plus `git diff --staged`
- Pasted or uploaded diff: use it directly

Also read the PR description and commit messages. They tell you what the change is *supposed* to do, which is the yardstick for "incorrect".

### 2. Learn the project's test setup

Before writing any test, find out how this project tests things: look at existing test files near the changed code, the test runner config (`pytest.ini`, `pyproject.toml`, `package.json`, `jest.config.*`, `go.mod`, `Cargo.toml`, `pom.xml`, etc.), naming conventions, fixtures, and mocking style. Tests must match the project's framework and conventions so the author can run them with the command they already use.

### 2a. AgroTefter: where tests run

The only unit tests are the server's API tests: `server/test/*.test.ts`, written with Node's own runner (`node:test`, `node:assert/strict`) and run against a real PostgreSQL. `scripts/test.sh` runs every check (app typecheck and build, server typecheck, API tests); see `README.md`.

**Run them in Docker.** The server's TypeScript runs without a build step, which needs a Node built with type stripping (22.18+); the system Node on this machine is built without it, so `npm test` in `server/` fails with `ERR_UNKNOWN_FILE_EXTENSION`. The `server-test` service in `docker-compose.yml` has the right Node and starts the database it needs:

```bash
docker compose run --rm --build server-test                                         # all API tests
docker compose run --rm --build server-test node --test test/review-<topic>.test.ts  # one file
```

`--build` matters: the image holds a copy of `server/src`, so without it a change to the server is not what gets tested. `server/test` is mounted into the container, so a new test file is picked up either way. `.env` must exist (copy `.env.example`).

**Where review tests go:** `server/test/review-<topic>.test.ts`. Copy the setup from `api.test.ts`: `buildApp` with `app.inject`, a fresh `remoteAddress` per request (the register and login routes allow 10 requests a minute per address), and accounts named `test-...`.

**Clean up.** The tests use the same database as `docker compose up`, the developer's own. Delete every account, trial code and row a review test creates, in an `after` hook. Never drop tables or the schema.

**The app (`src/`) has no unit-test runner,** only the typecheck and build (`scripts/test.sh app`). Prove an app bug through the server API when the bug is visible there. Otherwise list it under unproven suspicions and say that the project has no app test runner yet; do not add a test framework to the project as part of a review.

**Cross-check against the base** with the same Docker command after checking out the base commit; rebuild (`--build`) after every checkout.

### 3. Read the changes in context

A diff shows only fragments. For each changed function, open the whole file and look at callers, callees, and related types. Many real bugs live at the boundary between changed and unchanged code (a caller that relied on old behavior, a changed return value, a new nullable). Use `grep` to find call sites of any changed signature or behavior.

### 4. Hunt for bugs

Reason about what the code does versus what it should do. Common places to look:

- **Logic**: inverted or wrong conditions, off-by-one, wrong operator, unreachable branches, wrong variable used (copy-paste errors)
- **Boundaries**: empty input, zero, negative, max values, single element, very large input, unicode
- **Null/undefined/missing**: unchecked optional values, missing dictionary keys, empty collections
- **State and ordering**: mutation of shared or input data, stale caches, initialization order, resources not released
- **Errors**: swallowed exceptions, wrong exception type, partial failure leaving inconsistent state, missing rollback
- **Concurrency**: races, check-then-act, non-atomic updates (only when you can build a deterministic test)
- **Types and numbers**: integer overflow, float comparison, rounding, timezone and DST, string vs number coercion
- **Changed contracts**: renamed or reordered parameters, altered return shape, default value changes, behavior a caller still depends on
- **Security-flavored correctness**: missing authorization checks, unvalidated input reaching a query or path (include only if a test can demonstrate it)

Note each suspected bug with the file, line, and the concrete input that triggers it.

### 5. Prove each bug with a test

For every suspected bug, write one focused unit test and run it.

**The test must:**
- Assert the *correct* behavior, so it fails now (the bug) and passes after a proper fix. Never write a test that asserts the buggy behavior.
- Fail for the right reason: an assertion failure showing wrong output or wrong behavior. A failure from an import error, typo, missing fixture, or syntax error proves nothing, so fix the test until the failure is about the bug.
- Be minimal and deterministic: smallest input that triggers the bug, no sleeps, no network, no dependence on wall-clock time or random seeds (freeze or inject them).
- Test behavior, not implementation, so a legitimate fix in a different style still turns it green. Avoid asserting on private helpers or exact internal calls unless that is the contract.
- Have a descriptive name stating the expected behavior (e.g. `test_apply_discount_does_not_go_below_zero`) and a one or two line comment giving the location (`path/file.py:42`) and what's wrong.

**Where to put it:** prefer a new test file named for the review (e.g. `tests/test_review_<topic>.py`, `<topic>.review.test.ts`) in the project's normal test directory, so existing test files stay untouched. Group related tests for the same area in one file.

**Run it.** Execute the project's test command for just that file and confirm it fails with the expected message. Keep the failure output; it goes in the report. If you can't run tests (missing dependencies, no environment, sandbox limits), still write the test, say plainly that it is unverified, and say why. Never claim a test fails if you haven't seen it fail.

If the test passes, your suspicion was wrong. Re-examine once, then drop the finding.

**Cross-check against the base.** When the change comes from a branch or PR, also run the new test on the base branch (stash or copy the test file, check out base, run, return). A test that passes on base and fails on the change proves the change introduced the bug, which is the strongest evidence a reviewer can get. A test that fails on both means the bug predates the change; say so in the report, since it's still real but not this PR's doing. Leave the working tree on the branch you started on.

### 6. Check test coverage of the change

For each behavior the diff adds or changes, check whether any existing test exercises it. Look for changed branches, new error paths, and new edge cases that no test touches. Report these gaps by naming the untested behavior in plain terms. Gaps are not bugs, so they don't get failing tests, with one exception: if probing an untested path with a test reveals a real bug, that becomes a normal finding.

## Output

Keep the report scannable. Lead with the verdict, then findings, ordered most severe first (data loss or wrong results before edge-case quirks).

Each proven issue gets a **brief** summary (one sentence, two at most) followed by a **simple example**: a concrete input, what happens now, and what should happen. The example is what makes the issue click for a reader who hasn't opened the code, so use real values (`qty=0`, `total=25, size=10`, an empty list), not abstract descriptions. Keep the example to two or three short lines.

```
## Review summary
<one or two sentences: what the change does, how many proven issues, overall risk>

## Proven issues
### 1. <short title> — <path>:<line>
**What's wrong:** <one sentence: the bug in plain words>
**Example:** <concrete input> → now: <actual result>; expected: <correct result>
**Test:** `<path/to/test_file>::<test_name>`
**Failure output:**
<the relevant lines of the real failing run, trimmed>

### 2. ...

## Coverage gaps
- <untested behavior introduced by the diff, with file:line>

## Unproven suspicions (optional)
- <something that looks wrong but you couldn't demonstrate, and what blocked you>

## How to run
<exact command to run the added tests>
```

If you found no provable bugs, say so directly. Don't pad the report with speculative issues to look thorough. A clean review with a short list of coverage gaps is a perfectly good outcome.

State at the end that no source files were modified and which test files were added.

## Examples

**Example 1:**
Diff change: `if (qty > 0)` replaced with `if (qty >= 0)` in an order-total function that should reject empty orders.
Finding: zero-quantity orders now pass validation.
Report entry: "**What's wrong:** an order line with zero quantity is accepted instead of rejected. **Example:** `qty=0` → now: order accepted with total 0; expected: validation error."
Test: `test_order_with_zero_quantity_is_rejected` calls the function with `qty=0` and asserts the validation error. It fails today ("no error raised") and passes once the check is corrected.

**Example 2:**
Diff change: a pagination helper now computes `pages = total // size`.
Finding: when `total` isn't a multiple of `size`, the last partial page is dropped.
Test: `test_pagination_includes_partial_last_page` uses `total=25, size=10` and asserts 3 pages. Fails with "expected 3, got 2".

Notice what's absent from both: no corrected code. The report states the expected behavior and the test encodes it.
