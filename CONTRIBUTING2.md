# Contribution [#280]: [add github-copilot-cli provider leaf]

**Contribution Number:** 2  
**Student:** Peter Tran  
**Issue:** https://github.com/orthogonalhq/nous-core/issues/280<br>
**Status:** Phase IV Complete

---

## Why I Chose This Issue

I picked this issue as my CodePath AI301 Capstone contribution because it is a labeled *good first issue* that sits at the edge of the Nous architecture — a Tier 1 integration where I only need to understand the external tool (the GitHub CLI) and the provider-leaf contract, not the Cortex or memory internals. That made it a realistic entry point for getting familiar with a large, layered TypeScript monorepo while still shipping something users can actually select as an inference backend.

It also lined up with what I wanted to learn: how Nous standardizes CLI-backed model providers through the `agent-cli` protocol, how generated provider catalogs and snapshot tests are kept in sync, and the process-safety concerns (argv exposure, environment inheritance, abort/timeout handling) that come with spawning child processes. Working directly with a maintainer through several review rounds was exactly the kind of real open-source iteration I was hoping to practice.

---

## Understanding the Issue

### Problem Description

Nous exposes model providers as **certified provider leaves** under `self/subcortex/providers/src/providers/<vendor>/`. There was no leaf for the GitHub Copilot CLI, so a user who has the GitHub CLI (`gh`) installed and authenticated locally could not route inference through it. The issue asks to implement a GitHub Copilot CLI provider using the *current* CLI provider-leaf contract.

An important nuance called out in the issue: it originally referenced the old `AgentAdapter` / `self/subcortex/coding-agents` path, but that path is superseded. The 2026-06-18 update moved all CLI-backed work onto the `feat/contributor-friendly-inference-provider-surface` branch, where CLI leaves use `ProviderDefinitionLeaf`, built-in provider IDs derive from `vendorKey` (leaves must **not** hand-author `wellKnownProviderId`), and every CLI provider must declare an `executionCapabilityProfile` — one of `one_shot_command`, `session_bound_command`, or `persistent_process`.

### Expected Behavior

A certified `github-copilot-cli` provider leaf that runs the CLI headlessly (non-interactive), delivers a rendered prompt to it, and returns plain text back through the standard provider pipeline — following the same shape as the existing `codex-cli` reference leaf, satisfying the schema ABI contracts, and appearing correctly in the generated provider catalogs.

### Current Behavior

No `github-copilot-cli` leaf existed. The GitHub CLI was not selectable as a provider, and there was no catalog entry, adapter, or tests for it.

### Affected Components

- New provider leaf: `self/subcortex/providers/src/providers/github-copilot-cli/` (`definition.ts`, `adapter.ts`, `provider.ts`, `index.ts`)
- Shared `agent-cli` protocol layer under `self/subcortex/providers/src/protocols/agent-cli/` (consumed, not modified)
- Generated provider catalogs and the `provider-codegen` snapshot
- Roster / public-export assertions and provider-pipeline integration tests

---

## Reproduction Process

Because this is a new adapter rather than a bug fix, "reproduction" here means (1) confirming the baseline — no GitHub Copilot CLI provider exists — and (2) the manual smoke test that surfaced the deprecation of the command surface I originally targeted.

### Environment Setup

- pnpm 10 + Node.js 22+, dependencies installed with `pnpm install`; all work scoped to the `@nous/subcortex-providers` package.
- Checked out the `feat/contributor-friendly-inference-provider-surface` integration branch (and re-synced it after the maintainer updated the CLI leaf contract on 2026-06-18).
- GitHub CLI installed and authenticated with `gh auth login` for manual smoke testing.
- Studied the `codex-cli` leaf as the correct reference (it uses the `agent-cli` protocol), rather than `anthropic/`, which is the *native API* reference and a different pattern.

### Steps to Reproduce

1. On the integration branch, inspect the generated provider catalog — there is no `github-copilot-cli` entry.
2. Attempt the originally-planned headless command surface, `gh copilot suggest --target shell`.
3. Observed result: as of 2025-10-25 the command no longer produces usable output — the `gh-copilot` extension was deprecated on 2025-09-25 in favor of the newer GitHub Copilot CLI, so building the leaf on that surface would ship a dead command path.

### Reproduction Evidence

