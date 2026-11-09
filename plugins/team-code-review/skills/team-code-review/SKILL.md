---
name: team-code-review
description: Review a pull request or working diff the way your team's senior reviewers do — correctness, security, data integrity, performance, design, reuse and readability — using your project's own knowledge base (business rules, decisions, conventions) and report evidenced findings by severity. Use before opening a PR, before every push to an open PR, when asked to "review this PR/diff/branch", or when a reviewer comment suggests the same issue may exist elsewhere in the change. Requires a one-time setup (team-code-review-setup).
---

# Review a change against your team's rules

You are reviewing as a senior engineer on this codebase. The goal is a change the team's reviewers
approve in one round: no money or data risk, no security hole, no avoidable latency, no duplicated
helpers, and code that reads like the code around it (where that code follows the rules; see X3).

This skill has two layers:
- **The core** (this file): how to review, the passes, the security pass, severities and the report.
  It is the same for every team.
- **The project layer**: your team's knowledge base — business rules, decisions, conventions, helpers,
  environments, data access. It is written once during setup and lives in the repo. Every review
  reads it first.

## Step 0 — Load the project context

1. Find the config: `.code-review/config.md` at the repository root. If the change spans several
   repos, each repo has its own config; review each repo against its own.
2. **No config?** Stop and tell the user the project isn't set up yet, and offer to run the
   `team-code-review-setup` skill. If they want a review anyway, run the core passes only, and put
   "Project layer not configured — reviewed against generic rules only" at the top of the report.
3. Read the config, then read every knowledge-base file it lists, plus any extra checklist whose paths
   match (both, not either). If the knowledge base has a **path map** (code paths → sections), read
   the sections for the paths the change touches in full, and skim the rest. Docs matched only by a
   catch-all glob (`src/**`) are skimmed, not read in full. Skip docs marked draft, seed or example,
   and say in the report if one looked authoritative but didn't match the code.
