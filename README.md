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

Start at [the implementation architecture issue #2](https://github.com/arran4/gentoo-overlay-publisher/issues/2) and follow the [issue forest](docs/architecture.md#milestones--issue-forest). Non-GoReleaser adapters are **out of V1**. **No public Go API, Action `uses:` tag or CLI syntax should be treated as stable until its implementation and tests are merged.** The repository's own license and attribution policy must be explicitly settled before source code is extracted.
