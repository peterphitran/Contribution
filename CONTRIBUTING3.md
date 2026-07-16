# Contribution [#949]: harden polly test count reconciliation

**Contribution Number:** 3  
**Student:** Peter Tran  
**Issue:** https://github.com/omnigent-ai/omnigent/issues/949  
**Pull Request:** https://github.com/omnigent-ai/omnigent/pull/2140  
**Status:** Merged

---

## Why I Chose This Issue

I chose Omnigent issue #949 ("[Bug] Orchestrator-side measurement error") because it sits at the intersection of testing discipline and multi-agent orchestration. Omnigent is an AI agent framework and meta-harness where one orchestrator coordinates many coding sub-agents (Claude Code, Codex, Cursor, Pi, and others). In this issue the `polly` orchestrator had recorded that its Codex worker "falsely reported test counts" — but an independent read-only investigation showed the worker was accurate and the *orchestrator's own measurement* was wrong. A bug where a correct agent is mislabeled as dishonest is exactly the kind of trust-eroding failure worth fixing carefully.

I also chose it because the root cause is a subtle testing gotcha I wanted to understand deeply: `grep -c 'def test_'` counts test *functions*, but pytest counts collected *cases*, and `@pytest.mark.parametrize` expands one function into many cases. Working this issue meant learning how Omnigent structures agent prompts as behavioural contracts, how the `polly` example bundle is wired, and how the project treats prompt text as a testable surface — all valuable for understanding large-scale agent tooling.

---

## Understanding the Issue

### Problem Description

The `polly` orchestrator compared a worker's reported pytest total against a ground-truth number it computed with `grep -c 'def test_'` on a single new test file. When those numbers disagreed, it recorded the worker as having a "miscount", "over-report", or "fabrication" in `.polly/registry.json` and in its handoff. In the reported case the worker had actually run a *multi-file* gate and reported the correct collected total (e.g. 11, 66), while the orchestrator's "actual" figure (5, 7) was a `def test_` count of just one file. The accurate report was frozen into the registry as a lie.

### Expected Behavior

Before recording any test-count discrepancy, the orchestrator should establish ground truth against the *same command, same file set, and same commit* the worker reported — using `pytest --collect-only -q`, which counts real collected cases, rather than a source-function count. Workers should report the exact command and file set they ran and distinguish collected cases from test functions, so reconciliation is apples-to-apples. An accurate report must never be auto-labeled as dishonesty.

### Current Behavior

`polly` used `grep -c 'def test_'` on a single file as its correctness oracle. This is structurally guaranteed to under-count: it cannot see parametrized case expansion (off by 8 in one documented case) and it was applied to one file rather than the multi-file set the worker ran. The two artifacts combined ("claimed 11/47/66; actual 5/6/7") were labeled fabrication — a false defect that risked propagating into a bug report against the Codex harness and into future agent-trust heuristics.

### Affected Components

- The `polly` orchestrator prompt (`examples/polly/config.yaml`) — where test-count reconciliation is instructed.
- The cross-review workflow skill (`examples/polly/skills/cross-review/SKILL.md`) — where deterministic gates are run and reconciled.
- All six worker agent prompts (`examples/polly/agents/{claude_code,codex,cursor,hermes,opencode,pi}/config.yaml`) — how workers report test results.
- The polly example regression suite (`tests/e2e/omnigent/test_example_polly.py`) — guards the guidance from silently disappearing.

---

## Reproduction Process

### Environment Setup

Omnigent is a `uv`-managed Python 3.12+ project. I worked on Windows using WSL on a WSL-native checkout (the repo has tracked symlinks that break when git-for-Windows touches them, and pytest is far faster off the `/mnt/c` drive). Setup was `uv sync --extra all --extra dev`, then running tests with `uv run python -m pytest`. The one snag worth noting: full-repo `ruff`/`pre-commit`/web checks fail in my local WSL environment for unrelated reasons — the web tooling needs Node 22+ and I had Node v18.19.1, and some notebook fixtures under `web/` trip ruff. I validated the touched Python file in isolation and in a fresh WSL-native clone instead.

### Steps to Reproduce

