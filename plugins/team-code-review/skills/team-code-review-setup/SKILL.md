---
name: team-code-review-setup
description: One-time setup for the team-code-review skill in a repository — gathers the project's knowledge base (business rules, decisions, conventions, helpers), optional read-only database access and verification environments, and writes `.code-review/config.md` and `.code-review/knowledge-base.md`. Use when someone installs team-code-review, says "set up code review", "configure the review skill", "update the review knowledge base", or when team-code-review reports that the project isn't configured.
---

# Set up team-code-review for a repository

Goal: leave the repo with two committed files that every review reads first:
- `.code-review/config.md` — where things are and how the review may run (from `templates/config.md`).
- `.code-review/knowledge-base.md` — what reviewers know (from `templates/knowledge-base.md`), or a
  pointer to an existing knowledge base.

Work through the steps with the user. Ask one group of questions at a time, propose answers from what
you can detect, and let them correct you. Don't write anything until Step 6.

**Never put secrets in either file**: no passwords, tokens, API keys, or connection strings with
credentials. Record environment-variable names or tool names instead. If the user pastes a secret,
don't write it down; tell them to put it in an env var or their secret manager and give you the name.

## Step 1 — Detect what you can

Without asking, collect:
- Repo name, default/base branch (`git symbolic-ref refs/remotes/origin/HEAD`, and branches PRs
  usually target: `gh pr list --state merged --limit 20 --json baseRefName`).
- Stack: manifest files (`package.json`, `pyproject.toml`, `go.mod`, `Gemfile`, `pom.xml`, …),
  framework, ORM/query builder, test and lint commands.
- Existing context docs: `CLAUDE.md`, `AGENTS.md`, `.cursor/rules`, `README`, `docs/`, `adr/`,
  `CONTRIBUTING`, PR templates. These are the raw material for the knowledge base.
- Shared helper folders (`util(s)/`, `lib/`, `common/`, `middleware(s)/`) and the validation,
  response and error helpers used most often (grep for imports).
- An existing `.code-review/` folder. If it exists, this is an **update**: show what's there and ask
  what to change, instead of starting over.

Show the user a short summary of what you found.

## Step 2 — Knowledge base source

Ask: "Do you already have a knowledge base (business logic, decisions, conventions) as Markdown?"

- **Yes, a file or folder** → get the path(s). Read them fully. Map what they cover onto the template
  sections (product, business rules, decisions, architecture, conventions, helpers, house rules,
  sweep patterns, security, environments). List the sections that are **missing or thin** and offer to
  fill them in Steps 3–4. Either reference the existing files from the config (preferred when they are
  maintained elsewhere) or copy them into `.code-review/knowledge-base.md` — ask which.
- **It lives elsewhere** (wiki, Notion, Confluence, Google Docs) → ask them to export it to Markdown
  into the repo, or to paste it. Don't fetch private URLs you can't authenticate to.
- **No** → build it with them from the template, starting from the docs found in Step 1.

## Step 3 — Fill the important gaps with the user

Ask for what can't be detected from code, section by section. Keep each answer short and checkable:
1. **Business rules and invariants** — "What must always be true? What would be a disaster if a
   change broke it?" Give each an ID (BR1, BR2, …) and where it's enforced.
2. **Decisions** — "Which decisions keep getting questioned, or would be easy to undo by accident?"
3. **Conventions** — confirm the validation, response, error, naming and schema-change conventions
   you detected in Step 1. Ask how schema changes ship (migration tool or raw DDL, who applies them).
4. **Hot tables / sensitive data** — big tables that need bounded queries and online DDL; money and
   PII fields.
5. **Queues, crons and background jobs** — which queues serialise work (FIFO groups, single workers),
   the consumer error contract, DLQ/visibility/lock settings, and which cron jobs depend on others.
   Detect what you can from the scheduler config (`cron.yaml`, crontab, worker manifests) and confirm.
6. **Sensitive calls** — the functions that move money, change credentials or call payment providers,
   by name, so reviews apply the right checklist when other modules call them.
7. **Environments** — which environments are safe for local verification runs, which must never be
   touched, and what each shares with production (database, cache, queues). Default to "local only".

## Step 4 — Seed house rules from past reviews (optional, recommended)

House rules are what make the review sound like this team's reviewers. Offer to draft them from real
review comments:

1. Pull review comments from recent merged PRs (ask how many; 50–100 is a good start):
   ```bash
   gh pr list --state merged --limit 50 --json number --jq '.[].number' \
     | xargs -I{} gh api "repos/{owner}/{repo}/pulls/{}/comments" --paginate \
       --jq '.[] | {pr: {}, user: .user.login, path, body}'
   ```
   Also ask which reviewers' comments matter most, and whether to exclude bots.
2. Group the comments into recurring patterns. Keep a pattern only if it appears in at least two PRs
   or the user confirms it.
3. Write each as a house rule: ID (`H1`, …), severity, rule, Bad → Good, and — where the pattern is
   greppable — a sweep regex.
4. Show the draft to the user and keep only what they approve. Don't quote reviewers by name in the
   file unless the user asks you to.

## Step 5 — Data access (optional)

Explain why: the review can use read-only queries to size a risk (e.g. "how many rows have NULL in
this column?"). It is optional; the review works without it.

Ask how the agent should reach the database:
- **None** — skip.
- **An MCP server** already installed for the database → record its name.
- **A CLI** → record the command with credentials coming from an env var or a client config file
  path held in an env var (e.g. `mysql --defaults-extra-file="$REVIEW_DB_CNF"`). Never inline them.

Then:
- Strongly recommend a **read-only database user**, ideally on a replica. Record whether that is
  confirmed. If it is not read-only, record that the agent must still only run `SELECT`/`EXPLAIN`.
- With the user's permission, test the connection with a harmless query (`SELECT 1`). Don't run
  anything else during setup.

## Step 6 — Write the files

1. Create `.code-review/config.md` from `templates/config.md`, filled in.
2. Create `.code-review/knowledge-base.md` from `templates/knowledge-base.md` (or reference the
   existing knowledge base from the config). Remove template comments from sections you filled;
   leave `TODO:` markers in sections still empty.
3. Show the user the diff. Don't commit unless they ask; when they do, suggest committing both files
   so the whole team shares one setup.
4. If the repo has a `CLAUDE.md` / `AGENTS.md`, offer to add one line pointing reviews at
   `.code-review/config.md`.

## Step 7 — Check the setup

- Re-read both files and confirm: no secrets; every listed path exists; base branch exists on the
  remote; each sweep regex compiles and matches a sample line it is meant to catch
  (`echo '<sample>' | grep -E '<pattern>'`).
- If the knowledge base has a path map or catalog, check it: flag catch-all globs (`src/**`) that
  would pull large docs into every review, modules with no doc at all, and docs that are seeds,
  examples or drafts that don't match the real code. List these for the user to fix.
- Schema docs should name indexes and their columns, not just count them, so reviews can check
  query plans without database access.
- Report what is configured, what is still `TODO:`, and suggest a first trial: run
  `team-code-review` on a recently merged PR and compare its findings with the real review comments.
  Gaps found that way become new house rules.

## Updating later

Rerun this skill to add rules or decisions. When a review misses something a human reviewer caught,
add it as a house rule (and a sweep pattern if it's greppable) — that is how the knowledge base gets
better over time.
