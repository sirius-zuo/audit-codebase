---
name: audit-codebase
description: Use when asked to audit an entire codebase, or a directory within it, for correctness bugs, CLAUDE.md compliance, dead code, and cross-module duplication — not a diff or PR, the whole tree.
argument-hint: '[--sequential] [--focus "<area>"] [path]'
allowed-tools: Bash(git:*), Bash(find:*), Bash(grep:*), Bash(date:*), Bash(mkdir:*), Agent, Write
---

# Audit Codebase

## Overview

Audits an entire codebase, or a directory within it, for correctness bugs, CLAUDE.md compliance, dead code, and cross-module duplication. Unlike `review-code` (which reviews a diff/PR), this skill has no baseline to diff against — it walks the whole target tree, dispatching one subagent per top-level module to review that module's files directly, then merges and scores the results centrally.

Follow these steps precisely, in order. Make a todo list first, with one item per step below.

## Step 0: Parse arguments and pre-flight check

Parse `$ARGUMENTS` for these optional flags, in any order, and strip them from the string before continuing:

- `--sequential` — dispatch module subagents one at a time (wait for each to fully return before starting the next) instead of the default parallel dispatch.
- `--focus "<area>"` — bias the lenses in Step 2 toward this area (e.g. `"security"`, `"performance"`). All lenses still run regardless — focus changes emphasis and depth, not which lenses execute.

Whatever remains after stripping flags is the target path argument. If empty, the target is the current repo root (resolved via `git rev-parse --show-toplevel`, or the current working directory if not a git repo). If given, resolve it to an absolute path and verify it exists and is a directory.

State the resolved target path, dispatch mode (parallel or sequential), and focus area (or "none") before continuing.

**Pre-flight check:**

1. Verify the target contains recognizable source files:
   ```bash
   find <target> -maxdepth 3 -type f \( -name '*.py' -o -name '*.ts' -o -name '*.tsx' -o -name '*.js' -o -name '*.go' -o -name '*.java' -o -name '*.rs' -o -name '*.rb' \) 2>/dev/null | grep -v -E 'node_modules|vendor|dist|build' | head -1
   ```
   If this returns nothing, halt: `ERROR: No recognizable source files found in <target>. Point me at a codebase directory to audit. Stopping.`
2. Check codegraph availability by calling `codegraph_status`. If it reports "not initialized" or the tool call errors, set `codegraph_available = false` and state: "codegraph unavailable — falling back to find/grep for module discovery; dead-code and cross-module-duplication lenses will be skipped for this run." Otherwise set `codegraph_available = true`.

## Step 1: Discover modules, map CLAUDE.md files, and find dead-code candidates

### 1a. Enumerate modules

A module is one top-level directory directly under the target path (not the target path itself). Files sitting loose at the target path's top level (not inside any subdirectory) form one additional module named `<root>`.

- If `codegraph_available`: call `codegraph_files` scoped to the target path to get the directory tree, then take its top-level directories.
- Otherwise:
  ```bash
  find <target> -maxdepth 1 -type d -not -path '<target>' | grep -v -E '(^|/)(\.git|node_modules|vendor|dist|build)$'
  ```
  plus a check for loose files at the top level for the `<root>` pseudo-module.

For each module, record its directory path and the list of source files under it (recursively). Drop any module with zero recognized source files (e.g. a `docs/` directory left behind by a prior run of this skill, or any other non-code directory) — there's nothing for a subagent to review there, so don't dispatch one.

### 1b. Map CLAUDE.md files per module