- **Deprecation source:** GitHub changelog — *Upcoming deprecation of gh-copilot CLI extension* (2025-09-25); `gh copilot suggest --target shell` stopped emitting output by 2025-10-25.
- **My findings:** The viable, supported replacement is the GitHub Models extension (`gh models run <model>`), which supports non-interactive single-shot queries with plain-text stdout — verified against the extension's `cmd/run/run.go` single-shot + stdin behavior.
- **Branch/commits:** work delivered incrementally on PR [#396](https://github.com/orthogonalhq/nous-core/pull/396).

---

## Solution Approach

### Analysis

Nous already has a shared `agent-cli` protocol that handles process spawning and timeout enforcement for CLI-backed providers, so a new provider should be a *thin leaf* that supplies metadata plus a small process runner — not a bespoke integration. The root design decision was choosing the command surface: my initial target (`gh copilot suggest --target shell`) turned out to be deprecated, so the real fix was to retarget onto the supported `gh models run` GitHub Models surface, which changes the runtime semantics (general text model vs. shell assistant) but keeps the leaf on a command path we can support going forward.

### Proposed Solution

A certified `github-copilot-cli` provider leaf that delegates spawning/timeout to the `agent-cli` protocol layer and:

- Targets `gh models run <model>`, default model `openai/gpt-4o-mini`, installed via `gh extension install https://github.com/github/gh-models`.
- Declares `executionCapabilityProfile: 'session_bound_command'` (so it can't be assigned to Cortex Chat/System roles).
- Delivers the rendered prompt through **stdin**, not argv, so prompt content isn't exposed via process listings or argv-based logs.
- Honors the shared `AgentCliRunnerOptions.environmentPolicy` (`none` / `allowlist` / `explicit`) instead of inheriting the full parent environment.
- Escalates a timed-out child from `SIGTERM` to `SIGKILL` after a short grace period so a run always settles and never hangs the lane.

### Implementation Plan

Using the UMPIRE framework (adapted):

**Understand:** Add a CLI-backed model provider for GitHub Copilot using the current provider-leaf contract, headless and non-interactive, returning plain text.

**Match:** The correct reference is the `codex-cli` leaf (`agent-cli` protocol) — *not* `anthropic/`, which is the native-API shape. Reused `defineProviderAdapter`, `createAgentCliProviderAdapter`, and the shared runner/env/timeout surfaces.

**Plan:**
1. `definition.ts` — pure metadata: `vendorKey`, `providerClass: 'local_text'`, `executionCapabilityProfile`, `agentCli` headless config, install/auth notes, caveats.
2. `adapter.ts` — stateless translator: `renderGhCopilotPrompt` builds a plain-text prompt; `parseResponse` strips ANSI escape codes and never throws.
3. `provider.ts` — `createGhProcessRunner` (spawn + stdin + timeout escalation + env policy), `GitHubCopilotCliProvider implements IModelProvider`, and the `providerFactory`.
4. `index.ts` — barrel re-export so the generator scanner discovers the leaf.
5. Regenerate provider catalogs and update snapshot/roster tests.

**Implement:** Delivered across 14 commits on PR [#396](https://github.com/orthogonalhq/nous-core/pull/396), including the mid-review retarget commit `cab1b08`.

**Review:** Self-checked against `CONTRIBUTING.md` §Model Provider Adapters and the CLI provider guide — leaf shape, schema ABI, no hand-authored `wellKnownProviderId`, tests for definition/adapter/provider/runtime resolution.

**Evaluate:** `pnpm typecheck`, `pnpm lint`, scoped `pnpm test`, and `check:generated` all green; manual `gh models run` smoke test.

---

## Testing Strategy

Because this is a provider-adapter contribution, "testing" means the package's automated suite (unit + codegen + typecheck/lint) plus an end-to-end check of the runtime behavior the leaf describes.

### Automated Checks (CI)

- [x] **Unit — definition/adapter/provider:** the `provider-codegen` snapshot includes the `github-copilot-cli` catalog entry; `renderGhCopilotPrompt` composes system/context/tool sections; `parseResponse` strips ANSI escape sequences and never throws (returns empty text on bad input).
- [x] **Argv/stdin contract:** only the model id is passed in argv; the prompt is written to stdin.
- [x] **`environmentPolicy`:** honored for all three merge strategies (`none` / `allowlist` / `explicit`).
- [x] **Pre-start abort:** an already-aborted signal resolves without spawning `gh`.
- [x] **Timeout escalation:** when the child ignores `SIGTERM`, the runner force-kills with `SIGKILL` after the grace period and settles the run as timed-out.
- [x] **Generated catalogs + types:** `check:generated`, `typecheck`, and `lint` all clean.

### Integration / End-to-End Validation

- [x] **Provider pipeline:** the full definition → adapter → provider path is exercised with a fake runner.
- [x] **Public exports / roster:** assertions updated to include `github-copilot-cli`; **161 tests pass** for `@nous/subcortex-providers`.
- [x] **Merge-ref validation:** the maintainer confirmed the provider appears in System Status with the expected local-session setup guidance.

### Manual Testing

Smoke-tested `gh models run openai/gpt-4o-mini` to confirm the _runtime_ behavior the leaf targets: non-interactive single-shot output on stdout and prompt delivery through stdin (not argv). I cross-checked the invocation contract against the GitHub Models extension source (`cmd/run/run.go`) so the argv/stdin shape in the provider matches what `gh models run` actually expects.

---

## Implementation Notes

### Implementation Summary

The change adds a four-file provider leaf under `self/subcortex/providers/src/providers/github-copilot-cli/` (`definition.ts`, `adapter.ts`, `provider.ts`, `index.ts`), plus the regenerated provider catalogs and snapshot/roster test updates. The leaf is intentionally thin: it supplies metadata and a small process runner and delegates spawning and timeout enforcement to the shared `agent-cli` protocol, following the `codex-cli` reference leaf.

The key decision was choosing the command surface. I first built against `gh copilot suggest --target shell`, but manual smoke testing showed that surface was deprecated (2025-09-25) and no longer produced output, so I documented the deprecation and a `gh models run` pivot in the PR body and retargeted the leaf onto the supported GitHub Models surface (`defaultArgs: ['models', 'run']`, default model `openai/gpt-4o-mini`, installed via `github/gh-models`). The retarget changed the runtime semantics but kept the leaf on a command path the project can support going forward.

The remaining work came out of two maintainer review rounds and was mostly process safety. I moved the prompt from argv to stdin so prompt content isn't exposed through process listings or argv-based logs; replaced a blanket `Object.assign({}, process.env, …)` with a `buildProcessEnv` that respects the shared `environmentPolicy` (`none` / `allowlist` / `explicit`); guarded child stdin against `EPIPE` / `ERR_STREAM_DESTROYED` on early exit; narrowed and documented abort to pre-start only (matching the `codex-cli` snapshot semantics); added `SIGTERM` → `SIGKILL` timeout escalation so a timed-out child always settles instead of hanging the lane; and reverted a global loosening of `ProviderDefinitionSchema`, hydrating the derived provider id in the leaf test instead so the runtime schema stays strict.

### Code Changes

- **Files modified:** `definition.ts`, `adapter.ts`, `provider.ts`, and `index.ts` under `self/subcortex/providers/src/providers/github-copilot-cli/`, plus regenerated provider catalogs and snapshot/roster test updates.
- **Active branch:** `feat/contributor-friendly-inference-provider-surface` (merged into the upstream integration branch of the same name).
- **Key commits:** [`e7c5915`](https://github.com/orthogonalhq/nous-core/pull/396/commits/e7c59150f0ae890809bf80f307870ca8615d22ab) (definition) · [`9874877`](https://github.com/orthogonalhq/nous-core/pull/396/commits/98748773b0e24dad3eec4cc526a84d545de5520a) (adapter) · [`978026b`](https://github.com/orthogonalhq/nous-core/pull/396/commits/978026b0d21b1a93cf9cb576f4f0b09d8c45b396) (factory + barrel) · [`cab1b08`](https://github.com/orthogonalhq/nous-core/pull/396/commits/cab1b081399bc292c84d766e3f4cfaf2c6bd063d) (retarget to `gh models run`) · [`c26d8da`](https://github.com/orthogonalhq/nous-core/pull/396/commits/c26d8dae9b4241238217ded864d9c6f8ac22d51f) (timeout escalation to SIGKILL) · [`61df0d4`](https://github.com/orthogonalhq/nous-core/pull/396/commits/61df0d4f70cd4d60f264ebc7559358264e4200cc) (final test/comment cleanup).
- **Pull request:** https://github.com/orthogonalhq/nous-core/pull/396 (merged as `739d565`).
- **Approach decisions:**
  - Built a thin leaf over the shared `agent-cli` protocol instead of a bespoke integration, reusing `defineProviderAdapter` and `createAgentCliProviderAdapter`.
  - Delivered the prompt over stdin rather than argv for prompt privacy.
  - Kept the runtime `ProviderDefinitionSchema` strict and hydrated the derived id in tests instead of loosening the global schema.
  - Scoped abort to pre-start with a documented caveat, matching the `codex-cli` snapshot semantics.

---

## Pull Request

**PR Link:** https://github.com/orthogonalhq/nous-core/pull/396

**What I contributed:** A certified `github-copilot-cli` provider leaf (four files under `self/subcortex/providers/src/providers/github-copilot-cli/`) that delegates process spawning and timeout enforcement to the shared `agent-cli` protocol, targets the supported `gh models run` GitHub Models surface, delivers the prompt over stdin, respects the shared `environmentPolicy`, escalates timeouts `SIGTERM` → `SIGKILL`, and keeps the runtime provider-definition schema strict. Also regenerates the provider catalogs and updates snapshot/roster tests.

**PR Description:**

> **Summary**
> Adds a certified `github-copilot-cli` provider leaf — a thin leaf that delegates spawning and timeout enforcement to the shared `agent-cli` protocol, following the `codex-cli` reference.
>
> **Changes**
>
> - `definition.ts` — pure metadata: `vendorKey`, `executionCapabilityProfile: 'session_bound_command'`, and the `agentCli` headless config.
> - `adapter.ts` — stateless translator: renders the prompt to plain text, strips ANSI, and `parseResponse` never throws.
> - `provider.ts` — process runner (`createGhProcessRunner`), `GitHubCopilotCliProvider implements IModelProvider`, and `providerFactory`.
> - `index.ts` — barrel re-export for the generator scanner; plus regenerated catalogs and snapshot tests.
> - Targets `gh models run <model>` (default `openai/gpt-4o-mini`) after the legacy `gh copilot suggest --target shell` surface was deprecated; the prompt is delivered via stdin, the runner honors `environmentPolicy`, and timeouts escalate `SIGTERM` → `SIGKILL`.
>
> **Closes** #280

**Maintainer Feedback / Next Steps:**

- **Review 1 (@atlamors):** Requested changes — don't merge on the deprecated `gh copilot suggest --target shell` surface; retarget first. Plus three process-safety asks: keep prompt content off argv, respect the shared `environmentPolicy`, and either wire live aborts or narrow the support claim. → Addressed by retargeting to `gh models run`, moving the prompt to stdin, adding `buildProcessEnv`, and documenting pre-start abort.
- **Review 2 (@atlamors):** Two blockers — make timeout resolve even if the child ignores `SIGTERM`, and don't loosen `ProviderDefinitionSchema` globally. → Addressed by adding `SIGKILL` escalation and reverting the schema change (hydrate the derived id in the leaf test instead). Also refreshed the PR body off the old deprecation/pivot narrative.
- **Review 3 (@atlamors):** **Approved** and **merged** as an early-access CLI provider leaf (merge commit `739d565`). Merging auto-closed the originating issue [`#280`](https://github.com/orthogonalhq/nous-core/issues/280) as completed.

**Status:** Merged

---

## Learnings & Reflections

### Technical Skills Gained

- The Nous provider-leaf contract end to end: `ProviderDefinitionLeaf`, `vendorKey`-derived IDs, `executionCapabilityProfile`, the `agent-cli` protocol, and keeping generated catalogs/snapshots in sync.
- Safe child-process handling in Node.js — delivering input via stdin instead of argv, applying an environment allow/deny policy, guarding against `EPIPE` on early exit, and escalating `SIGTERM` → `SIGKILL` so timeouts always settle.
- Distinguishing a hydrated/runtime schema contract from leaf-authoring convenience, and why you shouldn't loosen a global schema to make one leaf easier to write.

### Challenges Overcome

- **A deprecated target mid-flight:** the command surface I built on was deprecated during development. I turned the smoke-test finding into a documented pivot proposal in the PR body, then retargeted to `gh models run` — which changed the runtime semantics and required re-reasoning about defaults and caveats.
- **Process-safety review:** translating abstract concerns (argv leakage, secret inheritance, hung lanes) into concrete, tested behavior across three review rounds.

### What I'd Do Differently Next Time

- Validate that the external command surface is *current and supported* before building on it, to avoid a mid-review retarget.
- Reach for stdin, environment policy, and timeout escalation from the first draft rather than adding them under review — they're table stakes for spawning processes, not follow-ups.
- Keep the PR description in lockstep with the code as it changes, so it never lags the actual behavior at review time.

---

## Resources Used

- Provider adapter docs — CLI provider guide, provider-leaf anatomy, schemas/ABI reference, and testing checklist (`docs.nue.orthg.nl/docs/development/provider-adapters/…`).
- `CONTRIBUTING.md` §Model Provider Adapters and the reference leaves at `self/subcortex/providers/src/providers/codex-cli/` (CLI / `agent-cli`) and `.../anthropic/` (native API).
- GitHub Models extension source (`github/gh-models`, `cmd/run/run.go`) for single-shot + stdin behavior.
- GitHub changelog — *Upcoming deprecation of gh-copilot CLI extension* (2025-09-25).
- Parent issue [#391](https://github.com/orthogonalhq/nous-core/issues/391) and related local-provider follow-up [#405](https://github.com/orthogonalhq/nous-core/issues/405).
