# Contribution [#6575]: [document schema contracts usage with hive router]

**Contribution Number:** 1  
**Student:** Peter Tran  
**Issue:** https://github.com/graphql-hive/console/issues/6575<br>
**Status:** Phase IV Complete

---

## Why I Chose This Issue

I chose this issue because it improves the experience of developers adopting GraphQL Hive. Good documentation matters for open-source projects, and giving Hive Router users the same guidance that Apollo Router users already have makes the platform easier to use.

The work is documentation-focused, so it helps me learn how GraphQL Hive and Hive Router work together, and how maintainers review and merge community contributions.

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
3. Read the section's only guidance, a single sentence (_"Point your Hive Gateway or Apollo Router instance to the supergraph of the contract schema:"_) followed by the contract supergraph endpoint URL pattern.
4. **Observed result:** Hive Router is not mentioned anywhere in the section, and there are no per-integration configuration examples (no copy-paste setup for Hive Gateway, Hive Router, or Apollo Router).
5. **Cross-check:** Confirm Hive Router *is* documented for the regular (non-contract) supergraph at `/docs/router/supergraph`, which shows the omission is specific to the contracts page. (Equivalently, the published page https://the-guild.dev/graphql/hive/docs/schema-registry/contracts#serving-a-contract-schema referenced only Hive Gateway / Apollo Router, as noted in the issue.)

### Reproduction Evidence

- **Branch in my fork:** https://github.com/peterphitran/docs/tree/docs/serving-contract-schema
- **Commit showing the change:** https://github.com/peterphitran/docs/commit/260e148dd22f0b601cd5381a4fc6a94c8b24bfb8
- **My findings:** The "Serving a Contract Schema" section already existed but covered only Hive Gateway and Apollo Router in one prose sentence, with no Hive Router guidance and no examples. Hive Router can load a supergraph directly from the Hive CDN via `source: hive` (with `endpoint` + `key`) in `router.config.yaml`, so the contract supergraph URL uses the exact same mechanism. I confirmed locally that the CDN serves the contract supergraph (HTTP 200) and that a gateway pointed at the contract endpoint serves the filtered schema (the excluded tagged field was removed).

---

## Solution Approach

### Analysis

This is a documentation completeness gap, not a code bug. The "Serving a Contract Schema" section was written before Hive Router was a first-class runtime, so it only told users to point "Hive Gateway or Apollo Router" at the contract supergraph. Hive Router can consume a supergraph directly from the Hive CDN, but that path was never documented for contracts, leaving Hive Router users without guidance (issue #6575).

### Proposed Solution

Add Hive Router to the "Serving a Contract Schema" section with a concrete, copy-paste configuration example, mirroring the existing Apollo Router guidance and the non-contract Hive Router supergraph docs. While there, split the prose into explicit per-integration subsections (Hive Gateway, Hive Router, Apollo Router) so each runtime has a self-contained example and a link to its canonical docs.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The contracts page documents serving a contract supergraph for Hive Gateway and Apollo Router but omits Hive Router; add equivalent Hive Router instructions (#6575).

**Match:** Reuse existing patterns. The non-contract Hive Router docs at `/docs/router/supergraph` already show loading from the CDN with `source: hive`, `endpoint`, and `key`. The Apollo Router contract guidance uses `HIVE_CDN_ENDPOINT` and `HIVE_CDN_KEY`. The contracts page already uses a "point X at the contract supergraph endpoint" structure with a documented endpoint URL pattern.

**Plan:**
1. Update the section intro to include Hive Router alongside Hive Gateway and Apollo Router.
2. Add a `### Hive Router` subsection: a `router.config.yaml` using `source: hive` with the contract supergraph URL as `endpoint` and the CDN access key as `key`, plus the `docker run` command that mounts the config.
3. Split the existing guidance into `### Hive Gateway` and `### Apollo Router` subsections with copy-paste examples for parity.
4. Link each integration to its canonical docs page and keep image tags / config shapes consistent with those pages.

**Implement:** https://github.com/peterphitran/docs/tree/docs/serving-contract-schema (commit `260e148`).

**Review:** Confirm all three internal doc links resolve, image references match the canonical pages (gateway untagged, router `:latest`), and the MDX renders; `oxfmt` only formats JS/TS, so no MDX formatting pass is needed.

**Evaluate:** Validate the documented flow against a local Hive Console: create a contract, publish a schema version, confirm the CDN serves the contract supergraph (HTTP 200), and run Hive Gateway against the contract endpoint to confirm the excluded (tagged) field is filtered from the served schema.

---

## Testing Strategy

Because this is a documentation contribution, "testing" means link/format/render validation plus an end-to-end check of the runtime behavior the docs describe.

### Automated Checks (CI)

- [x] **MDX link validation:** all three internal links I added (`/docs/gateway/supergraph-proxy-source`, `/docs/router/supergraph`, and `/docs/other-integrations/apollo-router`) resolve to existing pages. The repo runs a `validate-mdx-links` workflow on every `content/**/*.mdx` change to enforce this.
- [x] **Format check:** `oxfmt` only formats JS/TS, so the MDX edit cannot cause a formatting regression (`format:check` is unaffected).
- [x] **Page builds and renders:** the Fumadocs/MDX page compiles and renders the new subsections locally without errors.

### Integration / End-to-End Validation

- [x] **Local Hive Console:** stood up the `dev:hive` stack (Docker, WSL2), created a target and project, defined a `public-no-pii` contract that excludes fields tagged `pii`, and published a schema version.
- [x] **CDN serves the contract supergraph:** `GET .../contracts/public-no-pii/supergraph` returned **HTTP 200** with a valid supergraph SDL, the exact endpoint shape the new instructions tell users to point at.

### Manual Testing

To confirm the _runtime_ behavior (not just that the CDN responds), I ran Hive Gateway against both the full and the contract supergraph and compared the served schemas:

| Gateway              | Supergraph source        | `User.email` (tagged `pii`) |
| -------------------- | ------------------------ | --------------------------- |
| Base (port 4000)     | full supergraph          | **present**                 |
| Contract (port 4099) | `public-no-pii` endpoint | **filtered out**            |

All non-PII `User` fields remained in both. This proves that a gateway pointed at the contract supergraph endpoint serves the filtered schema, exactly as the new Hive Gateway, Hive Router, and Apollo Router instructions describe. I also reviewed the rendered MDX: the intro lists all three runtimes, and each `###` subsection shows a copy-paste example with the correct CDN endpoint pattern and a link to its canonical docs page.

---

## Implementation Notes

### Implementation Summary

The change is in a single file, `packages/documentation/content/docs/schema-registry/contracts.mdx` (+48 / -1). I rewrote the one-sentence intro of the `## Serving a Contract Schema` section to mention all three runtimes, then split the guidance into three self-contained, copy-paste subsections:

- **`### Hive Gateway`:** `docker run ... ghcr.io/graphql-hive/gateway supergraph "<contract endpoint>" --hive-cdn-key "<key>"`, linking to `/docs/gateway/supergraph-proxy-source`.
- **`### Hive Router`:** a `router.config.yaml` that loads the contract supergraph from the Hive CDN via `supergraph.source: hive` (`endpoint` = contract supergraph URL, `key` = CDN access key), plus the `docker run` that mounts the config. Links to `/docs/router/supergraph`.
- **`### Apollo Router`:** `docker run ... --env HIVE_CDN_ENDPOINT="<contract endpoint>" --env HIVE_CDN_KEY="<key>" ghcr.io/graphql-hive/apollo-router`, linking to `/docs/other-integrations/apollo-router`.

The key insight was that Hive Router already loads a supergraph from the Hive CDN with `source: hive`, so the contract supergraph URL uses the same mechanism as a regular supergraph. No new config shape was needed, just the contract-specific endpoint. The main friction was environmental: running the docs dev server from the Windows filesystem through WSL hit a nitro module-runner timeout, which I worked around so I could preview the rendered MDX locally.

### Code Changes

- **Files modified:** `packages/documentation/content/docs/schema-registry/contracts.mdx`, the only file changed in this contribution (+48 / -1).
- **Active branch:** https://github.com/peterphitran/docs/tree/docs/serving-contract-schema
- **Key commit:** [`260e148`](https://github.com/peterphitran/docs/commit/260e148dd22f0b601cd5381a4fc6a94c8b24bfb8), which adds Hive Router instructions and splits the section into per-runtime subsections.
- **Pull request:** https://github.com/graphql-hive/docs/pull/129
- **Approach decisions:**
  - Reused the existing `source: hive` CDN-loading pattern from the non-contract router docs instead of inventing a new Hive Router config for contracts.
  - Split the single prose sentence into three per-runtime subsections so each integration is self-contained and copy-paste-able, with a link to its canonical docs page.
  - Kept image references consistent with the canonical pages (Hive Gateway image untagged, Hive Router `:latest`, Apollo Router driven by `HIVE_CDN_ENDPOINT` and `HIVE_CDN_KEY`).

---

## Pull Request

**PR Link:** https://github.com/graphql-hive/docs/pull/129

**What I contributed:** Added Hive Router guidance to the "Serving a Contract Schema" section of the Schema Contracts docs and split the section into three self-contained, copy-paste subsections (Hive Gateway, Hive Router, and Apollo Router), each with a runnable example and a link to its canonical docs page. One file, `packages/documentation/content/docs/schema-registry/contracts.mdx` (+48 / -1).

**PR Description:**

> **Summary**
> The "Serving a Contract Schema" section only explained how to serve a contract supergraph with Hive Gateway and Apollo Router, leaving Hive Router users without guidance. This PR adds Hive Router and gives each runtime its own copy-paste example.
>
> **Changes**
>
> - Reworded the section intro to cover Hive Gateway, Hive Router, and Apollo Router.
> - Added `### Hive Gateway`, `### Hive Router`, and `### Apollo Router` subsections, each with a runnable example and a link to its canonical docs.
> - Hive Router loads the contract supergraph from the Hive CDN via `source: hive` (`endpoint` + `key`), the same mechanism already documented for a regular supergraph, just pointed at the contract endpoint.
>
> **Closes** graphql-hive/console#6575

**Maintainer Feedback / Next Steps:**

- Reviewed and **approved by `@n1ru4l`** with no requested changes.
- **Merged by `@dotansimha`** into `graphql-hive/docs:main`. The change is now live in the docs.
- Merging auto-closed the originating issue [`graphql-hive/console#6575`](https://github.com/graphql-hive/console/issues/6575) as completed. No further action required.

**Status:** Merged

---

## Learnings & Reflections

### Technical Skills Gained

- **GraphQL Hive schema contracts:** how `@tag`-based contracts produce a filtered contract supergraph, and how that supergraph is served separately from the full one.
- **Supergraph-from-CDN loading across runtimes:** Hive Router and Hive Gateway both load a supergraph from the Hive CDN (`source: hive` with `endpoint` + `key`), and Apollo Router does the equivalent via `HIVE_CDN_ENDPOINT` and `HIVE_CDN_KEY`. Recognizing that the contract endpoint uses the same mechanism was the core insight behind the fix.
- **Running Hive Console locally:** brought up the `dev:hive` stack with Docker under WSL2, created a target and project, defined a contract, and published a schema version end-to-end.
- **Verifying behavior, not just output:** introspected two running gateways and compared the schemas to prove a tagged (PII) field is actually filtered through the contract endpoint.
- **Docs tooling:** authoring Fumadocs/MDX in a Bun + Turborepo + TanStack Start/Vite monorepo, and the open-source workflow (fork, branch, commit, PR, CI link validation, then maintainer review and merge).

### Challenges Overcome

- **WSL `/mnt/c` dev-server timeout:** running the docs site from the Windows filesystem through WSL made the first cold SSR import exceed nitro's hard-coded 60s module-runner RPC timeout, so every page returned a 503. I traced it to the runner timeout and worked around it by raising `NITRO_VITE_RUNNER_TIMEOUT_MS` and making the Vite cache dir configurable, enough to preview the rendered MDX locally.
- **Port conflicts during validation:** the `dev:hive` stack already occupied the usual gateway ports (4000-4002), so my test gateway hit `EADDRINUSE`. I checked the listening ports and launched the contract gateway on a free port (4099) instead.
- **Proving the filter actually works:** rather than trust the docs, I stood up a `public-no-pii` contract and confirmed the base gateway served `User.email` while the contract gateway omitted it, turning "this should work" into a verified result.
- **Windows and WSL shell quoting:** PowerShell to WSL handoffs mangled quoted commit messages and piped grep patterns, so I switched to plain ASCII, quote-free commands to get clean commits.

### What I'd Do Differently Next Time

- **Run the docs dev server from the native Linux filesystem** (WSL home) instead of `/mnt/c` to avoid the nitro timeout entirely, rather than patching around it.
- **Keep the branch scoped from the start:** isolate the single doc change on its own branch immediately and keep the local dev-environment workarounds on a separate branch, so the working tree stays clean and the PR diff is minimal.
- **Build the verification harness earlier:** I confirmed the runtime filtering near the end; setting up the local contract gateway first would have let me write the examples with full confidence from the outset.

---

## Resources Used

- **Issue I worked on:** [graphql-hive/console#6575](https://github.com/graphql-hive/console/issues/6575)
- **Docs repo (contributing setup & monorepo layout):** [graphql-hive/docs](https://github.com/graphql-hive/docs)
- **Schema Contracts page (the page I edited):** https://the-guild.dev/graphql/hive/docs/schema-registry/contracts
- **Hive Router Supergraph Source** (the `source: hive` pattern I reused): https://the-guild.dev/graphql/hive/docs/router/supergraph
- **Hive Gateway Supergraph Proxy Source:** https://the-guild.dev/graphql/hive/docs/gateway/supergraph-proxy-source
- **Apollo Router integration:** https://the-guild.dev/graphql/hive/docs/other-integrations/apollo-router
- **Hive Console (for the local `dev:hive` stack):** [graphql-hive/console](https://github.com/graphql-hive/console)
- **Fumadocs (MDX docs framework):** https://fumadocs.dev
- **TanStack Start (app framework):** https://tanstack.com/start
- **Nitro dev server:** source of the SSR module-runner RPC timeout I debugged, tuned via the repo's `nitro-nightly` patch and `NITRO_VITE_RUNNER_TIMEOUT_MS`.