4. Note from the knowledge base:
   - the **base branch** and how to get the diff;
   - **business rules and invariants** (these are what a correct change must preserve);
   - **decisions** (ADRs) — a change that contradicts one is a Blocker unless the PR says it
     supersedes it;
   - **conventions**: layering, validation, responses, errors, naming, schema changes;
   - the **helper table** — shared utilities to reuse before writing new ones;
   - **house rules** (the team's own Bad → Good rules) and the **sweep patterns** (grep);
   - **environments** — which are safe for verification and which must never be touched;
   - **data access** — whether you may query a database, which one, and how.
5. Domain checklists apply by **reach, not only by path**: if the change calls into a sensitive area
   (a ledger post, a payment provider, auth/step-up) from another module, apply that area's checklist
   and read the callee, even though no file under its path changed.
6. Precedence: house rules and conventions in the knowledge base add to and refine the core passes;
   where they conflict, the project rule wins. Two exceptions the knowledge base cannot switch off:
   the security pass (Step 3) and failing closed on missing identity or permission (S1).

## Step 1 — Scope the change

1. Get the diff against the configured base branch:
   `git diff origin/<base>...HEAD --stat`, then the full diff. For a PR: `gh pr diff <n>`.
   Reviewing fixed commits without checking them out: `git diff "$BASE"..."$HEAD"`, files via
   `git show "${HEAD}:path"` (quote it — zsh treats `$H:s…` as a history modifier), search via
   `git grep <pattern> "$HEAD"`.
2. Read the PR description and the linked ticket: what is the change *supposed* to do? List each
   behavioural claim it makes (status codes, response fields, what is granted or refused, "costs no
   query", "inert until …") and check every one against the code and tests. A claim the code doesn't
   meet is a finding — often the most important one.
3. Note what the change touches: money paths, auth/permissions, schema, queues/cron/webhooks, public
   response shapes, anything the knowledge base marks as sensitive. These raise the bar for the
   whole review.
4. Read whole functions, not just hunks. Reviewers judge every line a PR touches — including old code
   that was moved, wrapped or copied into the diff (X2).

## Step 2 — Review in passes

Go through the passes in order; stop and report a Blocker as soon as you are sure of it. Each pass
lists the generic rules; apply the knowledge base's version wherever it is more specific.

### Pass A — Correctness & security (Blocker tier)
- Does the code do what the ticket says, for every branch, including failure paths?
- Does it preserve every **business rule and invariant** in the knowledge base that it touches?
- Does it contradict a recorded **decision**? If so, does the PR say it supersedes it?
- **Fail closed.** Missing user, membership, permission or config → refuse. Never invent an identity
  or default to a privileged role.
- Authorization is checked for *this* resource (tenant/owner scoping), not just "is logged in".
- No secrets, tokens or PII in logs, URLs, responses or error messages.
- Expected failures return a coded, user-safe message; unexpected ones are neither swallowed silently
  nor leaked raw.
- Concurrency: can two requests interleave and break it (double submit, double payout, lost update)?
  Use the project's lock/serialisation helper; never weaken existing serialisation.
- Step-up checks (OTP/PIN/biometric) are verified in the same request as the sensitive action, or
  via a one-time token issued for that action — never "verified at login, so allowed for N minutes".
- Credentials travel only in the standard auth header, in tokens whose claims say what kind of
  token they are.
- Status codes and messages describe the real state. Real 4xx pass through; unknown upstream
  failures become 5xx/503.
- Risky behaviour changes roll out behind a config switch, so old clients keep working until it flips.

### Pass B — Data integrity
- **One fact, one transaction.** Writes that describe the same change go in one transaction, with
  the transaction handle passed down.
- Side effects that cannot be rolled back — queue messages, emails, push, external API calls — happen
  **after** the commit, never inside it.
- Idempotency: a retry of a mutating path must not double-apply (stable source key / guard).
- Money is integer minor units; ledgers are append-only with compensating entries (unless the
  knowledge base says otherwise, and then follow it exactly).
- Nullable columns: every `!=` / `<>` / `NOT IN` filter on them is NULL-safe — `NULL != 'x'` is NULL,
  so those rows silently disappear.
- Scope every read/update to live rows (active, valid, not soft-deleted).
- Conditional updates gate their side effects: check the affected-row count before logging,
  notifying or paying — another request may have changed the row first.
- Inside a transaction, remember rows you already changed when you filter on them again.
- Changing what state can be revived or overwritten is a product decision — ask, don't assume.
- **Removed safeguards.** If the change removes or bypasses something that serialised, retried or
  guarded a path (a FIFO queue, a lock, a cron's ordering, a validation step), list every old bug that
  safeguard kept dormant. A pre-existing bug the change makes reachable is this PR's finding, at full
  severity — not a follow-up.

### Pass B2 — Queues, crons and async work
Apply when the change adds or moves work into a queue consumer, cron, webhook or retry loop.
- **At-least-once:** every message can arrive twice, late, or out of order. A replay must not
  double-apply; a delayed message must still be processed correctly.
- **Downstream selectors:** if the work creates rows that a later job picks up (by date window,
  status or flag), check that rows created late, from a delayed or replayed message, or for a past
  window are still selected. Rows that no job will ever pick up are lost work.
- **Late data:** a "done for this window" marker must not hide items that become eligible after it was
  set (late completions, retried upstream events).
- **Outcome reporting:** a consumer must not report success when its writes failed, or failure when
  they committed (separate transactions, nested commits). Check what the ack/alert actually reflects.
- **Timing:** visibility timeout vs. lock TTL vs. max receives (DLQ) vs. cron windows. Can a message
  reach the DLQ while its lock is still held? Must the consumer finish before another job runs?
- **Serialisation:** FIFO groups, partitions and single-worker queues are concurrency controls.
  Processing that work elsewhere needs an equivalent lock.
- **Poison messages:** classified with the project's queue error contract, logged, and alerted.

### Pass C — Performance & scalability
- Independent awaits run in parallel. For every second `await`, ask "does it use the first result?"
  If not, run them together — unless an early return skips work for most callers.
- Never start transactions inside a parallel map: each holds a pooled connection, and N in parallel
  can exhaust the pool and deadlock. Loop sequentially instead.
- No query inside a loop over a growing list → one bulk query.
- No duplicate query in one request → fetch once, pass it down; don't re-fetch what you were passed.
- List endpoints are paginated and bounded; no unbounded scans of the tables the knowledge base marks
  as hot or large.
- Reads that must be fresh (balances, just-written rows) use the primary/writer, not a replica.
- Heavy work belongs in a queue/cron, not the request path.
- New filters/sorts on a big table have a composite index in filter-then-sort order.
- Many writes → one bulk insert/upsert. Use "settle all" semantics when one item failing must not
  abort the rest, and handle the failed ones.
- Cache keys include every dimension the value depends on (tenant, country, currency, locale, role).
- Need a variant of a repository method? Add an option to the existing one, not a near-duplicate.

### Pass D — Design
- Layering follows the knowledge base (typically route → middleware → controller → service →
  repository). Controllers stay thin; no DB access in controllers; repositories hold queries only.
- Each function does one job; a long inline callback becomes a named function.
- The change lives in the right module; no cross-module reach-arounds or require/import cycles.
- No speculative abstraction: no option, flag or layer without a current caller.
- Backward compatible: never remove or reshape a live response field or route; new fields are
  additive; deploy order is safe when backend and clients ship separately.
- Config over code for things operations staff will want to change, with safe defaults. Config
  references stable ids, not display names. Lists that will grow are flagged as "move to config".
- Never assume country, currency, timezone or locale; derive them from the record.
- Don't change the behaviour of an endpoint other clients use; add a new, specific one.
- Services own access checks and receive the values they need, never the whole request object.
- Errors reaching the client are specific and (if the project translates) translated.
- Middleware adds to request context; it doesn't replace it.
- Helpers live in the util that matches their subject; repeated logic moves to a util the second time.

### Pass E — Reuse & redundancy
- Before any small helper, search the **helper table** in the knowledge base and the helper folders
  it names — not only the module's own utils.
- A wrapper must add behaviour. If the wrapped function already handles the case, call it directly.
- No copy-pasted logic between files; extend the existing helper instead (additively).
- **Open every callee the diff wraps or duplicates.** For each new retry loop, fallback, guard,
  repository method or "with transaction" variant, read the function it wraps or sits next to and
  ask: does it already retry, guard, accept a transaction, or take an option for this? If yes, the new
  code is redundant (R2) or should be an option on the existing method (P9). This pass is easy to skip
  when the review is busy with correctness — don't.
- **Config enums read at runtime:** an unknown or misconfigured value should fall back to a safe,
  documented default and be logged, not throw in the middle of a batch — unless failing is the
  documented intent. Prefer parsing a family of values (`T1`, `T7`, `T30` → days) over a hard-coded list.

### Pass F — Input validation
- Every path parameter goes through the project's param parser/validator — no ad-hoc `parseInt` /
  `Number(...)` / regex-in-route unless the knowledge base says that is the convention.
- Every query and body goes through the project's schema validation (unknown fields stripped),
  including list filters (enums, dates, search length).
- Validate once, at the edge; don't repeat in the service what the schema already guarantees.

### Pass G — Readability & consistency
- Names say what the thing does and follow the knowledge base's naming conventions.
- Early returns over nested `if`s.
- Matches the surrounding code's idiom and the project's comment policy.
- User-facing strings follow the project's i18n rules and never mention internal routes or fields.
- Follow existing route and column conventions rather than inventing new ones.
- **Rules beat neighbours (X3).** Matching the surrounding code never excuses breaking a rule. When
  the neighbours break it too, flag the new lines and list the old ones as an optional follow-up.
- Compare like with like (cast ids before comparing); prefer the simplest construct.

### Pass H — Tests & release
- New behaviour has tests at the level that would catch a regression. Watch for tests that mock away
  the bug (two transactions mocked as one object, a helper mocked to always succeed).
- If you can't run the tests without changing the working tree, don't; say so in the report.
- Tests cover failure paths and the option that adds a filter, not only the happy path.
- The PR description states: production impact, schema / config / infra changes and where they
  were applied, deploy order, and anything inert until a flag flips — in the form the knowledge base
  asks for.
- Schema changes follow the knowledge base's process (migration tool or raw DDL, naming, charset,
  online-DDL rules for big tables).
- Non-obvious behaviour (rounding, sort fallbacks, edge cases) is explained in the description.
- Open product questions are asked, not guessed.

### Pass I — House rules
Apply every house rule in the knowledge base that the change touches, with its own ID. These are the
team's recurring review comments and carry the severity the knowledge base gives them.

## Step 3 — Security pass

Run this after the passes, on every review, even when the checklist found nothing. The passes ask
"does this break a rule?"; this pass asks "how would someone abuse this change?" Work through the
six stages in order.

### 3.1 Map the change
For each changed or new entry point write down:
- **Entry point**: method + full path (with the router's mount prefix), or the queue consumer / cron /
  webhook handler name.
- **Who can reach it**: the roles and audiences from the knowledge base, plus unauthenticated,
  third-party webhooks, internal queues and cron.
- **Middleware chain**: read the router and list every middleware in order. Missing auth on a new
  router is the first thing to rule out.
- **Untrusted inputs**: params, query, body, headers, webhook payloads, queue message bodies, values
  read from config that staff can edit.
- **Assets touched**: money, credentials and step-up, PII, another tenant's data, feature switches —
  plus anything the knowledge base lists as sensitive.
- **Sinks**: raw SQL, outbound HTTP, file/object-storage paths, logs, responses, emails/SMS, ledger posts.

Then follow the change one hop outward: callers of any changed function, and other routes that now
reach the changed code. A safe-looking diff can open a hole in an old caller.

### 3.2 Hunt
Trace each untrusted input from entry point to every sink. Check each class that fits the map:
- **Broken object access (IDOR)**: a resource fetched or changed by id without scoping to the caller.
- **Missing or wrong authorization**: wrong audience router, permission not checked, role defaulted,
  staff-only action reachable from user routes, a proxy trusting a forwarded identity.
- **Step-up bypass**: a sensitive action possible without fresh step-up, or a token for one step
  reusable for another.
- **Money abuse**: amount, currency, fee or recipient taken from the client; negative or zero
  amounts; double-spend via concurrency or retries; replayable refunds/credits; balance from a replica.
- **Mass assignment**: request body spread into an update; schemas that allow unknown fields; allow-
  lists that let a caller set role, status, verification or balance fields.
- **Injection**: string-built SQL (including sort/column names from input), untrusted URLs in outbound
  HTTP (SSRF), user input in file or object-storage paths.
- **Webhooks and queues**: signature not verified, events not de-duplicated, message body trusted for
  amounts or owner ids that should be re-read from the DB.
- **Brute force and enumeration**: OTP/PIN/login without rate limits or lockout; responses that differ
  by whether an account exists.
- **Data exposure**: PII, tokens or internal fields in responses, logs, errors or URLs; full rows
  returned where a few fields are needed.
- **Fail-open defaults**: missing config, missing row or caught exception that lets the request through.
- Any class the knowledge base lists under its own threat notes.

Write each candidate as: entry point → input → path → sink → what an attacker gains. A candidate
without a concrete path is a question, not a finding.

### 3.3 Challenge
Argue against every candidate before keeping it:
- Is the input really attacker-controlled, or set by middleware/the server?
- Does something upstream already stop it (auth, validation, an access check, a constraint, a lock)?
  Read it; don't assume.
- Can the attacker reach this entry point with the role it needs?
- Is the precondition realistic?
- Is it new in this PR or pre-existing (`git show origin/<base>:<file>`)? Pre-existing holes are
  reported as follow-ups, not Blockers on this PR — unless the PR copied them (X2) or made them
  reachable by removing a safeguard (X6).
- What is the real impact if it works?

A candidate survives only if you can't disprove it. Keep the reason for every one you drop.

### 3.4 Record
Keep a ledger, one row per candidate, surviving or not:

| ID | Location | Class | Attack path (entry → sink) | Impact | Challenge result | Status |
|---|---|---|---|---|---|---|
| SEC-1 | `src/…/x.js:42` | IDOR | `GET /orders/:id` → `findById(id)` | read any user's order | no owner scope in service | pending verification |

### 3.5 Verify
Turn each survivor into evidence, strongest available first:
1. **Code proof**: the exact lines that make the path reachable, including the router mount and
   middleware list, and confirmation nothing on the path blocks it.
2. **Test proof**: a failing test that performs the abuse. Report it; don't commit it unless asked.
3. **Local run**: only against environments the knowledge base marks **safe for verification**
   (normally local). **Never** try an exploit against an environment the knowledge base lists as
   off-limits, and never against production. If the knowledge base doesn't say, local only.
4. **Data check**: only if the config grants read-only data access — counts and read-only queries,
   never writes.

Update the ledger: `confirmed` (evidence attached), `rejected` (how it was disproved), or `needs
validation` (what check would settle it, and who or what it needs).

### 3.6 Report
Every ledger row lands in the Security section of the report:
- **Confirmed** → also listed under Blockers (or Should-fix when impact is low). Attack path, evidence, fix.
  If the same issue is already a finding from the passes, don't repeat it: give the ledger row a one-line
  pointer to that finding.
- **Needs validation** → the open question and the check that would settle it. The PR shouldn't merge
  until it is answered.
- **Rejected** → one line each with the reason.

If there is no new entry point, untrusted input or sensitive asset, say so in one line.

## Step 4 — Sweep before reporting

One finding is usually one instance of a pattern. For every finding, search the **whole diff** for the
same pattern and report all instances together (X1). Run the knowledge base's sweep patterns
against the added lines:

```bash
git diff origin/<base>...HEAD | grep -nE "^\+.*(<pattern from the knowledge base>)"
```

Before flagging a hit, check it is new (`git show origin/<base>:<file>`). Moved or re-indented lines
show as `+` but are not this PR's decision — mention them, don't block on them.

## Step 5 — Verify, then report

Only report what you can point to in the code. For each finding confirm the line, the actual
behaviour, and that the fix is correct (read the helper you recommend — does it handle this case?).
Drop anything you are not confident about, or label it a question.

Severity (the knowledge base may move specific rules between tiers):
- **Blocker** — security, money/data loss or corruption, concurrency bug, broken business rule or
  decision, breaking API/schema change, unsafe schema change on a hot table.
- **Should-fix** — performance, missing transaction or live-row filter, missing validation,
  reimplemented helper, hard-coded country/currency/timezone, layering violation, generic error to
  the client, missing tests.
- **Nit** — naming, early returns, small readability; marked optional.

Output:

```markdown
**Verdict:** Approve | Approve with nits | Changes requested

### Blockers
- `path/file.ext:123` — <what is wrong> (<rule id>) → <concrete fix>. Also at: `file:line`, …

### Should-fix
- …

### Nits
- …

### Security pass
**Attack surface:** <entry points and who can reach them; or "no new attack surface — …">

**Confirmed**
- SEC-1 `path/file.ext:42` — <class>: <entry → sink>. Impact: <…>. Evidence: <…>. Fix: <…>

**Needs validation**
- SEC-2 `path/file.ext:88` — <suspected issue>. Open question: <…>. Settle by: <check>

**Rejected**
- SEC-3 <candidate> — <why it doesn't hold>

### Good
- <one or two things done well>

**Production impact:** <none until merged / what changes when it ships>
```

If nothing is wrong, say so in one line. Don't pad with style opinions.

## Core rule table

| ID | Area | Rule | Bad → Good |
|---|---|---|---|
| S1 | Security | Fail closed on missing identity/permission | fallback admin role → 403 |
| S2 | Security | Scope every resource to the caller | id-only lookup → id + owner scope |
| S3 | Security | Step-up verified in the same request | login-OTP time window → OTP in payload |
| S4 | Security | Credentials only in the auth header, typed tokens | custom token header → typed claim |
| S5 | Security | Status/message match the real state | vague 400 → precise 403 |
| D1 | Integrity | One fact, one transaction | two separate writes → one transaction |
| D2 | Integrity | Side effects after commit | enqueue inside transaction → after |
| D3 | Integrity | Mutations are retry-safe | blind insert → guard on source key |
| D4 | Integrity | NULL-safe negated filters | `col != 'x'` → `COALESCE(col,'') <> 'x'` |
| D5 | Integrity | Filter to live rows | update by id → + active/not-deleted |
| D6 | Integrity | Reviving state is a product call | silent resurrect → ask product |
| D7 | Integrity | Check affected rows before side effects | notify after conditional update → `if (!changed) return` |
| P1 | Performance | Parallelise independent awaits | sequential awaits → parallel |
| P2 | Performance | No queries in loops | N reads → one bulk read |
| P3 | Performance | No duplicate query per request | re-fetch → pass result down |
| P4 | Scalability | Bounded, paginated lists | unbounded scan → page + window |
| P5 | Correctness | Fresh reads on the writer | replica balance → writer |
| P6 | Scalability | Index matches new filter + sort | full scan → composite index |
| P7 | Performance | Bulk writes, grouped work | N upserts → one bulk upsert |
| P8 | Correctness | Cache key has every dimension | `key(type)` → `key(type, country)` |
| P9 | Design | Option on method, not near-duplicate | `findOnWriter()` → `find(p, { useWriter })` |
| P10 | Integrity | No transactions inside a parallel map | parallel transactions → sequential loop |
| G1 | Design | Thin controllers, layered calls | DB in controller → service/repository |
| G2 | Design | One job per function | long inline callback → named function |
| G3 | Design | No speculative abstraction | unused option/layer → delete |
| G4 | Design | Additive API changes only | renamed field → add new, keep old |
| G5 | Design | Never assume country/currency/timezone | hard-coded TZ → from the record |
| G6 | Design | Services own access checks; no request object in services | check in controller → service |
| G7 | Design | Specific errors reach the client | generic 500 → specific message |
| G8 | Design | New endpoint instead of changing a shared one | repurposed route → new route |
| R1 | Reuse | Search the project's helpers first | local helper → existing util |
| R2 | Redundancy | Wrappers must add behaviour | pass-through wrapper → direct call |
| V1 | Validation | Path params through the project's parser | `Number(req.params.id)` → param parser |
| V2 | Validation | Query/body through schema validation | raw `req.query` → schema |
| V3 | Validation | Validate once, at the edge | schema + service re-check → schema only |
| N1 | Readability | Names describe the job | `…Service` that enqueues → `…Producer` |
| N2 | Readability | Early returns | nested `if` → guard clause |
| N3 | Readability | Follow route/column conventions | invented param name → existing one |
| Q1 | Process | Ask product questions, document behaviour | guessed scope → asked |
| T1 | Tests | Test failure paths and filters | happy path only → failing cases |
| X1 | Process | Fix every instance, not one | one line → sweep the diff |
| X2 | Process | Copied old code is in scope | copied pattern → check it too |
| X3 | Process | Rules beat neighbouring code | "that's the convention there" → flag it |
| X4 | Security | Run the security pass every review | checklist only → full security pass |
| X5 | Security | Every security candidate gets a status | dropped idea → confirmed / rejected / needs validation |
| X6 | Process | Removing a safeguard owns the bugs it hid | "pre-existing" → this PR's finding |
| X7 | Process | Checklists apply by reach, not path | wallet post from another module → wallet checklist |
| Q2 | Async | Delayed/replayed work is still picked up | past-dated row no job selects → drain overdue |
| Q3 | Async | Consumer outcome matches committed state | "failed" after commit → report from the committing step |
| R3 | Reuse | Read the callee before adding retry/guard/trx variants | manual retry over a retrying helper → direct call |
| C1 | Config | Runtime config enums fail safe | throw on unknown value → logged fallback |

House rules from the knowledge base use their own IDs (e.g. `H1`, `H2`, …) and appear in findings
the same way.

## When posting review comments
One comment per finding, anchored to the line; say what and why in plain language and give the fix.
Group repeated instances in one comment ("also at …"). Call out good work briefly. Only post when the
user asks you to.
