# Architecture and implementation readiness

**Status:** Design and backlog established, 11 November 2026. Nothing here is a released API, CLI or GitHub Action.

## Purpose

**Gentoo Overlay Publisher publishes Gentoo ebuilds for binary programs that have already been built and published elsewhere.** In the initial implementation, the source of those artifacts is **GoReleaser**. The output is a GitHub pull request against a Gentoo overlay.

This project **does not** compile project code, build application binaries, produce source project releases, upload archives, distribute executables, or overwrite a release system's publishing workflow.

Its output can contain generated `.ebuild` files and necessary Gentoo package metadata: `Manifest`, `metadata.xml`, package-local `files/` helpers, and appropriate version/retention updates. It **must not** check upstream release binaries into the overlay or push them to a release host.

## Pipeline

```text
Other tool (initially GoReleaser, already complete):
  build binaries -> produce archives -> publish release assets
                                     |
                              immutable artifacts
                                     v
Gentoo Overlay Publisher:
  discover/validate release assets and archive contents
          -> build binary-ebuild IR and render ebuild text
          -> load existing overlay state
          -> plan ebuild + metadata + Manifest + retention changes
          -> publish branch + pull request to target overlay
                                     |
                            user/CI verifies & merges PR
                                     v
                    Portage downloads the external release
                    assets and installs the binary package
```

G2/Portage may be used separately to verify generated outputs. The public library does **not** depend on G2 or GoReleaser internals.

## Primary interfaces

**GitHub Action (first-class, intended routine usage):** a thin wrapper on the Go executable. An ordinary workflow should need source release identification, target overlay, package identity and credentials. It should not need a bespoke candidate YAML for conventional published release assets.

**Minimal CLI (scripts and local):** implicit `publish` for an explicit target and input; explicit `plan` is read-only, explicit `apply` modifies only local overlay content. Default `publish` constructs its plan from fresh remote state; saved plans are not required and must be revalidated if supplied.

**Go library:** an independently importable typed install IR and renderer for binary ebuilds, plus pure overlay reconciliation (`version`, `metadata`, `manifest`, `overlay`). One Go module. The library must not import GitHub, G2 or GoReleaser. Public IR should support today's needed install operations while leaving reasonable extension points for future Gentoo package constructs without implementing a full shell parser.

## Initial artifact adapter: GoReleaser only

Supported discovery *goals*, pending exact input contract:

- On a `release.published` workflow, inspect the already published GitHub Release assets and their public download URLs.
- For a same-workflow invocation after GoReleaser completes publication, optionally read `dist/artifacts.json` as additional structured evidence, confirming local file bytes and URLs against actual assets.
- Optionally interpret a **restricted** subset of `.goreleaser.yaml` / `.goreleaser.yml` only for archive names, IDs, platform mappings, wrapped-directory conventions and file install mapping. Do **not** implement GoReleaser config execution, arbitrary templates or hooks.
- Optional project-specific `.gentoo-overlay-publisher.yaml` may specify the overlay destination, package identity, install mapping, retention and PR policies. It must not be mandatory for simple supported release layouts.
- Reject ambiguous archive/member/platform mapping, missing public URL, malformed or unpublished assets, unavailable retained overlay files, and credentials accidentally embedded in artifact metadata.

