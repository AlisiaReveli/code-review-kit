# team-code-review config

<!--
Written by team-code-review-setup. Commit this file so the whole team shares it.
It must never contain secrets: name environment variables or tools, never their values.
-->

## Repository
- **Name:** <repo>
- **Base branch:** <main | staging | develop>
- **Stack:** <e.g. Node/Express + MySQL>
- **Test command:** <e.g. npm test -- <path>>
- **Lint command:** <e.g. npm run lint>

## Knowledge base
<!-- Paths relative to the repo root, or absolute URLs the agent can read. Read in this order. -->
- `.code-review/knowledge-base.md`

## Extra review checklists
<!-- Other installed skills to apply when matching paths change. Optional. -->
| Paths | Skill |
|---|---|

## Data access (optional)
- **Mode:** none | mcp | cli
- **MCP server / tool:** <name of an installed, read-only database MCP server>
- **CLI:** <e.g. `mysql --defaults-extra-file="$REVIEW_DB_CNF"` — reference an env var, never inline credentials>
- **Credentials from:** <env var names only, e.g. REVIEW_DB_CNF>
- **Read-only confirmed:** <yes — user `x` has SELECT only | no>
- **Allowed use:** counts and read-only lookups to size a risk; never writes, never bulk exports of PII.

## Verification environments
- **Safe for local exploit/regression runs:** local
- **Never run exploits against:** staging, production
- **How to start the app locally:** <command, or "see README">
- **How to get a test token for a role:** <command/script, or "ask the user">

## PR conventions
- **Branch naming:**
- **PR description must include:**
