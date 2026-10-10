# 08 — Testing: pure logic, hand-written checks, no framework

## The constraint

A solo project has a specific economics: the cost of test *infrastructure* is
paid immediately, and the benefit arrives later. Installing a test runner,
choosing an assertion library, wiring coverage and keeping all of it working is
a real cost — and on a project this size it would have consumed the budget
without addressing the actual risk.

The actual risk was not "does this function work". It was:

> **Will the notification behaviour change silently when I replace the mechanism
> that produces it?**

That question drove the whole approach.

## The approach: extract the decision, test the decision

Every piece of logic that decides *something* was extracted into a **pure
module** with no framework, no I/O and no dependencies:

| Module | The decision it owns |
|---|---|
| Message building | Given an event and a language, what text? |
| Notification intent | Does this write deserve a notification at all? |
| Coalescing | Given a burst of intents, how many notifications? |
| Freshness guards | Is this change recent enough to announce? |

Other suites cover extraction validation, log redaction, AI error mapping and
pending-write matching — the same pattern, applied wherever a decision hides.

Then each module got a **check script**: a plain Node file that imports the
compiled module, runs assertions, prints a result, and exits non-zero on failure.

**Seven suites, 265 assertions**, split by where the logic lives:

```bash
# one suite at the repository root — message building
$ npm run check:notify          # 46

# six in the client project
$ npm run check:extract         # 74
$ npm run check:notify-queue    # 59
$ npm run check:notify-guards   # 40
$ npm run check:ai-error        # 18
$ npm run check:log             # 15
$ npm run check:pending         # 13
```

No test runner, no assertion library, no config file, no coverage tooling.
All seven run in seconds.

## The one implementation detail worth stealing

Testing a TypeScript module from a plain Node script normally means adding a
build step to the repository. Instead, each check script compiles *only the
module under test* into a temporary directory, requires it from there, and
deletes the directory on exit:

```
check script → compile one file to a temp dir → require it → assert → cleanup
```

The project keeps a single source of truth, the tests stay plain Node, and
nothing generated is ever committed.

## What this bought — the reason it exists

The notification system was migrated from a server-side mechanism to a
client-driven one. The rules that decide *whether* a notification is deserved
are subtle — automatic data fills must not notify, adding a person to a trip
roster must not notify, but *linking* an account to that person must.

Those rules were **encoded as pure functions and covered by tests before the old
mechanism was removed**. The migration could then be validated as "same
decisions, different transport" instead of "hope it still behaves the same".

That is the entire payoff of the approach: not coverage for its own sake, but
**the ability to replace a subsystem without changing behaviour**.

## Regression coverage in practice

The notification suites each correspond to a bug that actually happened:

| Suite | The bug it prevents from returning |
|---|---|
| Intent rules | Automatic fills producing notifications |
| Coalescing | One notification per photo in a multi-photo import |
| Queue cleanup | Retrying forever against a record that was deleted |
| Freshness guards | Notifications for old data when the app opens |

The last one is the double-emit bug described in [04](04-stateless.md) — the
test asserts the exact scenario that produced it.

## What is *not* covered

Stated plainly, because a testing section that only lists wins is marketing:

| Gap | Consequence |
|---|---|
| **No end-to-end tests** | The full path (client → server → push channel → device) is validated **manually in production**, with a small script of repeatable checks |
| **No integration tests** | Server behaviour against a real database is verified by hand |
| **The suites do not run in CI** | The only CI workflow builds release artefacts. The checks are run locally before each change. ⚠️ This is a real gap: a local habit is not an enforced gate. |
| **Client UI is not tested** | Deliberate: the cost/benefit on a solo project did not justify it |

## The principle

> **Extract the decision from the side effect.**
>
> Anything that *decides* becomes a pure function and gets tests.
> Anything that *performs* stays thin, and is validated by hand.

This is what makes the suite small enough to maintain and sharp enough to matter.
It is also why the migration in [04](04-stateless.md) was possible at all.

→ Back to [README](../README.md)
