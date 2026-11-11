# Configuration design: portable GoReleaser Gentoo schema

**Status: compatibility-first design, 11 November 2026; no parser or Action is implemented yet.** This replaces the earlier simplified `package.install` proposal. It is based on the actual [GoReleaser fork `config.Gentoo` / `GentooInstallItem` types](https://github.com/arran4/goreleaser/blob/feature/gentoo-ebuild/pkg/config/config.go), the fork's [complete documented `gentoo_overlays` YAML](https://github.com/arran4/goreleaser/blob/feature/gentoo-ebuild/www/content/customization/publish/gentoo.md), and the [install IR, reducer, renderer and goldens](https://github.com/arran4/goreleaser/tree/feature/gentoo-ebuild/internal/pipe/gentoo).

## Core compatibility decision

**Use `gentoo_overlays` as the top-level array in the standalone publisher's optional `.gentoo-overlay-publisher.yaml`, with the same per-entry fields as the fork**. Do not substitute `package: {install: {executable: ...}}`, rename `doins` to `install.files`, or reduce supported install commands to one generic `destination` field. Each existing `gentoo_overlays:` entry should be copyable from `.goreleaser.yaml` with *no mandatory transformation of Gentoo-specific fields*. The standalone file can have its own root `version: 1` schema marker, independent of GoReleaser's `version: 2`.

For a normal GitHub Action run, **neither file is compulsory**: detect a matching GoReleaser config at the release's source tag/commit if available, read its `gentoo_overlays` stanza when present, inspect already published release archives, and allow explicit overrides. A publisher config is valuable where the producer has no `gentoo_overlays` stanza or a different overlay destination/policy is needed.

**Collapsed means omit defaulted or irrelevant keys from the *instance*, not remove supported keys from the schema.** Preserve the fork's full Gentoo helper and metadata vocabulary. V1 is binary-install ebuild publication, not upstream binary build/release/upload.

## Minimal migrated example: preserve original names

From the GoReleaser file (excerpt; `version: 2` and `builds`/`archives` remain in GoReleaser):

```yaml
gentoo_overlays:
  - name: header
    category: app-text
    license: MIT
    repository:
      owner: arran4
      name: gentoo-overlay-publisher-smoke-overlay
      pull_request:
        enabled: true
        draft: true
```

Into the optional publisher's file, simply copy that block:

```yaml
# .gentoo-overlay-publisher.yaml
version: 1
gentoo_overlays:
  - name: header
    category: app-text
    license: MIT
    repository:
      owner: arran4
      name: gentoo-overlay-publisher-smoke-overlay
      pull_request:
        enabled: true
        draft: true
```

No `project`/build commands, release-upload settings, full `archives` table or duplicated GoReleaser configuration are required. When the provided release proves one unambiguous executable per supported Linux archive, automatic installation may suffice; otherwise supply an explicit helper rule. The fork appends `-bin` to `name` when absent; preserve that compatibility where the name is safely resolved.

## Full vocabulary retained, with optional values

The fork's `config.Gentoo` schema has the following existing fields. Their YAML spelling and compatible meaning form the **porting baseline**:

| Concern | Existing fields (preserve names) | Intended location |
| --- | --- | --- |
| Selection and package identity | `id`, `ids`, `name`, `category`, `overlay_path`, `bin`, `type` | Provider-neutral package config, plus ID→release-archive adapter |
| Metadata and Gentoo package policy | `homepage`, `description`, `long_description`, `license`, `keywords`, `version_representation`, `bugs_to`, `maintainers`, `upstream`, `useflags`, `eclasses` | Package/ebuild model and metadata.xml updater |
| Automatic install and helpers | `bindir`, `dobin`, `doconfd`, `dodir`, `dodoc`, `doenvd`, `doexe`, `doheader`, `doinitd`, `doins`, `doman`, `dosbin`, `dosym`, `systemd`, `extra_install` | Public install-config model → typed ebuild IR → renderer |
| Auxiliary files | `files` (`glob`, `name_template`), `skip_files_validation` | Trusted source-file resolver; package `files/` emission |
| Version and retention | `keep_versions`, `conflict_resolution`, `version_retention_strategy` | Overlay planner; no spurious new `-rN` revisions |
| Manifests/cache | `manifest_hashes`, `thin_manifests`, `meta_cache` | Manifest and metadata planner, validated against `metadata/layout.conf` |
| Publishing | `repository`, `commit_author`, `commit_msg_template`, `skip_upload` | Separate Git/PR publisher; config shape supported without importing GoReleaser clients |

In the original fork `bin` **must be true** and `type` is effectively limited to `bin`. The new binary-only publisher can treat omission of both as `bin`; explicit `bin: true` and `type: bin` must remain accepted for painless migration. Do not silently accept unsupported nonbinary `type` values. Existing fork behavior and defaults, including `/opt/bin`, `app-misc`, revision conflict resolution, and `-bin` naming, must be recorded and tested. New defaults may be improved for Gentoo correctness **only as explicit compatibility-documented differences**, not by changing an existing entry's requested semantics.

### Install item shape: do not reinvent it

The ten item families `dobin`, `doconfd`, `doenvd`, `doexe`, `doheader`, `doinitd`, `doins`, `dosbin`, `dosym`, `systemd` (plus their supported behaviors) share the fork's item shape:

```yaml
doins:
  - src_id: default
    src: conf/app.conf
    dst: /etc/app/app.conf
    use: [config]
    archs: [amd64, arm64]
```

Fields are `src_id` (selected archive ID), `src` (source/archive member), `dst` (destination/rename), `use` (USE conditions, including negation), and `archs` (architecture guard). `src_id` suppresses only the corresponding default automatic binary install *for the matched archive/architecture/binary*, without discarding unrelated automatic installs. Archive ID, member path and architecture coverage must be validated against the actual released artifacts.

The list families `dodir`, `dodoc`, `doman` remain simple string lists, preserving the fork's configuration. The **IR** may internally represent all these as typed operations (including `newexe`, `newins`, `newbin`, `newman`, etc.), but YAML should remain compatible with the original helper-oriented structure. Use the fork's typed `InstallOp` descriptors, state-family tracking, condition planner and reducer rather than rebuilding ebuild shell output as string concatenation.

### Representative richer configuration

This example is **illustrative**; its source archive must actually contain the indicated paths for it to validate. Every omitted helper family in the table above remains supported.

```yaml
version: 1
gentoo_overlays:
  - id: myproject
    ids: [default, plugin]
    name: myproject
    category: app-admin
    license: MIT
    homepage: https://example.invalid/myproject
    description: Example binary package
    bindir: /usr/bin
    keywords: ["~amd64", "~arm64"]
    eclasses: [systemd]
    useflags:
      - flag: systemd
        description: Install the systemd service
      - flag: plugin
        description: Install the optional plugin
    doexe:
      - src_id: default
        src: myproject
        dst: /usr/bin/myproject
      - src_id: plugin
        src: plugin
        dst: /usr/libexec/myproject/plugin
        use: [plugin]
        archs: [amd64, arm64]
    doins:
      - src_id: default
        src: config/example.conf
        dst: /etc/myproject/example.conf
    systemd:
      - src_id: default
        src: share/myproject.service
        dst: myproject.service
        use: [systemd]
    dodoc: [README.md]
    doman: [man/myproject.1]
    keep_versions: 3
    version_retention_strategy: keep_latest
    conflict_resolution: Revision
    repository:
      owner: arran4
      name: gentoo-overlay-publisher-smoke-overlay
      pull_request:
        enabled: true
        draft: true
```

This preserves the original structure rather than forcing a second Gentoo policy model. Optional `files`, `maintainers`, `upstream`, `extra_install`, `meta_cache` and other fields can be added without changing nesting.

**Special cases:** `extra_install` is raw ebuild shell from trusted configuration, **not** code to execute by the publisher itself; preserve support and use G2/Portage validation without promising arbitrary static security analysis. `files` refers to additional trusted source-files to bundle into the overlay package's `files/` directory, not to upstream archives that the publisher uploads. `skip_files_validation` is explicit opt-out only, never inferred. `meta_cache` cannot claim complete eclass evaluation where G2/Portage is required.

## Configuration discovery, overlays and overrides

Keep existing `gentoo_overlays` entries in GoReleaser `.goreleaser.yaml` or `.goreleaser.yml` as a **first-class source**. The standalone publisher may read a standalone `.gentoo-overlay-publisher.yaml` to override or add selected entries **by `id`/resolved package destination**, without a second fully duplicated policy.

1. **Hard facts:** published release identity and tag, URLs, archive bytes/checksum/member layout, overlay base snapshot and repo ownership are validated separately and **are not overrideable** by configuration.
2. **Explicit Action/CLI overrides** select release source, target overlay, package ID and selective Gentoo/publishing options.
3. **Optional publisher YAML** supplies the same `gentoo_overlays` per-entry structure, including all helper families, and may add new overlays.
4. **GoReleaser YAML:** look up source config at the published tag/commit. Parse only `gentoo_overlays`, allowlisted `project_name`/`builds`/`archives` hints and selected releases; do **not** evaluate arbitrary templates, hooks or build steps.
5. **Existing overlay:** retain prior package settings and unowned metadata where safely observable; do not parse arbitrary existing ebuild Bash to invent install intent.
6. **Safe defaults:** fill only truly missing, unambiguous values. Publishing defaults to an isolated Draft PR, no direct default-branch push or implicit merge, no destructive retention by default, semantic no-op on rerun.

**Important:** merging overlay lists is an ordered, typed operation, *not* YAML list concatenation or a blind deep map merge. Select the same `id`/destination, reject duplicates, and replace complete install-family lists when overridden unless explicitly documented otherwise. Define null/empty/omitted semantics for lists, and display the provenance of each resolved field with `--explain-config` without exposing secrets. Cross-provider internal config types should never import GoReleaser's `pkg/config`.

**Legacy publishing fields:** accept the existing `repository` shape (owner, name, branch, `pull_request.enabled/base/draft`) and `commit_author`/`commit_msg_template`. Map them into a distinct publisher config. Historical inline `repository.token`, `pull_request.token` and `git.private_key` require a deliberate, secure migration policy—do not copy secrets into candidate plans or parsed artifact metadata. Translate legacy `skip_upload` to an explicit **nonpublishing** mode rather than treating it as permission to upload binaries. Distinguish package versions from the publisher's own repo version.

## Public Go API and refactor plan

**Default compatibility direction:** match existing Gentoo YAML keys, named install helpers and behavior while **refactoring the internal code** to form a cohesive public library. There are no pre-existing standalone library consumers that force awkward GoReleaser package boundaries.

Proposed single-module responsibilities:

```text
config/                    # GentooOverlay, InstallItem, UseFlag, Metadata,
                           # YAML parsing, validation and migration
ebuild/ir/                 # typed install program, op descriptors and conditions
ebuild/render/             # ebuild/phase rendering, eclass/USE handling
release/                   # normalized published-archive descriptors, no GoReleaser
version/                   # Gentoo package versions and revisions
manifest/                  # existing Manifest preservation and update
metadata/                  # metadata.xml updates, lossless unrelated content
overlay/                   # snapshots, retention, deterministic ChangeSet
internal/adapters/goreleaser/
                           # dist/artifacts.json, restricted YAML and release mapping
internal/gitpublish/       # Git branches, API/PR publication, auth boundaries
cmd/gentoo-overlay-publisher/
action.yml                 # wrapper on exactly the same CLI/runtime
```

The library exposes a portable **`config.GentooOverlay` and `config.InstallItem` schema** (names subject to implementation design), and converts to a validated, resolved config and then to ebuild IR. A future GoReleaser integration should ideally import the shared schema/types or use a very small, mechanically tested compatibility adapter; it must **not** duplicate a huge `Gentoo` struct and manually map each `do*` family. The public `overlay` planner can accept an already-generated ebuild without importing the IR.

**Refactor by responsibility, not just directory movement:**

| Fork source | New responsibility | Refactoring intent |
| --- | --- | --- |
| `pkg/config/config.go` Gentoo types, `internal/pipe/gentoo/config.go`, `gentoo.go` | `config` + narrow orchestration | Separate raw YAML, resolved immutable policy, release adapter and provider credentials |
| `installop.go`, `condition.go`, `install.go`, `reducer.go` | `ebuild/ir` planner, typed operations and reducer | Preserve semantic state/renames/guards and tests; remove GoReleaser context types |
| `ebuild.go`, `templates/ebuild.tmpl`, `generator.go` | `ebuild/render` and generation service | Explicit typed input/output and testable rendering, no release upload |
| `release.go`, `generated.go` | `release` model and `internal/adapters/goreleaser` | Normalize actual existing archives; avoid `artifact.Extra` leakage and provider internals |
| `version.go`, `retention.go`, `changeset.go`, `manifest.go`, `metadata.go` | `version`, `overlay`, `manifest`, `metadata` | Keep deterministic, independently testable reconciliation |
| `repository.go`, `publish.go` | `internal/gitpublish` | Separate Git/SCM I/O, PR safety and credentials from pure overlay planning |

**Full functional port expected.** Do not intentionally discard source-tested helper families, architecture-specific installs, binary suppression by `src_id`, eclass-order preservation, custom install paths, metadata/Manifest, retention, publication strategies, or tested failure propagation because an initial example is small. Fix existing source defects where identified; record deliberate behavior changes in compatibility tests.

## Required compatibility tests before acceptance

- Parse the **actual fork `gentoo_overlays` YAML block** using the standalone config parser without rewriting its Gentoo keys. Re-serialize/normalize and verify all relevant fields, including every `do*` family, `src_id`, `src`, `dst`, `use`, `archs`, `files`, `useflags`, `upstream`, `eclasses` and PR settings.
- Compare representative generated ebuild output to the fork's **real goldens**: `TestComplexInstallersTracking`, `TestDoRunWithDoinsAndDoman`, `TestDoRunWithSystemdAndUseFlags`, architecture-specific suppression, rename/exeinto/insinto state, multiarchive/multibinary cases, extra_install and metadata updates.
- Confirm valid override merging and precedence: omitted/empty/null list distinction, multiple `gentoo_overlays` entries, duplicate IDs and duplicate package destination rejection, explicit vs inferred `bindir`, safe no-op/retention/revision handling, and deterministic release-config selection.
- Prove a GoReleaser-free external program can import the configuration and IR without any GoReleaser/G2/GitHub dependencies, and independently plan changes with the `overlay` package.
- Verify that GoReleaser's accepted YAML has a **very small mechanically testable translation path** into the shared library config: future upstream adapter should reduce production Gentoo LOC instead of copying the full implementation.
- Keep source artifact build/upload outside the tool; real integration tests consume existing [header v0.0.6](https://github.com/arran4/header/releases/tag/v0.0.6) and publish **only Gentoo overlay files** to [the smoke repository](https://github.com/arran4/gentoo-overlay-publisher-smoke-overlay). No PR auto-merge.

## Action and CLI workflow examples (not yet runnable)

**The caller chooses the workflow trigger.** No `@v1` Action exists until this design and code pass integration testing.

```yaml
# Workflow A: in a separate workflow after release.published.
name: Propose Gentoo ebuild
on:
  release:
    types: [published]
permissions:
  contents: read
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          ref: ${{ github.event.release.tag_name }}
      - uses: arran4/gentoo-overlay-publisher@v1 # FUTURE, unreleased
        with:
          overlay-repository: arran4/gentoo-overlay-publisher-smoke-overlay
          package: app-text/header-bin
          goreleaser-config: auto
          token: ${{ secrets.OVERLAY_PUBLISH_TOKEN }}
```

```yaml
# Workflow B: AFTER the caller's existing successful GoReleaser release/upload step.
- uses: arran4/gentoo-overlay-publisher@v1 # FUTURE, unreleased
  with:
    overlay-repository: arran4/gentoo-overlay-publisher-smoke-overlay
    source: goreleaser-artifacts
    artifact-directory: dist
    goreleaser-config: auto
    token: ${{ secrets.OVERLAY_PUBLISH_TOKEN }}
```

For a local script, the same CLI accepts an explicit existing release and overlay destination; implicit `publish` constructs a fresh plan. Explicit `plan` previews and `apply` modifies only local files. Do not require a saved plan or a publisher YAML when a matching GoReleaser `gentoo_overlays` stanza and validated assets are sufficient.

## Small remaining design questions

1. **Config merging:** When the same `gentoo_overlays` ID exists in the GoReleaser and publisher YAML, should the latter replace the entire Gentoo entry or override field-by-field while treating each `do*` list as a replace-whole value? Recommended: field-by-field scalar overrides and replace-whole collection values, with explicit null/empty handling.
2. **Legacy credential fields:** Preserve the fork's `repository` YAML shape, but should token/private-key literal keys be strictly rejected in the public publisher YAML in favor of Action/CLI secret inputs? Recommended: reject literal embedded secrets; support secure environment references and never serialize credentials.
3. **Names:** Continue defaulting `app-misc/<project>-bin` for one unambiguous new package as the fork did, but refuse conflict/unknown licenses. Preserve existing target category and package identity; offer explicit overrides. This can be decided and tested with the header fixture.

Implementation should be a **semantic refactor of source files and data types**, with the source PR providing behavior and tests, not just a file copy into different directories.