1. Take a worker that runs a multi-file pytest gate and reports the collected total (e.g. `pytest tests/inner/test_ui.py tests/entities/test_conversation.py` → 27 passed).
2. Independently "verify" that total with `grep -c 'def test_'` on just the new file (`tests/inner/test_ui.py` → 7).
3. Observed result: the numbers disagree (27 vs 7), so a naive orchestrator flags the accurate worker as over-reporting — even though `pytest --collect-only -q` on the same two files collects 27 cases, exactly matching the worker.

### Reproduction Evidence

- **Issue investigation:** #949 documents four cases where every Codex-reported count reproduces exactly under `pytest --collect-only` at the worker's commit; the "discrepancy" was the orchestrator's single-file `grep` baseline.
- **My local reproduction:** `tests/inner/test_ui.py` has 7 `def test_` but collects 10 cases; the two-file gate has 12 `def test_` but collects 27 cases — the parametrized/multi-file undercount in miniature.
- **My findings:** the defect is entirely orchestrator-side measurement methodology, not worker dishonesty; the fix belongs in the prompt/skill contracts, not in runtime code.

---

## Solution Approach

### Analysis

The false "fabrication" signal came from three compounding measurement artifacts on the orchestrator side: (1) comparing a multi-file gate total against a single-file baseline, (2) using `grep -c 'def test_'` (functions) instead of collected cases, which misses `@pytest.mark.parametrize` expansion, and (3) freezing that apples-to-oranges comparison into `.polly/registry.json` labeled as a worker defect. No runtime code was wrong — the bug lived in how `polly`'s prompts told it to measure "ground truth."

### Proposed Solution

Harden the reconciliation contract in prose across every layer that participates:

- Teach the `polly` orchestrator to reconcile counts only against the same command, file set, and commit the worker reported, using `pytest --collect-only -q`, and to never use `grep -c 'def test_'` as a correctness oracle or record miscount/over-report/fabrication without re-collecting the same gate.
- Centralize the `--collect-only` ground-truth step in the cross-review skill.
- Have each worker report the exact command/file set and distinguish collected cases from test functions.
- Add a regression test so this guidance can't silently disappear.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** `polly` mislabels accurate multi-file/parametrized pytest reports as fabrication because it measures ground truth with single-file `grep -c 'def test_'` instead of `pytest --collect-only` on the worker's actual gate.

**Match:** The polly bundle already expresses behaviour as prompt contracts and already has a structural test (`tests/e2e/omnigent/test_example_polly.py`) that asserts guidance strings are present. I followed that existing pattern rather than adding code.

**Plan:** Step-by-step implementation plan
1. Add a test-count ground-truth paragraph to `examples/polly/config.yaml` (orchestrator).
2. Add the `--collect-only` reconciliation step to `examples/polly/skills/cross-review/SKILL.md`.
3. Add a "report exact command/file set; distinguish cases from functions" line to all six worker configs.
4. Add `test_polly_test_count_ground_truth_guidance` asserting the contract survives in every file.

**Implement:** Branch `fix/polly-test-count-ground-truth` → PR #2140 (merge commit `6afe05f`).

**Review:** Prose-only change plus one test; no runtime paths touched. Confirmed each asserted substring appears verbatim in the diff, ran the polly test file green, and ran ruff on the touched test.

**Evaluate:** The new test fails if any layer's guidance is removed; manual reproduction confirms `--collect-only` matches worker totals where `grep` does not.

---

## Testing Strategy

The change is prompt/skill prose, so the "unit" test is a structural spec test that reads the bundle files and asserts the contract text is present.

### Unit Tests

- [x] `test_polly_test_count_ground_truth_guidance` asserts the orchestrator config and cross-review skill both contain the `pytest --collect-only -q <same files>` guidance, the `grep -c 'def test_'` warning, "collected cases", and "parametrized".
- [x] Same test asserts the orchestrator config carries the "same command, same file set, and same commit" rule and the miscount/over-report/fabrication guard.
- [x] Same test loops over all six worker configs and asserts each tells workers to report the exact command/file set and distinguish collected cases from test functions.

### Integration Tests

Not applicable — no components interact at runtime; the behaviour lives entirely in agent prompt contracts, covered by the structural spec test above. The existing suite still loads and validates the whole bundle (orchestrator brain, six sub-agents, skills, guardrails).

- [x] Full `tests/e2e/omnigent/test_example_polly.py` passes in the working checkout.
- [x] Full file passes in a fresh WSL-native clone.