A provider-neutral archive model allows future adapters, but **non-GoReleaser providers are outside V1** and explicitly tracked in [future provider issue #9](https://github.com/arran4/gentoo-overlay-publisher/issues/9). Even the generic GitHub Release inspection path initially targets releases produced by GoReleaser.

## Overlay publishing and safety

- Incremental updates modify only an explicitly owned package and its required package files. Unrelated ebuilds and packages remain unchanged.
- Preserve existing XML metadata and Manifest records without rewriting unowned data. Apply revisions only for meaningful content changes; no-op reruns create no duplicate PR.
- Version/retention ordering must not discard a newer package when backfilling an older release. Filenames and hashes of retained distfiles must remain valid.
- Publishing uses tool-owned Git branches and reviewable PRs; no implicit direct-to-default-branch push or auto-merge. Existing PR policies include new-PR, ignore-existing, close-and-replace and explicit reuse, with controlled force-with-lease when specifically requested.
- The Action receives limited credentials appropriate to the destination overlay, not broad credentials to release source projects. No token or SSH private key in plan files, `dist/artifacts.json`, ebuilds, workflow artifacts or logs.
- Read-only preview and local apply must use the **same** planner and report precondition failures rather than silently overwriting stale state.

## Milestones / issue forest

| Stage | GitHub issue | Outcome |
| --- | --- | --- |
| 1 — Design freeze | [#2](https://github.com/arran4/gentoo-overlay-publisher/issues/2) | Stable scope, Go types, metadata, licensing and dependency rules |
| 2 — Ebuild construction | [#3](https://github.com/arran4/gentoo-overlay-publisher/issues/3) | Full relevant builder/IR/render extraction from fork PR #37 |
| 3 — Overlay management | [#4](https://github.com/arran4/gentoo-overlay-publisher/issues/4) | Metadata, Manifest, versions/revisions, safe deterministic ChangeSet |
| 4 — GoReleaser adapter | [#5](https://github.com/arran4/gentoo-overlay-publisher/issues/5) | Published binary archive discovery without building/publishing artifacts |
| 5 — CLI | [#6](https://github.com/arran4/gentoo-overlay-publisher/issues/6) | Implicit publish, explicit plan and local-only apply |
| 6 — First-class Action | [#7](https://github.com/arran4/gentoo-overlay-publisher/issues/7) | Same execution engine, Action input/output contract and PR publishing |
| 7 — Proof | [#8](https://github.com/arran4/gentoo-overlay-publisher/issues/8) | Real Portage fetch/install and isolated remote smoke PR |
| Later — other providers | [#9](https://github.com/arran4/gentoo-overlay-publisher/issues/9) | Explicitly deferred, no implementation in V1 |
| Later — upstream contribution | [#10](https://github.com/arran4/gentoo-overlay-publisher/issues/10) | New small upstream GoReleaser adapter PR using this library |

The [dedicated smoke overlay](https://github.com/arran4/gentoo-overlay-publisher-smoke-overlay) has been initialized and its first PR merged. That **does not** establish real Portage or Action publication success.

## Questions needed before writing implementation PRs

1. **First invocation shape.** Should V1 prioritize triggering on GitHub `release.published` (after released binaries are visible), using same-job `dist/artifacts.json` only when available? Or is same-workflow publishing the dominant workflow that must work first? Both should ultimately be supported without two engines.
2. **Binary/archive layout defaults.** Which of your real GoReleaser repositories is the first integration fixture? Record its release archive names, OS/arch mappings and whether archives contain a single executable, multiple commands, or supporting files. This determines the minimum Action inputs and the IR's install semantics.
3. **Licensing and attribution.** Choose a license for this public repository before extracting code from [fork PR #37](https://github.com/arran4/goreleaser/pull/37). Honor copyright/attribution and all source license requirements.
4. **PR ownership and credential policy.** Decide the initial default PR strategy (prefer `new-pr` with no force-push), and choose how the workflow gets cross-repository target permissions. No credentials in arbitrary untrusted PR workflows.

Design choices such as other release-producer support, source ebuild generation, full Bash analysis, G2 integration and automatic overlay-wide ownership deletion are **not blockers** for the GoReleaser-first binary workflow.

## Pausing and resumption

This design and the issue backlog can be prepared by ChatGPT using the connected GitHub repository without relying on a separate coding agent. The **large multi-file Go extraction, refactoring, regression suite and true end-to-end Portage/publishing verification** are substantial coding work; do not claim completion until the actual PRs and CI runs prove them. It is safe to pause implementation after this design/issue stage and resume from [issue #2](https://github.com/arran4/gentoo-overlay-publisher/issues/2). No background job is assumed.
