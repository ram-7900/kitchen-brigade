---
name: kitchen-brigade
description: Route coding work across model tiers and effort levels by the cost of a mistake, write tests before code, and keep each change to the smallest clean, safe solution. Use for multi-step tasks, refactors, work spanning many files or a large codebase, anything that could be handed to subagents, and writing new features or modules. Do not use for a single trivial edit (typo, one rename, a one-line fix), for questions about code, or for lookups whose target is already known.
---

# Kitchen Brigade

Spend expensive reasoning only where a mistake is expensive. Spend cheap capacity on everything else. Ship the smallest clean, safe, secure solution that meets the spec.

## 1. The brigade in Claude Code

You are the executive chef. The main thread runs on the model and effort the user chose, and you cannot change them. Lower tiers are subagents launched with the Agent tool, which takes `model` and `effort` (`low` to `max`). This skill is the explicit instruction that allows you to set `effort` on those calls.

| Tier | Runs as | Agent call | Owns |
|---|---|---|---|
| Executive | main thread | none | decomposition, architecture, irreversible decisions, ambiguity, tests for work scoring 4 or more |
| Sous | subagent | `model: opus`, `effort: high` | reviews diffs scoring 4 or more; integrates across stations |
| Chef de partie | subagent | `model: sonnet`, `effort: medium` | one bounded station from an order ticket |
| Commis | `Explore` agent or subagent | `model: haiku`, `effort: low` | grep, file discovery, renames, formatting, boilerplate |

Use only the models the Agent tool schema lists. If it has no `model` parameter, route by effort and fresh context only, and tell the user. A `fork` inherits your model and ignores `model` and `effort`: it isolates context but saves no money. If the main thread is not the strongest listed model and the work scores 5 or more, give the design and tests to a subagent on the strongest model at `effort: high`.

## 2. Routing rubric

Score each axis from 0 to 2, then add the three scores.

| Axis | 0 | 1 | 2 |
|---|---|---|---|
| Blast radius | one private function or file | a module with several callers | users, data, money, security, public API |
| Reversibility | `git revert` fixes it | needs a coordinated rollback or redeploy | irreversible: data written or deleted, migration run, message sent, secret exposed |
| Detectability | compiler, types or existing tests catch it | only a review or manual run catches it | silent: wrong output, corrupt data, an open hole |

| Total | Route | Extra checks at the pass |
|---|---|---|
| 0-1 | Commis | run tests and lint |
| 2-3 | Chef de partie | tests, plus you skim the diff |
| 4 | Executive writes tests, chef de partie implements | sous reviews the diff |
| 5-6 | Executive writes tests and design; chef de partie or executive implements | mandatory sous review, security scans, user approval before anything irreversible runs |

Overrides: auth, authorization, payments, crypto, data migrations, deletes, parsing input at a trust boundary and public API contracts score at least 5. Detectability 2 never scores below 4. If torn between two rows, take the higher one. Decide in seconds.

Delegate only if the ticket would be shorter and clearer than the work, and the work needs more than about 5 tool calls or would pull several hundred lines into your context. Otherwise work inline. Run only independent stations in parallel.

## 3. The grilled cheese rules

A grilled cheese needs American cheese, not an aged wheel: fit the ingredient to the dish. Writing code is cheap. Reading it is not, because every line is something a person reviews, maintains and debugs.

1. Before writing, check for an existing solution in this order: the repo, the standard library, dependencies already installed. Reuse beats writing new code.
2. State the expected size of the diff in lines before you start.
3. A 20-line job gets about 20 lines. Pick a shorter approach only if it is equally clear, safe and secure.
4. Don't add classes, interfaces, config options, layers or extension points that nobody asked for. Three similar lines beat a premature helper.
5. Don't add a dependency for something a few lines can do.
6. No code golf. If a line needs a comment to decode it, rewrite it plainly.
7. If the diff grows past about 1.5 times the estimate, stop and look for a simpler design. If none exists, state why the size is needed.
8. Never cut validation, error handling or security the task needs in order to meet a line count. Size is a warning sign, not a rule.

## 4. Tests first

The tests are the spec. They are the ticket's definition of done and what the pass checks.

1. Test behaviour: the happy path, bad input, boundaries and failure paths. Security-relevant code also gets abuse cases: injection payloads, other users' IDs, missing or expired credentials, oversized input.
2. Run the tests and confirm they fail for the right reason. An assertion failure is right; an import or syntax error is not.
3. Write the minimum code that makes them pass.
4. Refactor while the tests stay green, then run the whole relevant suite.

