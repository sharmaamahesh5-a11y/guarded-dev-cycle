---
name: guarded-dev-cycle
description: A development cycle for AI coding agents with hard gates for security, code quality and token cost. Use for any change to a codebase that has a database, external providers, or a CI pipeline. Loads on every task; pair it with the project's own conventions file.
---

# Guarded development cycle

Three things hold a codebase together when an agent writes most of the code: the project's conventions file (CLAUDE.md, AGENTS.md or equivalent) says what the code looks like; CI says what may merge; this skill says what the agent does between picking up a task and handing back a pull request. Where the three disagree, CI wins, then the conventions file, then this.

## 1. Before touching code

Read what the task names first, in the order it names them, before any search. A good task already did the search; a broad grep at the start spends tokens re-discovering it. If the task says a path exists or a function has a given signature, confirm it before writing anything that depends on it. If the claim is false, stop and say so; do not work around it.

Write the plan before the first edit: one pull request per stop point the task defines, each with its own branch. Two stops merged into one PR to save time is how a wrong assumption in the first stop gets buried under the second.

Choose the model by the change and keep it for the whole PR:

- Documentation, comments, configuration, copy: the smallest model, medium effort.
- A feature or fix inside one layer: the mid model, medium effort.
- Anything touching the data layer, identity or auth, permissions, data deletion or retention, the provider gateway, webhooks, the schema, or CI workflow files: the strongest model, high effort.

Downgrading mid-task to save cost on a security change is the one saving that costs the most.

## 2. Reading rules

Read by line range, never a whole file over 400 lines. Do not re-read a file already in context; quote the line you have. Do not read the test directory wholesale; search it for the module name and open one file. If a code graph or index exists, query it before a broad search, and know what it cannot see (dynamic dispatch, exec-loaded modules, call nesting).

Comments say why, name the task or PR that introduced the change, and never restate what the code does. Deferred work is written as `DEFER(<owner>, <trigger>): <what>` where the trigger is a checkable condition (a date, a count, "first real webhook from provider X"). A bare `TODO`, `FIXME`, or "not tested against production" does not merge. If a ledger script enforces this in CI, run it locally before handing back.

## 3. Security checklist, by what the change adds

Apply the matching lines and list any you skipped, with a reason, in the PR body.

- A table or column: row-level or tenant scoping in the schema; a deletion or anonymisation step for any personal data, or an explicit exemption with a reason; the schema drift test still passes if one exists; an audit entry if a person can change the value.
- A route or endpoint: a permission check as a declared dependency, not an inline `if`; identity taken from the request context; row scoping applied to anything that returns records; no internal ids, agent names or raw statuses in user-facing copy.
- A background or scheduled path: identity declared at the entry point with the narrowest scope that works; one short transaction per item, never held across a model call or an HTTP call; never rely on a connection obtained with no context.
- A call to an external provider: through the one gateway the project has; secrets and payloads redacted in logs; retry and circuit-breaker thresholds from the shared constant, not re-declared.
- A webhook: a per-tenant secret, a scoped lookup, no global fallback that would accept any tenant's payload.
- A setting or feature flag: seeded and catalogued where the project keeps them; never read inside an identity guard or on a request path without a transaction; if nobody will ever change it, it is a constant.
- Anything that sends to a real person: a human approval gate exists and is tested; the sender identity comes from the account that owns the recipient, not a default.
- A default being tightened: instrument in one PR (log or count who hits the old path), observe in production, flip in the next PR. Never both in one.

## 4. Quality rules

Failing test first, then the minimal code that passes it; documentation-only and configuration-only changes are the exception. A cleanup or refactor PR has zero behaviour change; if a test must change, the PR explains why the old assertion was wrong, or the change moves to its own PR. Every deletion carries the search proving zero callers, pasted in the PR body. Every user-facing string, colour and font goes through the project's one place for each; no ad hoc values in components. Every regression test names, in a comment, the change whose bug it would have caught.

## 5. Hard gates before a PR is handed back

All four block. Do not ask for a merge until each is true and shown in the PR body.

1. The full test suite green, including any second tier (a real database, a build step), run locally and shown.
2. Every security job in CI green on this commit: dependency audits, secret scanning on the diff, any project-specific scanners. If a job is label-gated, say so and apply the label.
3. An independent review pass on the final diff (a review skill or a second agent), with no unresolved high-severity finding; findings pasted with severity and what was done about each.
4. No new marker without a `DEFER(owner, trigger)`.

Everything in sections 3 and 4 is expected; a skipped item is listed with a one-line reason and does not block on its own.

## 6. PR body

The task name and the stop it covers. File and line evidence for each change. Search proofs for deletions. Test output for every tier. Review findings with severity. Skipped checklist items with reasons. Line counts before and after for cleanup PRs. The session's token cost. What the next stop is, or "task closed".

## 7. Never

Never run an autonomous multi-task build mode; one PR, then stop for review. Never flip a default in the PR that instruments it. Never change the deploy branch, the hosting configuration or a production setting; say what the human must change and when. Never rewrite history. Never add a table, object or dependency the task did not name without asking. Never paste a file's contents into a PR body. Never describe a path as fixed when it has not run; mark it `DEFER` with the trigger that will run it.