### Manual Testing

Reproduced the mismatch shape by hand — `tests/inner/test_ui.py` shows 7 `def test_` but 10 collected cases, and the two-file gate shows 12 `def test_` but 27 collected cases — confirming `--collect-only` is the only correct oracle. Ran `ruff check` and `ruff format --check` on the touched test file (clean).

---

## Implementation Notes

### Investigation & Reproduction

Started from the #949 investigation, which had already refuted the "Codex lies about test counts" claim and pin-pointed the orchestrator's `grep`-based baseline. Reproduced the parametrized/multi-file undercount locally to be sure the fix targeted the real cause.

### Implementation & Review

Wrote the guidance into the orchestrator, the cross-review skill, and all six worker prompts, then added the regression test. During review the PR passed the reviewer SLA window (a second reviewer, @hzub, was added), then was approved and merged. The automated "Polly AI Review" flagged the test's exact-substring assertions as intentionally brittle — an accepted trade-off for a "guidance must not disappear" guard.

### Code Changes

- **Files modified:** `examples/polly/config.yaml`; `examples/polly/skills/cross-review/SKILL.md`; `examples/polly/agents/{claude_code,codex,cursor,hermes,opencode,pi}/config.yaml`; `tests/e2e/omnigent/test_example_polly.py` (9 files, +64 lines).
- **Key commits:** `6afe05f` — fix: harden polly test count reconciliation (#2140).
- **Approach decisions:** Fixed the contract in prose at every layer that measures or reports counts rather than adding runtime code, matching how the polly bundle already encodes behaviour; backed it with a structural test so the guidance is regression-guarded.

---

## Pull Request

**PR Link:** https://github.com/omnigent-ai/omnigent/pull/2140

**PR Description:** Closes #949. Teaches `polly` to reconcile pytest test counts against the same command, file set, and commit the worker reported; centralizes the `pytest --collect-only` ground-truth step in the orchestrator prompt and cross-review skill; has all six workers report their exact command/file set and distinguish collected cases from test functions; and adds a regression test that prevents the guidance from disappearing. Classified as a bug fix; no user-visible surface, so the Demo section is `N/A`.

**Maintainer Feedback:**
- The CI bot flagged the missing Demo section (`needs-demo`); resolved by confirming the change is a backend prompt/test with no visual surface (`N/A`).
- Review passed the SLA window, so @hzub was added as a second reviewer and the PR was reassigned.
- @hzub reviewed and approved with "looks good to me!", then merged into `omnigent-ai:main` (merge commit `6afe05f`, 52 of 53 checks passing).

**Status:** Merged

---

## Learnings & Reflections

### Technical Skills Gained

I learned the precise difference between pytest *collected cases* and `def test_` *functions*, and why `@pytest.mark.parametrize` makes source-function counting an unreliable oracle. I also learned how Omnigent encodes agent behaviour as prompt/skill contracts and treats that prose as a testable surface — writing a spec test that asserts guidance strings across an orchestrator, a skill, and six worker configs.

### Challenges Overcome

The hardest part was resisting the obvious-but-wrong framing ("the Codex worker lies") and instead fixing the measurement methodology. I also had to work around a local environment where full-repo lint/format/web checks fail for unrelated reasons (Node 18 vs required 22+, notebook fixtures), which I handled by validating the touched files in isolation and in a clean WSL-native clone.

### What I'd Do Differently Next Time

The regression test asserts exact substrings, so any future reword of the guidance will break it. Next time I'd consider asserting on a few resilient key phrases (or a normalized form) to reduce brittleness while still guarding the contract — though for this change the strict assertions were a deliberate, accepted trade-off.

---

## Resources Used

- Issue #949 investigation — the `pytest --collect-only` vs `grep -c 'def test_'` root-cause analysis and reconciliation tables: https://github.com/omnigent-ai/omnigent/issues/949
- pytest docs on parametrizing tests (collected cases vs test functions): https://docs.pytest.org/en/stable/how-to/parametrize.html
- Omnigent `CONTEXT.md` and `CLAUDE.md` — repo map, testing strategy, and the WSL-on-Windows dev workflow.
- The `examples/polly` bundle and `tests/e2e/omnigent/test_example_polly.py` — existing patterns for prompt-as-contract structural tests.
