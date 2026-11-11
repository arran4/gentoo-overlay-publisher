# Gentoo Overlay Publisher

**Status: design and foundation.** The public library, publisher executable and reusable GitHub Action are **not yet implemented or released**.

Gentoo Overlay Publisher generates and publishes **Gentoo ebuilds to install application binaries that have already been built and published by another tool**. It provides a reusable Go library, a first-class GitHub Action and a minimal CLI for scripts and local use. **GoReleaser is the initial release artifact provider.**

## Scope

**Not a build or artifact-publishing system:** it does not compile application code, build archives, upload binaries, or create the source project's release. It reads existing published archives and opens a pull request containing only the ebuilds and supporting Gentoo overlay metadata required to install those binaries. Portage subsequently fetches the external published binary archives via their `SRC_URI`.

- **Library:** reusable, independently importable ebuild builder/IR, optional rendering, Gentoo version/revision/retention semantics, `metadata.xml` preservation, Manifest generation, repository snapshots and deterministic changesets.
- **GitHub Action (planned):** inspect **already published GoReleaser-produced binary releases**, generate the binary ebuild candidates, reconcile the target overlay and propose only the ebuild/metadata changes in a pull request. It should plan automatically; saved plans are optional.
- **CLI (planned):** same engine as the Action. `publish` will be the default with explicit input and destination; `plan` is read-only inspection and `apply` modifies a local overlay only.
- **External verification:** G2 and Gentoo/Portage can verify results in CI, but the library must not import G2 or depend on it to publish.
- **Integration:** GoReleaser first. Other release-system adapters and workflow-generator integration are explicitly deferred and tracked in [issue #9](https://github.com/arran4/gentoo-overlay-publisher/issues/9), while the library's data model stays provider-neutral.

The design targets **one repository and one root `go.mod`**, with narrow public package imports. Release-tool-specific YAML parsing, `dist/artifacts.json` processing and GitHub API operations belong in adapters, not the core library.

## Portable GoReleaser-compatible Gentoo configuration

The optional `.gentoo-overlay-publisher.yaml` uses the **same `gentoo_overlays` fields as the Gentoo section of the existing GoReleaser fork**. Existing `doexe`, `dobin`, `doins`, `doman`, `dodoc`, `systemd`, `src_id`, conditional `use` and architecture `archs` rules remain in scope. The new file **does not require** the unrelated GoReleaser `builds`, `archives` or release-upload sections.

```yaml
# Optional .gentoo-overlay-publisher.yaml — proposed, not yet implemented
version: 1
gentoo_overlays:
  - name: header
    category: app-text
    license: MIT
    doexe:
      - src_id: default
        src: header
        dst: /usr/bin/header
    repository:
      owner: arran4
      name: gentoo-overlay-publisher-smoke-overlay
      pull_request:
        enabled: true
        draft: true
```

For normal GoReleaser workflows, the publisher should **also read an existing `gentoo_overlays` block directly from the matching `.goreleaser.yml` or `.goreleaser.yaml`**, if present, without requiring a second config file. The YAML helper schema should be copyable across with virtually no translation; internal Go types and source files must be refactored into provider-neutral, cohesive packages rather than simply copied.

Full supported vocabulary, migration examples, compatibility testing and planned public package boundaries are in [configuration design](docs/configuration.md).

## Default usage: publish immediately, with no explicit plan or release tag

A normal Action invocation should require **only enough information to identify the package and overlay** (plus target-repository authentication). It discovers a reliable GoReleaser release tag, generates the ebuild from existing published binaries, calculates/validates a temporary internal plan, then opens a reviewable overlay PR. A user does **not** need to invoke `plan`, save a plan, run `apply`, upload a release, or create a second publisher YAML.

```yaml
# Place after an existing successful GoReleaser publication.
# Future Action, not yet implemented or released.
- uses: arran4/gentoo-overlay-publisher@v1
  with:
    overlay-repository: arran4/gentoo-overlay-publisher-smoke-overlay
    package: app-text/header-bin
    token: ${{ secrets.OVERLAY_PUBLISH_TOKEN }}
```

A release event supplies `release.tag_name`; a post-release job may supply `dist/metadata.json` (`tag`, `version`, `commit`). An explicit Action `tag` or CLI `--tag` overrides discovery, including historical backfills. Git tag/environment context follows; **latest published release is an opt-in last resort**, warning with the chosen tag. Missing evidence may fall back, but contradictory or stale release identities must fail rather than silently publish the wrong version. See [release-tag rules and examples](docs/configuration.md#release-tag-resolution-zero-setup-publish-deterministic-and-fail-safe) and [issue #13](https://github.com/arran4/gentoo-overlay-publisher/issues/13).

## Quick start (planned Action usage)

**These examples are design previews, not runnable commands yet.** The `@v1` Action tag will be created only after implementation, real smoke tests and release. The Action is intended to run **after** GoReleaser has finished building and publishing the upstream binary archives; it never creates or uploads those archives.

### Option A: separate workflow on a published release

```yaml
name: Publish Gentoo ebuild

on:
  release:
    types: [published]

permissions:
  contents: read

jobs:
  publish-ebuild:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          ref: ${{ github.event.release.tag_name }}
      - uses: arran4/gentoo-overlay-publisher@v1 # future tag
        with:
          overlay-repository: arran4/gentoo-overlay-publisher-smoke-overlay
          package: app-text/header-bin
          source: github-release
          goreleaser-config: auto
          token: ${{ secrets.OVERLAY_PUBLISH_TOKEN }}
```

### Option B: directly after an existing GoReleaser release step

```yaml
# Place this AFTER your existing successful GoReleaser release step.
- uses: arran4/gentoo-overlay-publisher@v1 # future tag
  with:
    overlay-repository: arran4/gentoo-overlay-publisher-smoke-overlay
    package: app-text/header-bin
    source: goreleaser-artifacts
    artifact-directory: dist
    goreleaser-config: auto
    token: ${{ secrets.OVERLAY_PUBLISH_TOKEN }}
```

Neither trigger is imposed by the tool; the workflow author chooses the schedule, event or calling job. On publish, the tool **plans automatically against the current overlay state** and opens a reviewable PR. A saved plan is optional.

### Optional configuration

For GoReleaser workflows, auto-discover the matching `.goreleaser.yml` or `.goreleaser.yaml` at the release tag/commit and accept its original `gentoo_overlays` entries when present. Read only the supported `builds`/`archives` hints required to reconcile *already published* assets; never execute GoReleaser's build configuration or arbitrary templates. An optional `.gentoo-overlay-publisher.yaml` uses the **same Gentoo fields** for selective overrides and new package policy rather than duplicating an entire release config.

See [configuration rules, safe defaults, legacy compatibility and more examples](docs/configuration.md). The intended project license is [MIT](LICENSE.md), matching GoReleaser while preserving required [source attribution](THIRD_PARTY_NOTICES.md).

## Documentation

- [Documentation website (GitHub Pages, once enabled)](https://arran4.github.io/gentoo-overlay-publisher/)
- [Architecture, scope, issues and open decisions](docs/architecture.md)
- [Documentation source](docs/index.html)
- [GitHub repository metadata and Pages setup](.github/REPOSITORY_METADATA.md)
- [Dedicated smoke-test overlay](https://github.com/arran4/gentoo-overlay-publisher-smoke-overlay)

The initial Pages website is intentionally a **stub**. It will be expanded with a tested Action usage guide, CLI reference, Go API examples, architecture and integration coverage as those interfaces are implemented.

## Development and acceptance

The first implementation will extract, redesign and extend the **binary ebuild** functionality and overlay-management features from [the existing GoReleaser fork contribution](https://github.com/arran4/goreleaser/pull/37). This is not an implementation of GoReleaser's release pipeline. Existing ebuild and overlay regression cases should be carried into provider-neutral packages while preserving sensible future extension points.

Verification should include pure planner tests, golden ebuild output, real Gentoo/Portage validation and separately secured end-to-end PR publishing into the [smoke overlay](https://github.com/arran4/gentoo-overlay-publisher-smoke-overlay). The smoke repository is **not** the implementation repository.

A future, separately reviewed upstream GoReleaser PR should depend on the released library and demonstrate a substantially smaller GoReleaser-specific code contribution, with its production LOC reduction measured rather than assumed.

Start at [the implementation architecture issue #2](https://github.com/arran4/gentoo-overlay-publisher/issues/2) and follow the [issue forest](docs/architecture.md#milestones--issue-forest). Non-GoReleaser adapters are **out of V1**. **No public Go API, Action `uses:` tag or CLI syntax should be treated as stable until its implementation and tests are merged.** The project uses the [MIT license](LICENSE.md), matching GoReleaser, with [upstream attribution](THIRD_PARTY_NOTICES.md) preserved when adapting source.
