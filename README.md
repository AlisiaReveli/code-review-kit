# team-code-review

An AI code-review skill that reviews pull requests the way your team's senior reviewers do. The review
method is shared; what your team knows (business rules, decisions, conventions, recurring review
comments) comes from a knowledge base in your repo.

It works with Claude Code, Codex, Cursor and other agents that support `SKILL.md` skills.

## What you get

- **`team-code-review`**: reviews a PR or working diff in passes (correctness, data integrity,
  performance, design, reuse, validation, readability, tests, your house rules). It runs a security pass
  on every review, sweeps the whole diff for repeat instances, and reports evidenced findings by severity.
- **`team-code-review-setup`**: one-time setup per repo. It finds your existing docs, asks for your
  knowledge base, can draft house rules from your past PR review comments, and can configure optional
  read-only DB access. It writes `.code-review/config.md` and `.code-review/knowledge-base.md`.

## Install

**Claude Code**

```
/plugin marketplace add AlisiaReveli/code-review-kit
/plugin install team-code-review@code-review-kit
```

**Codex, Cursor and other agents** (via [skills.sh](https://skills.sh))

```bash
npx skills add AlisiaReveli/code-review-kit
```

## Set up a repo

In the repo, ask your agent to "set up code review", or run the `team-code-review-setup` skill. Have
these ready:

1. **Your knowledge base** as Markdown, if you have one: business logic, decisions (ADRs), conventions.
   If you don't have one, setup builds it with you from `templates/knowledge-base.md`.
2. **Who your reviewers are** if you want house rules drafted from their past PR comments (needs `gh`).
3. **Database access (optional)**: a read-only DB MCP server, or a CLI command that reads its credentials
   from an environment variable. Use a read-only user. Secrets never go into the config.
4. **Which environments are off-limits** for verification runs. The default is local only.

Commit `.code-review/` so your whole team shares one setup.

## Use it

Ask "review this PR", "review my diff before I push", or "review PR 123". The report has a verdict,
Blockers / Should-fix / Nits with `file:line` and fixes, a security section (confirmed / needs
validation / rejected), and the production impact.

## Make it better over time

When a human reviewer catches something the skill missed, add it to the knowledge base as a house rule
(and a sweep pattern if it can be grepped), or rerun setup.

## License

MIT — see [LICENSE](LICENSE).
