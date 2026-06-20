# audit-codebase

A Claude Code skill that audits an entire codebase, or a directory within it, for correctness bugs, CLAUDE.md compliance, dead code, and cross-module duplication — not a diff or PR, the whole tree.

## Why this exists

`review-code` reviews a diff (PR, local branch, or commit range) — it has no mode for reviewing code that has no diff to anchor on. As coding agents write more and more of a codebase's code, each individual diff can look clean under diff-scoped review while compliance drift, dead code, and cross-module duplication accumulate unreviewed across the whole repo over time. `audit-codebase` is the whole-repo counterpart: it reuses `review-code`'s lens design, re-scoped from "this diff" to "this module's files," and adds lenses that only make sense at whole-codebase scope: dead-code/unused-export detection, cross-module duplication, and a codegraph-backed contract-consistency check (a function's real callers vs. what they assume — the no-diff equivalent of `review-code`'s cross-file tracer).

It does not replace `architect-codebase-review` (architecture/diagrams) or `design-pattern-review` (pattern applicability) — this skill is specifically about correctness bugs, compliance, dead code, and duplication. Its reuse/simplification/efficiency/altitude lenses overlap with the separate `simplify` skill (which also auto-fixes what it finds); `audit-codebase` reports them alongside everything else in one unified audit rather than requiring a second pass, and never auto-fixes them itself — same overlap-is-fine precedent as `review-code`.

## What it checks

Per module (one top-level directory at a time):
1. CLAUDE.md compliance
2. Bug scan (full read of the module's files — no diff to anchor on)
3. Git blame/history context (including guards/validations removed at some point and never restored)
4. Code-comment compliance
5. Dead-code verification (confirming candidates a global codegraph-driven scan flagged)
6. Contract-consistency (codegraph-gated) — a function's real callers vs. what they assume
7. Language-pitfall specialist
8. Reuse — code that reimplements an existing helper elsewhere in the codebase
9. Simplification — unnecessary complexity present in the module, regardless of when it was added
10. Efficiency — wasted work present in the module, regardless of when it was added
11. Altitude — fixes implemented as a bandaid instead of at the right depth

Across the whole audit, once:
12. Cross-module duplication (the same concern solved differently in two or more modules)

Lenses 6 and 12 are codegraph-gated — both are skipped (not attempted with a weaker fallback) when `.codegraph/` isn't initialized for the audited repo.

Each finding gets a confidence score (0–100, only ≥80 survives) and an independent severity (Critical/Major/Minor) — see `SKILL.md` for the full rubric, identical to `review-code`'s. Findings are ordered by severity then confidence, with correctness-bug-style lenses (1–7) breaking ties ahead of cleanup-style findings (8–11).

## Usage

```
/audit-codebase [--sequential] [--focus "<area>"] [path]
```

- `path` (optional) — directory to scope the audit to. Defaults to the repo root.
- `--sequential` — dispatch module subagents one at a time instead of the default parallel dispatch (capped at 6 concurrent).
- `--focus "<area>"` — bias all lenses toward an area (e.g. `"security"`) without skipping any of them.

There is no `--fix` flag — this skill is report-only.

## Output

A markdown report and an HTML rendering of it, both written to `docs/audit-codebase/<timestamp>-<slug>.md` / `.html`, where `<slug>` is the scoped path (sanitized) or `full-repo`. Findings are grouped by module, with a combined cross-module & dead-code section at the top and a combined appendix at the bottom for sub-80-confidence candidates.

## Installation

This directory is a self-contained skill (`SKILL.md` at its root), installed the same way as `review-code`:

```bash
ln -s /Users/jinzuo/projects/skills/audit-codebase ~/.agents/skills/audit-codebase
```
