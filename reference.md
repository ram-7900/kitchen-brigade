# Kitchen brigade reference

Load this only when the rubric in SKILL.md leaves a routing decision unclear.

## Worked routing examples

| Task | Blast | Rev | Detect | Total | Route |
|---|---|---|---|---|---|
| Rename a local variable | 0 | 0 | 0 | 0 | inline (too small to delegate) |
| Rename a symbol across 40 files | 1 | 0 | 0 | 1 | commis, then run the tests |
| Add a date-formatting helper | 0 | 0 | 0 | 0 | inline, tests first |
| New CSV import module with edge cases | 1 | 0 | 1 | 2 | chef de partie |
| Change caching on an internal service | 1 | 1 | 2 | 4 | executive tests, chef de partie implements, sous reviews |
| Change a public REST response shape | 2 | 1 | 1 | 4, raised to 5 by override | executive tests and design, sous reviews |
| Password reset flow | 2 | 1 | 2 | 5 | executive tests (with abuse cases), sous reviews, security scans |
| Column backfill migration on a prod table | 2 | 2 | 2 | 6 | executive design and tests, dry run on a copy, user approves the run |
| Refund amount rounding | 2 | 2 | 2 | 6 | same as above; money also triggers the override |

## Filled ticket example

```
TASK: Parse uploaded CSVs into Row objects, rejecting malformed lines with line numbers.
STATION: src/importer/csv_parse.py
INPUTS: Row dataclass at src/importer/models.py:12; existing reader helper
        src/util/io.py:40 open_text() handles encoding - reuse it.
CONSTRAINTS: stdlib csv only, no new deps; ~40 lines; max 10 MB enforced by caller;
             edit files only with the Edit/Write tools, never shell rewrites
DONE WHEN: `pytest tests/importer/test_csv_parse.py` passes; ruff clean;
           `python -m build` passes (no UI, so no width check)
ESCALATE IF: ambiguous spec, edit needed outside STATION, 2 failed attempts,
             a test looks wrong, the work is riskier than this ticket says
RETURN: <=15 lines: changes as file:line, test result, TEST EDITS (or none),
        any ESCALATION.
```

## When the tool has no model switching

- If the Agent tool has no `model` parameter, every tier runs on the same model. Routing still pays off in two ways:
  - **Effort:** if `effort` is available, set `low` for the commis, `medium` for the chef de partie and `high` for the sous.
  - **Context isolation:** a subagent's file reads and failed attempts stay out of the main context.
- If there are no subagents at all, the brigade becomes a set of phases in one thread: design, tests, implementation, pass. Summarise at each phase boundary and drop details you no longer need. Tell the user that model routing is unavailable.

## Notes on Claude Code agents

- `Explore` is read-only and finds code; it doesn't review it. Use it as the commis for searches only.
- `Plan` is read-only. Use it for a second opinion on a design, not for edits.
- `general-purpose` with `model` and `effort` set is the default chef de partie or sous.
- `isolation: "worktree"` gives a station its own copy of the repo, so parallel stations that edit nearby files don't collide.
- Rank the models by what the Agent schema lists. If it lists a model whose strength you don't know, don't guess: keep the executive on the main thread and the sous on `opus`.
