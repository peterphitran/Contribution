# Contribution [#6575]: [document schema contracts usage with hive router]

**Contribution Number:** 1  
**Student:** Peter Tran  
**Issue:** https://github.com/graphql-hive/console/issues/6575<br>
**Status:** Phase II Complete

---

## Why I Chose This Issue

I'm interested in contributing to the GraphQL Hive issue focused on documenting schema contract usage with Hive Router because it provides an opportunity to improve the experience of developers adopting GraphQL Hive. Clear and complete documentation is essential for open-source projects, and ensuring that Hive Router users have the same guidance currently available for Apollo Router will make the platform more accessible and easier to use. I enjoy contributing to tools that help developers be more productive, and this issue has a direct impact on the usability of the project.

The contribution is documentation-focused; it requires understanding how GraphQL Hive and Hive Router interact, which will help me deepen my knowledge of GraphQL infrastructure and large-scale open-source projects. I want to learn more about GraphQL schema management, documentation best practices, and the workflow maintainers use to review and integrate community contributions.

---

## Understanding the Issue

### Problem Description

The Schema Contracts documentation currently provides instructions for serving contract schemas with Hive Gateway and Apollo Router, but it does not provide equivalent instructions for Hive Router. Users looking to consume contract schemas through Hive Router do not have a documented example on the Contract Schema page.

### Expected Behavior

The documentation should include instructions and a configuration example showing how to use a contract schema with Hive Router, similar to the existing guidance for Apollo Router.

### Current Behavior

The "Serving a Contract Schema" section only references Hive Gateway and Apollo Router. There are no documented instructions or examples for Hive Router.

### Affected Components

Schema Contracts documentation
"Serving a Contract Schema" section
Hive Router documentation/examples

---

## Reproduction Process

### Environment Setup

I reproduced the gap against the docs site, and separately stood up Hive Console to confirm the runtime behavior the docs describe.

- **Docs site:** `graphql-hive/docs` is a Bun + Turborepo monorepo (Node >= 24); `packages/documentation` is a TanStack Start + Vite + Fumadocs (MDX) app. I ran it inside WSL2 (Ubuntu) on Windows.
- **Challenges & fixes:**
  - With the repo on the Windows filesystem (`/mnt/c`) accessed through WSL, the first cold SSR import exceeded the nitro dev module-runner's hard-coded 60s RPC timeout, so every page returned a 503 (`Vite environment "nitro" is unavailable`). I worked around it by raising the runner timeout (`NITRO_VITE_RUNNER_TIMEOUT_MS`) in the postinstall patch and making the Vite cache directory configurable.
  - Used `nvm` to pin Node 24 for the toolchain.
- **Behavior validation (to de-risk the fix):** I ran the Hive Console `dev:hive` stack in WSL2 with Docker, created a contract, published a schema version, and confirmed the local CDN serves the contract supergraph at the documented `.../contracts/<contract_name>/supergraph` path.

### Steps to Reproduce

These steps reproduce the documentation gap deterministically against the base of this contribution (commit `8903d44`, the parent of my change):

1. Open the Schema Contracts page source at the base commit: https://github.com/graphql-hive/docs/blob/8903d44/packages/documentation/content/docs/schema-registry/contracts.mdx (or check out `main` at `8903d44` locally and open `packages/documentation/content/docs/schema-registry/contracts.mdx`).
2. Locate the `## Serving a Contract Schema` section (around line 235).
3. Read the section's only guidance — a single sentence: _"Point your Hive Gateway or Apollo Router instance to the supergraph of the contract schema:"_ — followed by the contract supergraph endpoint URL pattern.
4. **Observed result:** Hive Router is not mentioned anywhere in the section, and there are no per-integration configuration examples (no copy-paste setup for Hive Gateway, Hive Router, or Apollo Router).
5. **Cross-check:** Confirm Hive Router *is* documented for the regular (non-contract) supergraph at `/docs/router/supergraph`, which shows the omission is specific to the contracts page. (Equivalently, the published page https://the-guild.dev/graphql/hive/docs/schema-registry/contracts#serving-a-contract-schema referenced only Hive Gateway / Apollo Router, as noted in the issue.)

### Reproduction Evidence

- **Branch in my fork:** https://github.com/peterphitran/docs/tree/docs/serving-contract-schema
- **Commit showing the change:** https://github.com/peterphitran/docs/commit/260e148dd22f0b601cd5381a4fc6a94c8b24bfb8
- **My findings:** The "Serving a Contract Schema" section already existed but covered only Hive Gateway and Apollo Router in one prose sentence — no Hive Router guidance and no concrete examples. Hive Router can load a supergraph directly from the Hive CDN via `source: hive` (with `endpoint` + `key`) in `router.config.yaml`, so the contract supergraph URL plugs into the exact same mechanism. I validated locally that the contract supergraph is served by the CDN (HTTP 200) and that a gateway pointed at the contract endpoint serves the filtered schema (the excluded tagged field was removed).

---

## Solution Approach

### Analysis

This is a documentation completeness gap, not a code bug. The "Serving a Contract Schema" section was written before Hive Router was a first-class runtime, so it only told users to point "Hive Gateway or Apollo Router" at the contract supergraph. Hive Router can consume a supergraph directly from the Hive CDN, but that path was never documented for contracts, leaving Hive Router users without guidance (issue #6575).

### Proposed Solution

Add Hive Router to the "Serving a Contract Schema" section with a concrete, copy-paste configuration example, mirroring the existing Apollo Router guidance and the non-contract Hive Router supergraph docs. While there, split the prose into explicit per-integration subsections (Hive Gateway, Hive Router, Apollo Router) so each runtime has a self-contained example and a link to its canonical docs.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The contracts page documents serving a contract supergraph for Hive Gateway and Apollo Router but omits Hive Router; add equivalent Hive Router instructions (#6575).

**Match:** Reuse existing patterns — the non-contract Hive Router docs at `/docs/router/supergraph` already show loading from the CDN with `source: hive` / `endpoint` / `key`; the Apollo Router contract guidance uses `HIVE_CDN_ENDPOINT` / `HIVE_CDN_KEY`; the contracts page already uses a "point X at the contract supergraph endpoint" structure with a documented endpoint URL pattern.

**Plan:**
1. Update the section intro to include Hive Router alongside Hive Gateway and Apollo Router.
2. Add a `### Hive Router` subsection: a `router.config.yaml` using `source: hive` with the contract supergraph URL as `endpoint` and the CDN access key as `key`, plus the `docker run` command that mounts the config.
3. Split the existing guidance into `### Hive Gateway` and `### Apollo Router` subsections with copy-paste examples for parity.
4. Link each integration to its canonical docs page and keep image tags / config shapes consistent with those pages.

**Implement:** https://github.com/peterphitran/docs/tree/docs/serving-contract-schema (commit `260e148`).

**Review:** Confirm all three internal doc links resolve, image references match the canonical pages (gateway untagged, router `:latest`), and the MDX renders; `oxfmt` only formats JS/TS, so no MDX formatting pass is needed.

**Evaluate:** Validate the documented flow against a local Hive Console — create a contract, publish a schema version, confirm the CDN serves the contract supergraph (HTTP 200), and run Hive Gateway against the contract endpoint to confirm the excluded (tagged) field is filtered from the served schema.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
