# audit-codebase — Test Scenarios & Evidence

Manual verification scenarios. No automation — invoke the skill against the described input and compare against expected output. `/tmp/audit-codebase-fixture` (see the implementation plan, Task 2) is the primary fixture for AC-H1 through AC-H3.

## AC-H1: Happy path — full fixture audit, parallel dispatch, codegraph unavailable

**Input:** `/tmp/audit-codebase-fixture` (codegraph not initialized for this fixture — exercises the fallback path).

**Invocation:** `cd /tmp/audit-codebase-fixture && invoke audit-codebase`

**Expected output:**
- 5 modules discovered: `auth`, `billing`, `reports`, `app`, `notify`.
- Meta line states "Codegraph: unavailable, fallback used."
- Cross-module & dead-code section reads `Cross-module & dead-code findings: skipped — codegraph unavailable.` — **not** populated with findings (this lens needs codegraph to confirm a real duplication or a real zero-caller symbol; without it, both are skipped entirely rather than attempted with a weaker fallback).
- `auth` section: 2 findings — missing-docstring, and plaintext password logging (the latter merges the CLAUDE.md violation, the bug-scan flag, and the git-history "TEMP, never removed" evidence into one row, since they're the same underlying line — converging lenses corroborate a finding's confidence, they don't multiply it into separate rows).
- `reports` section: 2–3 findings — the cents-truncation bug, the docstring/behavior mismatch, and (legitimately, not scripted) a duplicate-currency-formatter finding, since `reports/invoice.py` imports and calls both `billing.format.format_money` and the buggy `format_amount` side by side — this is visible to the `reports` module subagent directly, distinct from the orchestrator-level cross-module pass (3a), which stays skipped.
- `notify` section: 6 findings — the docstring/behavior mismatch on `send_welcome_email` (lens 4), the mutable-default-argument pitfall in `queue_digest` (lens 7), the `format_money` reuse against `billing/format.py` (lens 8), the redundant-loop simplification and efficiency findings in `should_send_digest` (lenses 9–10, may appear as one merged row), and the hardcoded `"admin"` bandaid (lens 11).
- `notify` section also reports the removed `None`-check guard in `purge_old_sessions` (sharpened lens 3), citing the "refactor: simplify purge_old_sessions" commit.
- `billing` and `app` omitted from the body; "2 of 5 modules reviewed had no findings above the confidence threshold" (or similar) note present — `app`'s only standalone finding (a docstring-overstatement nitpick) lands in the appendix below 80 confidence; the real cross-module bug touching `app/signup.py` gets reported under `notify` instead (see Verified, below) rather than under `app`.
- Both `docs/audit-codebase/<timestamp>-full-repo.md` and `.html` exist with matching content.

**Verified:** run end-to-end against the extended fixture on 2026-06-19 — output at `/tmp/audit-codebase-fixture/docs/audit-codebase/2026-06-19-210151-full-repo.{md,html}`. Confirms 5 modules discovered, the `auth`/`reports` findings carried over from the original run, and all 6 of `notify`'s non-codegraph-gated findings (sharpened git-history, language-pitfall, reuse, simplification/efficiency, altitude). `billing` and `app` both stayed in the "no findings" count at the module-subagent level — but the real run surfaced something not scripted in the plan: `notify`'s and `app`'s subagents each independently found half of a genuine cross-module bug (`send_welcome_email` returns `None` on success; `handle_signup` checks `is True`), which the centralized scoring step (3c) connected into one Critical finding even though lens 6 (contract-consistency) correctly stayed skipped without codegraph — a different mechanism than lens 6 catching it, but the same outcome the spec hoped lens 6 alone would deliver, achieved here because the fresh-eyes scoring pass synthesized two already-surfaced facts. Also fixed a real gap this run exposed: module enumeration was picking up the skill's own `docs/audit-codebase/` output directory from the prior run as a phantom empty module — Step 1a now drops any module with zero recognized source files.

**Fail signals:**
- Dead-code or cross-module findings appear despite codegraph being unavailable (Step 1c/3a's skip condition not honored).
- Any of the 5 file-level fixture-designed findings (see Task 2's list, excluding dead-code and cross-module) is missing.
- Only one of the two output files exists.

## AC-E3b: codegraph available — dead-code and cross-module lenses run

**Input:** Same fixture, but with `.codegraph/` initialized for it first (`codegraph init -i` run against `/tmp/audit-codebase-fixture`) — not exercised as part of this implementation plan (the plan's Task 6 end-to-end run uses AC-H1, since codegraph in this session is scoped to the project workspace, not arbitrary paths). Documented here for whoever runs this skill against a codegraph-indexed repo.

**Invocation:** `cd /tmp/audit-codebase-fixture && invoke audit-codebase`

**Expected output:** Same 5 file-level findings as AC-H1, plus:
- Meta line states "Codegraph: available."
- Cross-module & dead-code section populated: `legacy_auth_check` (dead code, confirmed zero callers via `codegraph_callers`) and the `format_money`/`format_amount` duplication (confirmed via `codegraph_search`/`codegraph_explore`).
- `main` (billing/cli.py) never appears anywhere in the report — confirms the pre-filter excluded it before it ever reached a module subagent or the dead-code scan.

## AC-E4: Contract-consistency — codegraph available

**Input:** Same fixture, with `.codegraph/` initialized for it (`codegraph init -i` run against `/tmp/audit-codebase-fixture`) — not exercised as part of this implementation plan, for the same reason as AC-E3b (codegraph in this session is scoped to the project workspace, not arbitrary paths). Documented here for whoever runs this skill against a codegraph-indexed repo.

**Invocation:** `cd /tmp/audit-codebase-fixture && invoke audit-codebase`

**Expected output:** Same findings as the updated AC-H1, plus: the `notify` module's lens 6 finds that `handle_signup` (in `app/signup.py`, a different module) calls `send_welcome_email` and checks `sent is True` — but `send_welcome_email` returns `None` on success, so every successful signup is reported as `"email_failed"`. This finding requires `codegraph_callers` to discover `handle_signup` as a real caller of `send_welcome_email` across module boundaries; it cannot be found by a lens that only reads `notify/`'s own files.

## AC-H2: `--sequential` mode

**Input:** Same fixture as AC-H1.

**Invocation:** `cd /tmp/audit-codebase-fixture && invoke audit-codebase --sequential`

**Expected output:** Identical findings to AC-H1 — `--sequential` changes dispatch timing only, never lens behavior or results.

## AC-E1: Scoped to one subdirectory

**Input:** Same fixture (codegraph still unavailable for it).

**Invocation:** `cd /tmp/audit-codebase-fixture && invoke audit-codebase auth`

**Expected output:**
- Only the `auth` module is discovered and reviewed.
- `auth`'s findings (password logging, TEMP-debug-logging, missing docstring) appear.
- Cross-module & dead-code section still reads "skipped — codegraph unavailable" (codegraph availability, not scope, is what gates this section here).
- No mention of `billing`, `reports`, or `app`.

If this scenario is instead run against a codegraph-indexed fixture (see AC-E3b) scoped to `auth`, the dead-code section should additionally carry the partial-scope caveat: "Dead-code findings are scoped to `auth` — a symbol with no callers found here may still be called from outside this subtree."

## AC-E2: No source files found

**Input:** An empty directory, or one containing only a `README.md`.

**Invocation:** `invoke audit-codebase` against that directory.

**Expected output:** Halts with `ERROR: No recognizable source files found in <path>. Point me at a codebase directory to audit. Stopping.` No `docs/audit-codebase/` directory created.