- **Who writes the tests:** for work scoring 4 or more, the executive or sous writes them and a lower tier implements against them unchanged. For work scoring 3 or less, one tier writes both, tests first.
- **Changing tests:** a lower tier must never edit, delete, skip or weaken a test. If it needs to, it escalates.
- **Untestable code:** if code can't be tested (pure UI, an external system with no sandbox), say why and name the manual check to run instead. Never skip testing silently.
- **What not to test:** implementation details, such as private calls or mocks of the project's own code.

## 5. The order ticket

Every delegation uses this ticket. The subagent sees nothing except what the ticket contains.

```
TASK: <one sentence: the outcome, not the method>
STATION: <files/dirs it may edit; everything else is read-only>
INPUTS: <paths, symbols, file:line, facts already established>
CONSTRAINTS: <size budget ~N lines, no new deps, style, security needs>
DONE WHEN: `<test command>` passes; files in <test paths> unchanged; lint clean
ESCALATE IF: ambiguous spec, edit needed outside STATION, 2 failed attempts,
             a test looks wrong, the work is riskier than this ticket says
RETURN: <=15 lines: changes as file:line, test result, any ESCALATION. No file dumps.
```

If you can't fill in TASK and DONE WHEN concretely, the spec isn't ready. Resolve it yourself or ask the user; don't delegate it.

## 6. The pass

Nothing is committed or reported as done until all of these hold:
- You ran the DONE WHEN tests yourself and saw them pass. A subagent saying they pass is not enough.
- `git diff` on the test paths shows the implementer left them unchanged.
- Lint and type checks are clean.
- You read the diff against the grilled cheese rules: size against the estimate, and no abstractions or dependencies nobody asked for.
- For work scoring 4 or more: a sous reviewed the diff, and the security scans the project requires have run.
- For an irreversible step (running a migration, deploying, deleting data): the user approved it.

If the pass fails, send the work back once with the exact failure. If it fails a second time, move it up one tier.

## 7. Escalation (lower tier to higher)

A subagent can't ask questions while it works, so it escalates by stopping early and returning `ESCALATION:` with what it tried, the exact failure and the question. It escalates when:
- the spec is ambiguous, or two readings both fit
- the fix needs a file outside STATION
- two attempts have failed the same check
- a test looks wrong or needs changing
- the work turns out riskier than the ticket's score (for example, it touches auth)

When you receive an escalation, answer it and send a new ticket, or move the work up one tier. Each tier gets at most 2 attempts. If the executive also fails twice, stop and ask the user.

## 8. De-escalation (executive to lower tiers)

Once a decision is made and only execution is left, hand it down:
- design settled and tests written: implementation goes to a chef de partie
- searches or surveys across many files: commis (`Explore`)
- mechanical repetition (renames across files, formatting, boilerplate that follows a pattern): commis
- reviewing a diff that scores 3 or less: do it yourself from the diff; don't spawn a sous

Keep for yourself: anything still ambiguous, integration across stations, and the final pass.

## 9. Token discipline

- When you know the region you need, Grep for it and then Read only those lines with `offset` and `limit`.
- Keep these out of context: lockfiles, generated or minified files, whole logs (grep for the error), passing test output (read only failures) and raw subagent transcripts.
- If finding something needs more than 3 queries, or you don't know where it is, send a commis. You get back conclusions as `file:line`.
- Cap what a subagent returns using the ticket's RETURN line.
- At the end of each phase, write a summary of decisions, open items and `file:line` references, and work from it instead of re-reading files.
- Start a fresh subagent (not a fork, and not a continued agent) when you switch stations, after 2 failed attempts (the failed context biases the third), and for any review.
- Give a reviewer the diff and the tests, not your conclusion.
- Make at most 2 retries per check per tier. Never loop on the same error.
- Put facts already established in the ticket's INPUTS so nobody works them out again.

## 10. Anti-patterns

- The executive doing mechanical work: bulk renames, grep sweeps, boilerplate.
- A commis making design decisions: API shape, schema, error semantics.
- Delegating a task whose ticket is vaguer than the work itself.
- Spending more on delegation than the task costs, such as spawning an agent for a 3-line fix.
- Writing code before tests.
- Writing 200 lines where 20 would do.
- Adding abstractions, config or dependencies nobody asked for.
- Trusting a subagent's "tests pass" without running them yourself.
- A lower tier editing tests to make them pass.
- Briefing a reviewer with your hypothesis, which it will then tend to confirm.

## 11. When not to use this skill

Work inline, with no routing or tickets, on a one-file edit of up to about 20 lines that scores 3 or less, on questions or code reading, and on lookups whose target you already know.

Tests first and the grilled cheese rules still apply to small code changes. An edit that doesn't change behaviour (a typo, a comment, a rename the compiler checks) needs no new test; just run the existing tests.

Worked routing examples, a filled-in ticket, and the fallback for tools without model switching are in [reference.md](reference.md). Read it only when the rubric is unclear.
