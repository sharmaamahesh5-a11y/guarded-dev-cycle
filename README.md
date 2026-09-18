# Guarded development cycle

A skill for AI coding agents that adds hard gates for security, code quality and token cost to every task, from reading the ticket to handing back the pull request.

It works with Claude Code, Cursor, Codex, Gemini CLI and any other agent that reads `SKILL.md` files. One file, about 100 lines, loaded on every task.

## The problem it solves

Left to itself, a coding agent takes the shortest path. It reads more than it needs and then reads it again. It writes a change that passes its own tests and never asks what else that change touches. It fixes a silent default by making it loud in the same PR, and takes production down for the callers nobody knew about. It marks a path "fixed against the docs" that has never run. It writes `TODO` and moves on.

None of this shows up in a single PR. It shows up three weeks later, when three separately correct changes stack on one user path and the product does nothing when the user clicks send.

This skill exists because that happened to us. Every rule in it is a rule we added after a specific failure, and the file says why each rule is there.

## What it does

The skill splits the agent's job into seven parts and puts a rule on each.

**Before touching code.** Read what the task names, in order, before any search. Confirm claims before depending on them; if one is false, stop and say so. Plan one PR per stop point. Pick the model by the change (docs on the smallest model, security and schema on the strongest) and keep it for the whole PR.

**Reading.** Read by line range, never whole large files. Never re-read. Search tests, do not read them wholesale. Comments say why, not what. Deferred work is written as `DEFER(owner, trigger)` with a checkable trigger, and a bare `TODO` does not merge.

**Security, by what the change adds.** A checklist keyed to the kind of change rather than a generic list: a new table needs scoping and a deletion step; a new route needs a permission check and identity from the request; a background path declares identity at the entry; a provider call goes through the gateway with redacted logs; a webhook has a per-tenant secret; a setting is catalogued and never read inside an identity guard; anything that sends has a human gate; a tightened default is instrumented in one PR and flipped in the next.

**Quality.** Failing test first. Cleanup PRs change no behaviour. Every deletion carries the search proving zero callers. Every regression test names the bug it would have caught.

**Four hard gates.** The full suite green on every tier; every CI security job green on the commit; an independent review pass with no unresolved high finding; no marker without a `DEFER`. Everything else is expected and listed if skipped, but only these four block.

**The PR body.** Evidence per change, search proofs, test output, review findings, skipped items with reasons, line counts for cleanups, the session's token cost, and what the next stop is.

**Never.** No autonomous multi-task builds. No instrument-and-flip in one PR. No touching the deploy branch or production settings. No rewriting history. No new tables or dependencies the task did not name. No "fixed" for a path that has not run.

## What you get

Fewer regressions reaching production, because the security checklist runs on every change type and the four gates cannot be talked around. PRs a human can review in minutes, because the body carries the evidence instead of the reviewer going to find it. A token cost per task you can see and compare, because it is written in every PR. And one PR at a time, so a wrong assumption is caught at the first stop instead of buried under the third.

## Install

Claude Code, project-level (recommended, so it ships with the repo):

```
mkdir -p .claude/skills/guarded-dev-cycle
curl -o .claude/skills/guarded-dev-cycle/SKILL.md \
  https://raw.githubusercontent.com/kreoflow/guarded-dev-cycle/main/skills/guarded-dev-cycle/SKILL.md
```

Claude Code, user-level (every repo on your machine): same file into `~/.claude/skills/guarded-dev-cycle/SKILL.md`.

Any agent that supports the skills CLI:

```
npx skills add kreoflow/guarded-dev-cycle
```

Cursor: copy `skills/guarded-dev-cycle/SKILL.md` into `.cursor/skills/`. Codex, Gemini CLI, Windsurf and others: their equivalent skills or rules directory.

The skill's description says it loads for any change to a codebase with a database, external providers or CI. If you want it only on request, change the description to a slash command name.

## Adapting it to your project

The skill is written in roles, not file names: "the data layer", "the identity module", "the provider gateway", "the schema", "the security workflow". It works as it is. It works better if your project's conventions file (CLAUDE.md, AGENTS.md, or similar) names the actual files for each role, because the skill tells the agent to read that file and CI's own definition of green before this one.

Three things are worth adding to your conventions file if they are not there:

1. The command that runs the full suite, and whether there is a second tier (a real database, a build step) the agent must also run.
2. The names of your CI security jobs, so the agent can check each one on the commit.
3. Whether you have a review skill or a second-agent review step, and what "high severity" means in it.

If your project has no deletion or retention step for personal data, no per-tenant webhook secrets, or no gateway for provider calls, the matching checklist lines will be skipped with a reason in every PR body. That is the skill telling you what to build next.

## The cycle, step by step

1. **Task arrives.** It names files and line ranges. If it does not, the agent's first job is to produce that list and show it, not to start editing.
2. **Read.** The named files by range, in order. Confirm any claim the work depends on.
3. **Plan.** One PR per stop point, with branch names. Model chosen by the change.
4. **Build.** Failing test, then code. Security checklist lines applied as the change adds things. No deferral without a trigger.
5. **Gate.** Full suite on every tier. CI security jobs on the commit. Independent review of the final diff. Ledger clean.
6. **Hand back.** The PR body in the shape section 6 describes. See `templates/pr-body.md`.
7. **Stop.** Wait for review. The next stop starts from the merged state, not from a stale branch.

## Why these choices

Why only four hard gates: a gate the agent can satisfy by writing "n/a" is not a gate. Suites, security jobs, an independent review and the deferral rule are the four that a PR body cannot fake, so those block. The checklist is expected, visible and skippable with a reason, which is how you learn what your project is missing.

Why instrument, then flip: a silent default has callers nobody knows about, by definition. Logging who hits it for a week in production is the only inventory that is complete. Flipping in the same PR turns the inventory into an outage.

Why one PR per stop: an agent that runs three stops in one pass carries a wrong assumption from the first through the other two, and the reviewer sees only the sum. Stopping costs one round trip and buys a place to catch it.

Why the token cost in every PR: you cannot tune what you cannot see. Two tasks of similar size with a threefold cost difference is a reading-rules problem, and the number is what makes it visible.

## Templates

`templates/pr-body.md` is the PR body shape. `templates/task.md` is the task shape that makes step 1 cheap: evidence, fix by PR, acceptance with the bug each assertion catches, out of scope, and the read list for the agent.

## Origin

Built by the team at Kreoflow while shipping a B2B product with a two-person team and an AI agent writing most of the code. The rules came from incidents, one at a time; the skill is the part that transferred.

MIT licence.
