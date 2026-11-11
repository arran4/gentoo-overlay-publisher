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

**GitHub Action (first-class, intended routine usage):** a thin wrapper on the Go executable. An ordinary workflow should need source release identification, target overlay, package identity and credentials. It should not need a bespoke candidate YAML for conventional published release assets. **The caller chooses its trigger**: separate `release.published` workflows, invocation after a successful GoReleaser release in the same job, manually dispatched workflows and other trusted triggers must be supported by the same engine. See [quick starts](../README.md#quick-start-planned-action-usage) and [configuration design](configuration.md).

**Minimal CLI (scripts and local):** implicit `publish` for an explicit target and input; explicit `plan` is read-only, explicit `apply` modifies only local overlay content. Default `publish` constructs its plan from fresh remote state; saved plans are not required and must be revalidated if supplied.

**Go library:** an independently importable typed install IR and renderer for binary ebuilds, plus pure overlay reconciliation (`version`, `metadata`, `manifest`, `overlay`). One Go module. The library must not import GitHub, G2 or GoReleaser. Public IR should support today's needed install operations while leaving reasonable extension points for future Gentoo package constructs without implementing a full shell parser.

## Initial artifact adapter: GoReleaser only

Supported discovery *goals*, pending exact input contract:

- On a `release.published` workflow, inspect the already published GitHub Release assets and their public download URLs.
- For a same-workflow invocation after GoReleaser completes publication, optionally read `dist/artifacts.json` as additional structured evidence, confirming local file bytes and URLs against actual assets.
- Optionally interpret a **restricted** subset of `.goreleaser.yaml` / `.goreleaser.yml` only for archive names, IDs, platform mappings, wrapped-directory conventions and file install mapping. Do **not** implement GoReleaser config execution, arbitrary templates or hooks.
- **Configuration compatibility is a first-class acceptance requirement:** optional `.gentoo-overlay-publisher.yaml` contains the **same top-level `gentoo_overlays` entries and Gentoo-specific YAML keys as fork PR #37**, with only a publisher schema `version: 1` added. The `gentoo_overlays` stanza in a matching GoReleaser file is a primary, validated source—not merely a vague hint. Parse supported `project_name`/`builds`/`archives` as restricted artifact-selection hints, but never execute arbitrary GoReleaser release configuration. A second publisher YAML should not be mandatory. [Complete field inventory, examples, compatibility matrix and porting plan](configuration.md).
- Reject ambiguous archive/member/platform mapping, missing public URL, malformed or unpublished assets, unavailable retained overlay files, and credentials accidentally embedded in artifact metadata.

A provider-neutral archive model allows future adapters, but **non-GoReleaser providers are outside V1** and explicitly tracked in [future provider issue #9](https://github.com/arran4/gentoo-overlay-publisher/issues/9). Even the generic GitHub Release inspection path initially targets releases produced by GoReleaser.

## Porting design: full feature parity with a cohesive shared library

The original [fork PR #37](https://github.com/arran4/goreleaser/pull/37) provides the full semantic baseline: a `config.Gentoo` struct with `gentoo_overlays` YAML; `GentooInstallItem` with `src_id`, `src`, `dst`, `use`, `archs`; all helper families (`dobin`, `doconfd`, `dodir`, `dodoc`, `doenvd`, `doexe`, `doheader`, `doinitd`, `doins`, `doman`, `dosbin`, `dosym`, `systemd`); and `files`, `extra_install`, `useflags`, metadata, manifest, revision, retention and publication controls.

Port the full useful binary-ebuild functionality and original tests/goldens into the public single-module library. The required work is **semantic refactoring of data types, file contents, responsibilities and runtime coupling**, not a directory rename or a reduced rewrite. Preserve helper YAML spelling and tested install behavior; separate parser/validated policy, normalized published artifact input, typed installation IR/reducer/renderer, overlay state + ChangeSet, and Git PR publication. The core must never import GoReleaser or G2; the future upstream GoReleaser PR should import the library or use a *thin tested adapter*, not reimplement every do-helper.

The `gentoo_overlays:` block should copy directly into standalone config after adding only `version: 1`; optional publisher overrides can merge by matching `id`/destination without repeating the whole GoReleaser file. Distinguish migration of intentionally configured legacy `bindir: /opt/bin` from safe Gentoo defaults for a new package. Keep public API portable and add an explicit compatibility test between the original fork YAML and the new parser. Full details: [configuration design](configuration.md) and [issue #12](https://github.com/arran4/gentoo-overlay-publisher/issues/12).

## Configuration and safe defaults

The redesigned compatibility-first config in [docs/configuration.md](configuration.md) distinguishes **hard facts** from **configuration preferences**. Existing published assets, archive contents, public URLs and current overlay snapshots are validated facts and cannot be overridden by conflicting YAML. Preference resolution is explicit Action/CLI input → optional Gentoo Overlay Publisher YAML → supported/restricted GoReleaser config fields → reliably known existing overlay conventions → conservative defaults.

Preserve **explicit** GoReleaser Gentoo install settings, Gentoo installation correctness and already-established package policy. The original GoReleaser fork defaults to `/opt/bin` for `bindir`, but that historical choice must not automatically override a Gentoo-appropriate install destination for new ordinary binaries. Conversely, do not silently change an existing package's known install path. Missing licensing, archive mapping or unresolvable metadata should be actionable errors, not fabricated values.

**License fixed:** this public repository is MIT, matching [GoReleaser's MIT license](https://github.com/goreleaser/goreleaser/blob/main/LICENSE.md). Preserve required upstream copyright and attribution, and use [third-party notices](../THIRD_PARTY_NOTICES.md) when source is adapted. The repository's license is **not** necessarily the Gentoo `LICENSE` of each package being published.

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
| 2 — Ebuild construction | [#3](https://github.com/arran4/gentoo-overlay-publisher/issues/3) | Full relevant builder/IR/render extraction, all helper families, source golden tests, refactored files/types |
| 3 — Overlay management | [#4](https://github.com/arran4/gentoo-overlay-publisher/issues/4) | Metadata, Manifest, versions/revisions, safe deterministic ChangeSet |
| 3.5 — Configuration resolution | [#12](https://github.com/arran4/gentoo-overlay-publisher/issues/12) | Contextual GoReleaser hints, safe defaults, Action overrides and explainable provenance |
| 4 — GoReleaser adapter | [#5](https://github.com/arran4/gentoo-overlay-publisher/issues/5) | Published binary archive discovery without building/publishing artifacts |
| 5 — CLI | [#6](https://github.com/arran4/gentoo-overlay-publisher/issues/6) | Implicit publish, explicit plan and local-only apply |
| 6 — First-class Action | [#7](https://github.com/arran4/gentoo-overlay-publisher/issues/7) | Same execution engine, Action input/output contract and PR publishing |
| 7 — Proof | [#8](https://github.com/arran4/gentoo-overlay-publisher/issues/8) | Real Portage fetch/install and isolated remote smoke PR |
| Later — other providers | [#9](https://github.com/arran4/gentoo-overlay-publisher/issues/9) | Explicitly deferred, no implementation in V1 |
| Later — upstream contribution | [#10](https://github.com/arran4/gentoo-overlay-publisher/issues/10) | New small upstream GoReleaser adapter PR using this library |

The [dedicated smoke overlay](https://github.com/arran4/gentoo-overlay-publisher-smoke-overlay) has been initialized and its first PR merged. That **does not** establish real Portage or Action publication success.

## Questions needed before writing implementation PRs

1. **Invocation trigger decided:** users choose their own workflow trigger. Publish on `release.published`, after a successful same-job release or from an explicitly configured trusted workflow; provide multiple pasteable examples, never force one scheduling model.
2. **Binary/archive layout defaults.** Proposed first fixture: [arran4/header v0.0.6](https://github.com/arran4/header/releases/tag/v0.0.6), a published GoReleaser release with Linux/386, amd64 and arm64 tar.gz archives. The reference [app-text/header-bin](https://github.com/arran4/arrans_overlay/tree/main/app-text/header-bin) package already exists; test output must go to the dedicated smoke overlay, **not** the live overlay. Confirm install-path and auxiliary-file policy; do not blindly copy older archive architecture guards. Richer follow-up fixture: [go-playerctl v0.0.23](https://github.com/arran4/go-playerctl/releases/tag/v0.0.23).
3. **Licensing decided:** match GoReleaser under the MIT license; retain source attribution and copyright notices when extracting material from [fork PR #37](https://github.com/arran4/goreleaser/pull/37). License and notices are included in PR #11.
4. **PR ownership and credential policy remains:** prefer new Draft PRs on tool-owned branches, no force-push by default; caller supplies cross-repository target permissions securely. No credentials in arbitrary untrusted PR workflows.

Design choices such as other release-producer support, source ebuild generation, full Bash analysis, G2 integration and automatic overlay-wide ownership deletion are **not blockers** for the GoReleaser-first binary workflow.

## Pausing and resumption

This design and the issue backlog can be prepared by ChatGPT using the connected GitHub repository without relying on a separate coding agent. The **large multi-file Go extraction, refactoring, regression suite and true end-to-end Portage/publishing verification** are substantial coding work; do not claim completion until the actual PRs and CI runs prove them. It is safe to pause implementation after this design/issue stage and resume from [issue #2](https://github.com/arran4/gentoo-overlay-publisher/issues/2). No background job is assumed.
