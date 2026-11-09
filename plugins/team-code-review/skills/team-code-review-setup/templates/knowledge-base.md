# <Project name> — review knowledge base

<!--
This file is what the team-code-review skill reads before every review. It should hold what a new
senior engineer would need to review a PR well on day one: what the product does, the rules that
must never break, the decisions already made, and the conventions reviewers enforce.

Write rules, not essays. Each item should be checkable against a diff. Delete any section that
doesn't apply. Keep secrets OUT of this file: no passwords, tokens, hosts with credentials.
-->

## 1. Product and domain

<!-- 5–10 lines. What the system does, who uses it, and the words the code uses for them. -->

- **What it is:**
- **Audiences / roles:** <!-- e.g. customer, merchant, driver, staff, partner API, webhooks -->
- **Glossary:** <!-- domain term → what it means → table/model name -->
  - `term` — meaning (`table_name`)

## 2. Business rules and invariants

<!--
Things that must always be true. A change that breaks one is a Blocker.
Give each an ID so findings can cite it.
Examples: "An order's total equals the sum of its items plus fees, in minor units."
          "A user can only withdraw to a bank account verified in the last 90 days."
-->

| ID | Rule | Where it is enforced | Why |
|---|---|---|---|
| BR1 | | `path/to/service` | |

## 3. Decisions (ADRs)

<!--
Decisions already made, so reviewers don't re-litigate them and changes don't silently undo them.
A PR that contradicts one must say it supersedes it.
-->

### D-001 — <title>
- **Decision:**
- **Why:**
- **Consequences for code:** <!-- what a reviewer should check -->
- **Status:** accepted | superseded by D-0xx

## 4. Architecture and layering

- **Stack:** <!-- language, framework, ORM/query builder, DB, cache, queue -->
- **Request flow:** <!-- e.g. route → middleware (auth, context) → controller → service → repository -->
- **Where things live:** <!-- module layout, router mounting, audience prefixes -->
- **Background work:** <!-- queues, consumers, cron; what must not run in the request path -->

## 5. Conventions

### 5.1 Validation
- Path params: <!-- helper and how it is wired -->
- Query/body: <!-- schema library + middleware/helper; unknown fields stripped? -->

### 5.2 Responses and errors
- Response envelope:
- Expected (client) errors:
- Unexpected errors:
- i18n: <!-- are messages translation keys? where are they translated? -->

### 5.3 Data and schema changes
- Migration process: <!-- migration tool, or raw DDL applied out-of-band? who applies it, where first? -->
- Table/column naming:
- Money: <!-- integer minor units? column names? -->
- Big/hot tables (need online DDL, bounded queries, indexes):
- Indexes on hot tables: <!-- name → columns, so reviews can check query plans without DB access -->
- Tables whose rows drive limits or counters elsewhere: <!-- e.g. failed-attempt logs that feed a daily cap -->

- Soft delete / "live row" columns:

### 5.4 Concurrency and idempotency
- Lock / serialisation helper:
- Idempotency pattern for mutations:
- Read replica vs writer: <!-- what must read from the writer -->

### 5.5 Queues, crons and background jobs
- Queues and what they serialise: <!-- e.g. FIFO queue X, single message group = one wallet post at a time -->
- Consumer contract: <!-- base class/interface, poison-message error type, ack/retry rules -->
- DLQ and timeouts: <!-- maxReceiveCount, visibility timeout, lock TTLs that must line up -->
- Cron schedule dependencies: <!-- job A must finish before job B; date windows each job selects -->
- Helpers that already retry: <!-- e.g. batch send retries failed entries internally -->

### 5.6 Naming and style
- <!-- naming rules, comment policy, file/function size expectations -->

### 5.7 Helper table — search these before writing a new helper

| Need | Use | Location |
|---|---|---|
| Parse a path id | | |
| Validate a body/query | | |
| Paginate | | |
| Format money | | |
| Lock / single-flight | | |
| Parse JSON safely | | |

## 6. House rules

<!--
Your team's recurring review comments, written as Bad → Good. These are the most valuable part of
this file: they are what makes the review sound like your reviewers. Setup can draft these from your
past PR review comments.
Severity: Blocker | Should-fix | Nit
-->

| ID | Severity | Rule | Bad → Good |
|---|---|---|---|
| H1 | | | |

## 7. Sweep patterns

<!--
Regexes for patterns that keep coming back. The review greps the added lines for each one.
-->

| Pattern | Catches | Rule |
|---|---|---|
| `Number\(req\.params` | unparsed path ids | V1 |

## 8. Security notes

- **Sensitive assets:** <!-- money, PII fields, credentials, step-up flows -->
- **Auth model:** <!-- token type, header, how roles/permissions are checked, any proxies -->
- **Known threat patterns in this codebase:** <!-- past incidents, classes reviewers watch for -->
- **Webhooks:** <!-- providers, how signatures are verified, de-duplication -->

## 9. Environments and release

| Environment | Safe for verification runs? | Shares with production (DB / cache / queues)? | Notes |
|---|---|---|---|
| local | yes | | |
| staging | no | <!-- e.g. DB: yes, Redis: ?, queues: own --> | |
| production | never | — | |

- **Base branch for PRs:**
- **Deploy order rules:** <!-- backend before clients? config before code? -->
- **What a PR description must contain:** <!-- impact, DDL, config, rollout flag, … -->

## 10. Path map (optional, for large knowledge bases)

<!-- Code paths → sections above, so the review reads the right sections in full. -->

| Code path (glob) | Read sections |
|---|---|
| `src/payments/**` | 2, 3 (D-001), 5.4, 8 |

<!-- Also list sensitive entry points by name (e.g. `LedgerService.post`, `PaymentProvider.charge`), so
a change that calls them from any module gets the matching checklist. -->

| Sensitive call | Read sections |
|---|---|
| | |