For each module: the root CLAUDE.md (if one exists at the target path or above, same discovery as review-code's Step 1) always applies, plus any CLAUDE.md file found within that module's own subtree.

### 1c. Global dead-code / unused-export scan (codegraph only)

Skip this entire substep if `codegraph_available` is false — note in the running notes: "dead-code scan skipped — codegraph unavailable."

For every exported/public symbol found under the target path: call `codegraph_callers` on it. If it returns zero callers, it's a candidate. Record each candidate as `{symbol, file, line, module}`.

### 1d. Pre-filter dead-code candidates

Before bucketing candidates into their modules for Step 2, drop any candidate matching these patterns — they're noise, not findings, and module subagents should not have to re-derive this:

- Named `main`, `Main`, `init`, or matches a CLI-entry-point naming convention for the project's language.
- Defined in a file matching a test/fixture naming convention (`test_*.py`, `*_test.go`, `*.spec.ts`, `*.test.ts`, or under a `test/`, `tests/`, `__tests__/`, or `fixtures/` directory).
- Re-exported from a barrel/index file (`__init__.py`, `index.ts`, `index.js`) whose only content is re-exporting other modules' symbols.
- Referenced as a string literal anywhere in a config file (`*.config.js`, `*.yaml`, `*.yml`, `*.json`) under the target path — `grep -rl "<symbol-name>"` over config files is sufficient; a string match disqualifies it as a candidate even without proving real usage.

Everything surviving the pre-filter is handed to its owning module's subagent in Step 2 as `dead_code_candidates`.

**Running state after Step 1** (carried into Step 2, one entry per module):
```
{
  module: "<directory name, or '<root>'>",
  files: ["<path>", ...],
  claude_md_paths: ["<path>", ...],
  dead_code_candidates: [{symbol, file, line}, ...]
}
```

## Step 2: Dispatch one subagent per module

For each module recorded in Step 1, dispatch a subagent with this task (substitute the module's actual `files`, `claude_md_paths`, and `dead_code_candidates` from Step 1's running state, and the `--focus` area from Step 0 if one was given):

> Review the files listed below as a self-contained code review task. You are not reviewing a diff — read each file's current content directly.
>
> Files: `<files>`
> Relevant CLAUDE.md files (read these yourself): `<claude_md_paths>`
> Dead-code candidates to verify (each may be a false positive — confirm with file-level context before reporting): `<dead_code_candidates>`
> Focus area (if any): `<focus, or "none">`
>
> Run these eleven passes, in order, over the listed files. Append every issue found to one running list, tagging each with which pass flagged it:
>
> 1. **CLAUDE.md compliance** — audit the files against the CLAUDE.md files listed above.
> 2. **Bug scan** — read the files directly and scan for obvious bugs. Avoid small issues and nitpicks; ignore likely false positives. Exclude: pre-existing issues, things that look like bugs but aren't, pedantic nitpicks, issues a linter/typechecker/compiler would catch, general code-quality issues not required by CLAUDE.md, and issues explicitly silenced in the code (e.g. a lint-ignore comment).
> 3. **Git blame/history context** — read the git blame and history of each file, and identify any bugs visible only in light of that history (e.g. a fix that was reverted, a TODO/TEMP marker that was never resolved). Also explicitly hunt for guards, validations, or error-handling that were present at some point in a file's history and were removed without the invariant they enforced being re-established elsewhere — flag the removal itself as the finding, citing the commit that dropped it.
> 4. **Code-comment compliance** — read code comments and docstrings in the files, and check the code complies with any guidance or claims made in those comments.
> 5. **Dead-code verification** — for each dead-code candidate listed above: confirm with file-level context whether it's genuinely unused (no dynamic dispatch, no reflection, no string-based lookup referencing it) or a false positive the earlier pre-filter missed. Only include candidates you confirm as genuinely dead in your findings, tagged with lens `dead_code`.
> 6. **Contract-consistency** (skip this pass entirely if `codegraph_available` is false — note "skipped — codegraph unavailable" instead of running it) — for each public function defined in these files, call `codegraph_callers` to find its real callers across the whole codebase, not just within this module. For each caller, check whether the function's current behavior (return shape, error/exception behavior, side effects) matches what that call site assumes. Flag a mismatch.
> 7. **Language-pitfall specialist** — scan these files for classic pitfalls of their language/framework (JS falsy-zero/`==` coercion/closure capture; Python mutable defaults/late-binding closures; Go nil-map writes/range-var capture; SQL injection; timezone/DST drift; float equality). Flag any instance present in the code.
> 8. **Reuse** — flag code in these files that reimplements something the codebase already has elsewhere; grep shared/utility modules and adjacent files, and name the existing helper that should be called instead.
> 9. **Simplification** — flag unnecessary complexity present in these files: redundant or derivable state, copy-paste with slight variation, deep nesting, dead code left behind. Name the simpler form that does the same job. Flag this regardless of when the complexity was added — do not restrict to anything "newly introduced."
> 10. **Efficiency** — flag wasted work present in these files: redundant computation or repeated I/O, independent operations run sequentially, blocking work on hot paths, or closures that keep an enclosing scope alive longer than needed. Name the cheaper alternative. Same scope as pass 9 — flag regardless of when it was added.
> 11. **Altitude** — check that code in these files is implemented at the right depth, not as a fragile bandaid; flag special cases layered on shared infrastructure as a sign the fix isn't deep enough, and suggest generalizing the underlying mechanism instead. Same scope as pass 9 — flag regardless of when it was added.
>
> Finally, write one or two sentences summarizing any notable structural pattern or convention this module uses for some concern (e.g. error handling, formatting, validation) — this will be compared against other modules' summaries later, so be specific about *how* the module does it, not just *that* it does it.
>
> Return: the merged findings list (each tagged with its lens), the confirmed dead-code list, and the structural summary.

### Dispatch mode

- **Default (parallel):** dispatch all module subagents at once, batching if there are more than 6 modules so no more than 6 run concurrently at any time.
- **`--sequential`** (if the flag was given in Step 0): dispatch one module subagent, wait for it to fully return, then dispatch the next. The task template above is identical either way — only dispatch timing changes.

Collect every module's returned result before continuing to Step 3.

## Step 3: Aggregate — cross-module pass, merge, score, filter

### 3a. Cross-module consistency pass

Skip this entire substep if `codegraph_available` is false — note in the running notes: "cross-module consistency pass skipped — codegraph unavailable." This pass depends on `codegraph_search`/`codegraph_explore` to confirm a suspected duplication is real and not just a name collision; without codegraph there's no reliable way to do that confirmation cheaply across the whole audit, so (per the Codegraph Dependency decision) it's skipped entirely rather than attempted with a weaker fallback.

Read every module's `structural_summary` from Step 2 side by side. Look for two or more modules describing the same underlying concern (error handling, formatting, validation, caching, retry logic, etc.) in different or inconsistent terms.

For each candidate pair/group spotted this way: **confirm before reporting** — never report from the summaries alone, since they're compressed observations, not proof. Confirm with a targeted `codegraph_search`/`codegraph_explore` call on the relevant symbol names to read both implementations side by side. Only keep the finding if, after confirming, the duplication is real and the implementations meaningfully diverge (not just stylistic difference).

Tag each confirmed cross-module finding with lens `cross_module`.

### 3b. Merge

Combine into one list: every module's `findings`, every module's `confirmed_dead_code` (tag lens `dead_code`, location from the candidate's `file`/`line`), and the cross-module findings from 3a.

### 3c. Score

Re-read the merged list with fresh eyes and assign two independent things per issue, using review-code's exact rubric:

**Confidence, 0–100:**
- **0** — not confident at all; false positive or pre-existing issue.
- **25** — somewhat confident; might be real, not verified; if stylistic, not explicitly called out in the relevant CLAUDE.md.
- **50** — moderately confident; verified real, but a nitpick or rare in practice.
- **75** — highly confident; double-checked, likely to be hit in practice, or directly called out in CLAUDE.md.
- **100** — absolutely certain; double-checked and confirmed, will happen frequently.

**Severity, independent of confidence:**
- **Critical** — breaks functionality, security/data-loss risk, or incorrect behavior on a common path.
- **Major** — a real bug, less common path or with a workaround.
- **Minor** — real but low-impact.

For any issue flagged via a CLAUDE.md, double-check the CLAUDE.md actually calls out that issue specifically before scoring it high.

For findings from lenses 8–11 (Reuse, Simplification, Efficiency, Altitude), frame the eventual Impact description (Step 5) as the concrete cost — what's duplicated, wasted, or harder to maintain — rather than a crash or failure scenario; these lenses don't have a crash to describe.

### 3d. Filter

Split into `main_table` (confidence ≥ 80) and `appendix` (confidence < 80, kept — not discarded).

## Step 4: Re-check eligibility

Re-run the pre-flight check's intent from Step 0: confirm the target path still exists and nothing material changed mid-audit. If codegraph reported pending/stale files for any file already reviewed (per the staleness-banner convention), re-read those specific files directly now rather than trusting stale graph data, and re-run the relevant lens(es) for them before continuing to Step 5.

## Step 5: Report the result

**Determine the output file path:**

1. Build a slug: the scoped target path (sanitized — `/` replaced with `-`), or `full-repo` if no path argument was given in Step 0.
2. Build a timestamp: `date +%Y-%m-%d-%H%M%S`.
3. File path stem: `docs/audit-codebase/<timestamp>-<slug>` (relative to the repo root being audited). Create the `docs/audit-codebase/` directory if it doesn't exist.

**Citation format** for the Location column: `path:line`, relative to the audited repo root.

**Markdown structure**, written to `<stem>.md`:

```markdown
### Codebase audit

Target: <target path or "repo root"> — Modules reviewed: <N> — Focus: <area, or "none"> — Codegraph: <available | unavailable, fallback used>

#### Cross-module & dead-code findings

| # | Severity | Location | Impact | Description |
|---|----------|----------|--------|--------------|
| 1 | Major | reports/format_helpers.py:1 | <consequence> | <description> |

#### <module-name>

| # | Severity | Location | Impact | Description |
|---|----------|----------|--------|--------------|
| 1 | Critical | auth/login.py:4 | <consequence> | <description> |
```

If `codegraph_available` was false for this run, replace the entire "Cross-module & dead-code findings" section with: `Cross-module & dead-code findings: skipped — codegraph unavailable.` (both lenses were skipped at Step 1c/3a, so there is nothing to report here, not an empty result).

Write one module section per module that has at least one surviving (`main_table`) finding, in the order modules were discovered in Step 1. Omit modules with zero surviving findings from the body, but add one summary line after all sections: `<M> of <N> modules reviewed had no findings above the confidence threshold.` If `main_table` is entirely empty, replace the whole findings section with: `No issues found above the confidence threshold across <N> modules reviewed.`

**Ordering:** within each module's findings table, sort by severity (Critical, then Major, then Minor), then by confidence descending within each severity. When two findings tie on both, list correctness-bug-style findings (lenses 1–7) before cleanup-style findings (lenses 8–11).

**Appendix** — one combined section (not per-module), same convention as review-code:

```markdown
<details>
<summary>Appendix: candidates below the confidence threshold (not reported above)</summary>

| # | Confidence | Severity | Module | Location | Rationale |
|---|------------|----------|--------|----------|-----------|
| A | 50 | Minor | billing | billing/format.py:2 | <why it's real but didn't clear the bar> |

</details>
```

Omit the appendix section entirely if `appendix` is empty.

**HTML rendering.** Render the same content into this template — replace every `{PLACEHOLDER}`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Codebase Audit — {TARGET}</title>
<style>
  body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif; background:#f5f5f7; color:#1d1d1f; margin:0; padding:2.5rem 3rem; line-height:1.6; }
  h1 { font-size:1.4rem; margin-bottom:0.25rem; }
  h2 { font-size:1.05rem; margin:1.75rem 0 0.5rem; color:#1d1d1f; }
  .meta { color:#6e6e73; font-size:0.875rem; margin-bottom:1.5rem; }
  table { width:100%; border-collapse:collapse; background:#fff; border-radius:8px; overflow:hidden; box-shadow:0 1px 3px rgba(0,0,0,0.08); margin-bottom:1.5rem; }
  th, td { padding:0.65rem 0.9rem; text-align:left; border-bottom:1px solid #e5e5ea; font-size:0.9rem; vertical-align:top; }
  th { background:#fafafa; font-weight:600; color:#3a3a3c; }
  tr:last-child td { border-bottom:none; }
  .sev-critical { color:#d70015; font-weight:600; }
  .sev-major { color:#bf5700; font-weight:600; }
  .sev-minor { color:#6e6e73; font-weight:600; }
  .clean-note { color:#6e6e73; font-size:0.875rem; margin-top:1rem; }
  details { margin-top:1.5rem; }
  summary { cursor:pointer; font-weight:600; color:#3a3a3c; }
</style>
</head>
<body>
  <h1>Codebase Audit</h1>
  <div class="meta">Target: {TARGET} — Modules reviewed: {N} — Focus: {FOCUS} — Codegraph: {CODEGRAPH_STATUS}</div>
  <h2>Cross-module &amp; dead-code findings</h2>
  {CROSS_MODULE_TABLE_OR_NONE}
  {MODULE_SECTIONS}
  <div class="clean-note">{CLEAN_MODULES_NOTE}</div>
  {APPENDIX}
</body>
</html>
```

`{MODULE_SECTIONS}` is one `<h2>{module name}</h2><table>...</table>` block per module with surviving findings, same row data as that module's markdown table section. Severity cells use the same `sev-critical`/`sev-major`/`sev-minor` classes as review-code's HTML output.

Write the markdown to `<stem>.md`, the HTML rendering to `<stem>.html`, then print the markdown content to the console.

## Edge cases

- **Module exceeds practical read size.** If a module subagent cannot read all of its assigned files within a reasonable context budget, it should report back a partial review and note in its structural summary: "module exceeds practical read size — reviewed partially." Step 5 must surface this verbatim in that module's section header (e.g. `#### billing (reviewed partially — exceeds practical read size)`) rather than silently presenting a partial review as complete. Recommend re-running `audit-codebase` scoped to a subdirectory of that module.
- **Partial-scope dead-code caveat.** When a `path` argument was given in Step 0 (not auditing the whole repo), prefix the dead-code section with: "Dead-code findings are scoped to `<path>` — a symbol with no callers found here may still be called from outside this subtree." Do not claim full confidence on dead-code findings when scope is partial.
- **No findings anywhere.** Already covered by Step 5's "No issues found above the confidence threshold across <N> modules reviewed" line.

## Notes

- Do not check build signal, or attempt to build or typecheck the app — assume CI handles this separately.
- No `--fix` flag — this skill is report-only.
- Make a todo list first, one item per step in this skill.
