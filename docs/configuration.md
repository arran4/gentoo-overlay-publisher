# Configuration design: GoReleaser-first binary ebuild publication

**Status: proposed contract, 11 November 2026. These inputs, fields, commands and Action tags are not implemented or released.** This document is intended to guide issue [#2](https://github.com/arran4/gentoo-overlay-publisher/issues/2), the GoReleaser adapter [#5](https://github.com/arran4/gentoo-overlay-publisher/issues/5) and the Action [#7](https://github.com/arran4/gentoo-overlay-publisher/issues/7).

## What the publisher does not configure

This project does **not** configure how GoReleaser builds, archives, releases, uploads or signs binaries. Those operations must already be completed. Gentoo Overlay Publisher receives published binary assets and produces only the ebuild and the accompanying Gentoo overlay file changes needed for Portage to install them.

GoReleaser is the **only initially implemented release-system source**. Other adapter interfaces may be defined but remain deferred to [#9](https://github.com/arran4/gentoo-overlay-publisher/issues/9).

## User-facing entry points

The **GitHub Action is first-class** and the primary workflow interface; it calls the same small CLI and independent Go packages that scripts and local users may invoke.

- Default **publish**: resolve inputs, discover an already published release, generate a **fresh** overlay plan, validate and open/update a reviewable PR on a new branch. It must not require a saved plan or preceding local apply.
- Explicit **plan**: read-only human-readable/JSON preview. Optional output file only at the user's request.
- Explicit **apply**: modify a local target overlay with stale-content precondition checks, but never push, commit or create a PR.
- With no unambiguous release input or target destination, fail with an actionable diagnostic rather than making a guess.

### Configuration sources

These are layers, **not** alternative copies of a full release-engine configuration.

1. **Explicit Action inputs / CLI flags** override publication preferences and selected package policy fields.
2. **Optional publisher config** at `.gentoo-overlay-publisher.yaml` (or `.yml`) supplies package-specific Gentoo policy not provided through Action inputs.
3. **Restricted GoReleaser config** at `.goreleaser.yaml` or `.goreleaser.yml`, or a path explicitly supplied through an Action input, can supply *hints* about already-created release assets: project name, selected builds/archive IDs, GOOS/GOARCH, archive naming and packaged file paths. If an earlier fork's `gentoo_overlays` stanza is available, its Gentoo-specific values may be imported through a specifically supported, validated compatibility subset. No arbitrary hooks/templates/env/script execution.
4. **Existing overlay state** supplies package history and retained values for *updates* where a semantic value can be recognized reliably; it must not be indiscriminately reinterpreted as a full shell script to discover unspecified policy. Preserve unrelated content regardless.
5. **Built-in safe defaults** fill only safe, unambiguous gaps.

**Hard facts are not overridable:** actual release tag and publication status, archive bytes, archive contents, SHA/size, reachable public URL, target repository identity/current-base file digests, allowable paths and ownership boundaries. When hints disagree with facts, fail rather than silently invent a publishable result.

**Provenance is important:** the planner should record each resolved choice's source (explicit input / publisher config / GoReleaser hint / preserved overlay value / default) without including secrets. A `--explain-config` or `plan --format json` should make resolution inspectable.

### Automatic discovery of GoReleaser config

- `goreleaser-config: auto` (proposed default): when possible, inspect exactly one recognized file at the **source release tag/commit** (not whatever source branch HEAD happens to be when a `release.published` workflow runs). Explicit same-workflow artifact metadata may be associated with the matching commit/tag instead.
- `goreleaser-config: none`: disable GoReleaser YAML hint loading altogether.
- `goreleaser-config: ./path.yml`: inspect that explicit file as a restricted hint source.
- If both filename variants are present without a selector, or the selected config is for a different release, require clarification/selection.
- Never interpret the full GoReleaser config, call GoReleaser to rebuild, expand arbitrary templates, fetch secrets referenced by templates, or infer a public download URI from a local filename.
- Reading `dist/artifacts.json` is a separate input; parse an explicit allowlist of required metadata instead of copying arbitrary `Extra` values. Do not include secrets in ebuilds, plans, diagnostics or Actions artifacts.

## Safe-default policy (initial design)

| Area | Default and contextual behavior | Unsafe behavior to avoid |
| --- | --- | --- |
| Release source | Prefer actual published GitHub Release assets when triggered by `release.published`; in a post-GoReleaser workflow, use selected `dist/artifacts.json` only after published assets/URLs are confirmed | Assuming a build file has been uploaded |
| Package identity | Reuse explicitly selected package/known existing package; for new packages, tentatively retain fork-style `<project>-bin` and `app-misc` fallback **only where unique**, visible in the plan/PR | Guessing between multiple matches or changing an established package's category |
| Ebuild version and EAPI | Validate Gentoo version syntax and use a supported, tested EAPI; preserve an existing package's established EAPI/policy if safe | Copying arbitrary SemVer prerelease values verbatim, inventing an unsupported EAPI |
| Install destination | Keep an explicitly configured destination, including legacy `/opt/bin`, and preserve an existing package's policy **only if reliably known**; otherwise use Gentoo-appropriate `dobin`/binary destination for a new ordinary executable | Imposing the GoReleaser fork's `/opt/bin` default on every new package, guessing arbitrary shell behavior |
| Install content | Inspect archive members; install only safely identified executable(s) and explicitly configured auxiliary files | Guessing between multiple executables, mixing architectures or treating all archive members as executables |
| License/metadata | Use a valid Gentoo LICENSE expression provided by configuration or a reliably verified mapping; preserve unowned `metadata.xml` | Silently inventing a project's Gentoo license or replacing unrelated metadata |
| Architecture | Map actual Linux archives to Gentoo architecture/KEYWORDS with validated arch-specific install semantics | Treating Gentoo architecture keywords as arbitrary USE flags or selecting packages for unsupported platforms |
| Manifest/distfiles | Generate correct hashes from real bytes, unique distfile naming and thin/thick Manifest consistent with target overlay | Clobbering retained DIST entries, collisions, missing archive checksums |
| Revisions | No-op on semantically identical output; create revisions only when meaningful content changes and policy allows | Churning `-rN` on harmless regeneration |
| Retention | Keep existing versions by default; remove only under an **explicit** safe retention policy with a complete snapshot | Deleting retained versions or newer releases based on incomplete discovery |
| Publishing | Target a separate tool-owned branch and reviewable **Draft PR**; do not merge, direct-push to default branch or force-push automatically | Touching user branches, direct default-branch modifications, implicit destructive operations |
| Authentication | Caller supplies the target-overlay token through a secret/secure environment | Recording credentials in YAML, artifacts, logs or plans |

A default is not a license to fabricate data. Missing required Gentoo-specific fields, credentials, ambiguous install layouts or unsupported GoReleaser templates should produce a clear error showing which Action input/config key will resolve the ambiguity.

### Optional publisher-specific YAML

The file should be small and contain **Gentoo-only intent**, not a GoReleaser build recipe. Proposed shape, not yet an API/schema:

```yaml
version: 1
goreleaser:
  config: auto   # auto, none, or an explicit path
package:
  atom: app-text/header-bin
  license: MIT
  # When already maintained in the overlay, preserve explicitly known policy.
  install:
    executable: header
    destination: /usr/bin/header
overlay:
  repository: arran4/gentoo-overlay-publisher-smoke-overlay
  base: main
  retention:
    mode: preserve
publication:
  pull_request: true
  draft: true
```

Deliberately leave credentials, upstream artifact upload settings and full GoReleaser templates **out**. More complex archive member/install mappings may require separate typed structures as IR gains concrete tested use cases.

### Contextual GoReleaser import should not override Gentoo meaning

The source fork [`defaultGentooConfig`](https://github.com/arran4/goreleaser/blob/feature/gentoo-ebuild/internal/pipe/gentoo/gentoo.go) defaults `bindir` to `/opt/bin`, category to `app-misc`, appends `-bin` to the package name, and requires an explicit license. This history is valuable for legacy compatibility, **but the new tool should not automatically impose all legacy defaults** on fresh packages. Explicit compatible `gentoo_overlays` fields should retain their chosen destination and install operations. Existing package ebuilds may not be safely analyzed as arbitrary shell code; in that case, preserve raw content and require explicit install destination rather than assert it was inferred.

The common GoReleaser `nfpms` config is **not** automatically a Gentoo configuration. Its file/layout/license fields can only be used as *verified hints* where they are compatible with Gentoo semantics.

## GitHub Action quick-start patterns (not runnable yet)

These examples are **design targets** for the eventual README. The Action's `@v1` tag and input contract do not exist yet. Once the Action has been tested and released, replace tags with approved release SHA/version pins.

### A. Trigger after a published GitHub Release

This is convenient for a separate release-follow-up workflow. It does **not** change when or how the source project publishes binary archives.

```yaml
name: Propose Gentoo ebuild update

on:
  release:
    types: [published]

permissions:
  contents: read

jobs:
  ebuild:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          ref: ${{ github.event.release.tag_name }}
      - uses: arran4/gentoo-overlay-publisher@v1 # FUTURE - not released
        with:
          overlay-repository: arran4/gentoo-overlay-publisher-smoke-overlay
          package: app-text/header-bin
          source: github-release
          goreleaser-config: auto
          token: ${{ secrets.OVERLAY_PUBLISH_TOKEN }}
```

### B. Call immediately after the existing GoReleaser release step

In the **same trusted job** that already ran GoReleaser, place the Action **after** that project's existing successful release/upload step. Reuse the existing `dist/artifacts.json` when available; the publisher still verifies the assets are actually published and fetchable.

```yaml
# Existing build/release step runs first. It is NOT part of this publisher.
- uses: arran4/gentoo-overlay-publisher@v1 # FUTURE - not released
  with:
    overlay-repository: arran4/gentoo-overlay-publisher-smoke-overlay
    package: app-text/header-bin
    source: goreleaser-artifacts
    artifact-directory: dist
    goreleaser-config: auto
    token: ${{ secrets.OVERLAY_PUBLISH_TOKEN }}
```

### C. Explicit preview / local scripting

```sh
# FUTURE CLI - not implemented
gentoo-overlay-publisher plan \
  --release arran4/header@v0.0.6 \
  --package app-text/header-bin \
  --overlay ./checkout \
  --explain-config

# Same CLI has implicit publish, which ALWAYS plans freshly against the target.
gentoo-overlay-publisher \
  --release arran4/header@v0.0.6 \
  --package app-text/header-bin \
  --repository arran4/gentoo-overlay-publisher-smoke-overlay
```

All three patterns should share the same resolved configuration object, discovered asset model, change planner, error codes and PR safety rules. **A caller chooses the event/trigger** and the Action does not force it to be `release.published`.

## Remaining questions, not blanket blockers

1. Is the split between GoReleaser **build hints** and optional publisher **Gentoo policy** preferable to interpreting the old fork's `gentoo_overlays` stanza as the primary source when present? Suggested design: accept a validated compatibility subset first and let explicit publisher configuration override it.
2. For a **new package** with no explicit category/name, should `app-misc/<project>-bin` remain the visible safe default when unique, or should the Action require `package:` from day one? For existing managed packages, preserve the actual atom and explicit choices.
3. Which fields are genuinely inferable from existing ebuild content without invoking G2? Prefer conservative preservation and explicit configuration over a brittle Bash parser. This can be settled by first implementing the `header` fixture and recording its limitations.

An incomplete configuration should produce a precise validation diagnostic rather than silently publishing a malformed or mislocated ebuild.
