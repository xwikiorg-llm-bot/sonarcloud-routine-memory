# `java:S2629` — "Invoke method(s) only conditionally"

OKF-denylisted, and rightly: the corpus's own `dropped-issues.md` entry records a **withdrawn PR**
(`xwiki-commons#1888`) that deleted an eager `toString()` on the "SLF4J calls it itself" reasoning.
In XWiki a log argument is stored as an object and XStream-serialised into the job log, so
rewriting the call changes what is persisted — `okf/conventions/logging.md` is the authority.

**But the entry's conclusion ("the remaining sites all need an `isXxxEnabled()` guard, which is a
judgement call") is false for a large, free subset — and the classifier is one token on the flagged
line: THE LOG LEVEL.**

## The free classifier: `warn` / `error` are always enabled

`okf/conventions/logging.md` states it outright: *"`warn` and `error` are always enabled with the
default configuration"*. So on a `warn`/`error` call an `isWarnEnabled()` guard **can never skip
anything** — there is nothing to defer, and the rule's premise (the level may be off) is simply
false. That is a complete, checkable false-positive argument, identical for every such site, and it
needs no per-site dataflow reasoning at all:

```java
    // warn/error are always enabled in XWiki's default logging configuration, so guarding this call
    // could never skip the evaluation of its arguments.
    @SuppressWarnings("java:S2629")
    protected void logWarning(String explanation, Throwable e)
```

`@SuppressWarnings("java:S2629")` on a **method** is already the established in-repo resolution —
`JGroupsNetworkAdapter`, `XWikiHibernateBaseStore`, `MailSenderPlugin`, `WorkspacesMigration`
(platform), `AbstractInstallPlanJob`, `DefaultJobProgress` (commons). Cite them; that precedent is
what makes the batch uncontroversial.

Measured 2026-09-14 over the 37 unclaimed sites: **17 shipped** (platform 10, commons 7), the rest
are `debug`/`info`.

## Mechanics

* The annotation goes on the **enclosing method**, so one annotation clears every S2629 key in it —
  `RightsFilterListener#cancel` held 3 and `FailingTestDebuggingTestExecutionListener#executionFinished`
  held 6. Count by key, not by edit.
* Put the `//` reason immediately above the `@SuppressWarnings`, after any `@Override`. That is the
  form the `checkstyle:InterfaceIsType` suppressions use and it survived review.
* Pure insert above a declaration ⇒ it cannot inherit a pre-existing finding, so the `Quality /
  Analyze` collision pre-check is trivially clean.
* Locate the enclosing declaration by walking up from the flagged line to the nearest `{` alone on
  its line, then up over annotations/Javadoc — Sonar flags the **log call**, not the method.

## The `warn`/`error` classifier is NECESSARY BUT NOT SUFFICIENT — check the ARGUMENT SHAPE too

Review of platform #6379 narrowed the lever, and correctly. *"`warn` is always enabled"* answers
**"would a level guard save anything?"** — it does not answer **"is this call written the way XWiki
wants?"**. Those come apart on exactly one shape: an argument that is a **string concatenation**.

* Nine of the ten platform sites were already in the parameterized form and the rule was objecting
  to the *argument expression* (`ExceptionUtils.getRootCauseMessage(e)`, `getRightDescription(…)`,
  `StringUtils.join(…)`). A guard could only defer those, and at `warn` there is nothing to defer ⇒
  suppression is right.
* `LoggingScriptService#deprecate` was `warn("[DEPRECATED] " + message)` — **concatenation**, which
  the logging best practices forbid outright. There the rule is pointing at a real convention
  violation, and suppressing it blesses the smell. Vincent: *"Looks like the fix is wrong here."*
  The fix is `warn("[DEPRECATED] {}", message)`, which clears the issue for real and lets the
  annotation be **deleted**.

So split the `warn`/`error` pool once more, on one token of the flagged line: **does the argument
list contain a `+`?** Concatenation ⇒ fix it; anything else ⇒ suppress. This is free, and it is the
difference between a suppression a reviewer accepts and one they push back on.

**Two mechanics for the concatenation fix:**

* **SLF4J's placeholder is `{}`, never `%s`.** SLF4J does no printf formatting, so
  `warn("[DEPRECATED] %s", message)` logs the literal `%s` and silently drops the argument. Worth
  saying out loud: it was the form suggested in review, and taking it verbatim would have shipped a
  broken log line. Confirm the logger's type first (`org.slf4j.Logger` here, via `LoggerFactory`).
* **It changes `ILoggingEvent#getMessage()` from the flattened string to the PATTERN**, so any test
  asserting the flattened form on `getMessage()` fails. Adapt it to `getFormattedMessage()` (and
  pin the pattern too, which documents the split). The *rendered* output is byte-identical, so
  console-matching consumers — `LogCaptureValidator` / `registerExpected(...)` in the functional
  ITs — are unaffected; grep for them anyway before changing a message, since they match text.

## Two site-specific reasons worth reusing

* **The flattened message is the contract.** `LoggingScriptService#deprecate` concatenates on
  purpose: `LoggingScriptServiceTest#deprecate` asserts `LogEvent#getMessage()` equals the
  concatenated string. (Same fact the corpus records as "a logging fix can be contradicted by a test
  asserting the RAW log message" — here it is the *suppression's* justification rather than a drop.)
* **Running the call IS the point.** `FailingTestDebuggingTestExecutionListener#executionFinished`
  runs `top`/`lsof`/`docker ps` in order to log their output, and the block already only executes
  under `isInCI()`. `dropped-issues.md` had recorded exactly that sentence as the reason to DROP the
  6 issues — which is the recurring lesson: *a drop reason that describes a deliberate idiom is the
  finished suppression comment.*

## What stays open

The `debug`/`info` sites. There a level guard really would skip work, so whether to add one is a
judgement about the surrounding method, not a false positive — keys are in `dropped-issues.md`.
Do not suppress them with a hand-waved reason; the gate on this pool is that the comment must be
*true and checkable*.

## Outcome

Shipped 2026-09-14 as platform #6379 (10 keys — **9 suppressed, 1 turned into a real fix after
review**) and commons #1976 (7 keys over 2 methods), alongside the `java:S1214` half of the same sweep. 21 `debug`/`info` keys stay open and
are listed in `dropped-issues.md`. Both PRs green first try on `Quality / Analyze` and the
SonarCloud project gate.
