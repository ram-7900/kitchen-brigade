# kitchen-brigade

A Claude Code skill that routes coding work across model tiers and effort levels based on what a mistake would cost. It also has the agent write tests before code, and keep each change to the smallest clean, safe solution.

> **Built with Claude.** This skill was written with Claude (Claude Code), and later versions will be too. That's intentional: the project is a skill for AI coding agents, refined by running it on real work.

## The idea

The skill uses a kitchen brigade as its metaphor for routing work:

| Tier | Model / effort | Does |
|---|---|---|
| Executive chef | main thread, strongest model | architecture, irreversible decisions, tests for high-stakes work |
| Sous chef | `opus`, high effort | reviews diffs, integrates stations, runs the final quality gate ("the pass") |
| Chef de partie | `sonnet`, medium effort | one bounded module against a written spec (the "order ticket") |
| Commis | `haiku`, low effort | search, renames, formatting, boilerplate |

Work is routed by **blast radius × reversibility × detectability**, each scored 0–2. A badly diced onion gets caught at the pass, so cheap tiers can handle that kind of work. A broken sauce already on the guest's plate (auth, payments, migrations, anything that fails silently) goes to the top tier with a mandatory review.

**The grilled cheese principle:** a grilled cheese needs American cheese, not an aged wheel. Fit the solution to the problem. Writing code is cheap, but every line has to be read, reviewed and maintained. If 20 lines solve it, write 20 lines, as long as they stay clear and secure.

**Tests first:** tests are the spec. Lower tiers implement against tests written by a higher tier, and they are never allowed to edit those tests to make them pass.

## Files

- [`SKILL.md`](SKILL.md): the skill itself (body under 150 lines, because it costs tokens every time it loads)
- [`reference.md`](reference.md): worked routing examples, a filled-in order ticket, and what to do when the tool has no model switching

## Install

```bash
git clone https://github.com/ram-7900/kitchen-brigade ~/.claude/skills/kitchen-brigade
```

Claude Code picks the skill up automatically for multi-step tasks, refactors and new features. It skips trivial edits.

## How it evolves

Each version is a proof of concept (POC):
1. Use the current version to build real features in real projects.
2. Record cost, routing decisions, test order and diff size in `results/`. That folder is gitignored because it holds notes from private projects.
3. Change the skill based on what the results show, then tag the next version.

| Version | Notes |
|---|---|
| `v1` | Baseline: routing rubric, grilled cheese rules, tests first, order ticket, escalation rules, token discipline |
| `v2` | After the first POC (a 31-commit UI audit plus feature work on a production app). The pass now requires the production build and a check at narrow width. Independent review and security scans become a hard gate for risky work and every deploy. Parallel stations each get their own worktree. Subagents edit only with edit tools. Test edits that keep the assertion must be listed and approved. Guard tests enforce conventions. No delegating what three greps can do. |

## Comparing versions

Run the same four tasks on each version:
- **Trivial:** add a small pure function
- **Medium:** a bounded module with edge cases
- **High-stakes:** auth or a migration
- **Restraint check:** a task with a tempting over-engineered solution

Compare:
- how many agents were spawned, and at which tiers
- whether tests landed before code
- diff size
- whether an implementer tier touched any test files
