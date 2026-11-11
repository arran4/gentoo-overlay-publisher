# Gentoo Overlay Publisher

**Status: design and foundation.** The public library, publisher executable and reusable GitHub Action are **not yet implemented or released**.

Gentoo Overlay Publisher aims to provide a small, reusable Go library for ebuild construction and safe Gentoo overlay reconciliation, with a GitHub Action as the primary publishing interface and a minimal CLI for local use and scripts.

## Scope

- **Library:** reusable, independently importable ebuild builder/IR, optional rendering, Gentoo version/revision/retention semantics, `metadata.xml` preservation, Manifest generation, repository snapshots and deterministic changesets.
- **GitHub Action (planned):** discover suitable release artifacts, prepare the candidate, reconcile the target overlay and propose changes in a pull request. It should plan automatically; saved plans are optional.
- **CLI (planned):** same engine as the Action. `publish` will be the default with explicit input and destination; `plan` is read-only inspection and `apply` modifies a local overlay only.
- **External verification:** G2 and Gentoo/Portage can verify results in CI, but the library must not import G2 or depend on it to publish.
- **Integration:** GoReleaser, Go2Gentoo, workflow generators and other producers can use the library or CLI without coupling the project's identity to a particular release tool.

The design targets **one repository and one root `go.mod`**, with narrow public package imports. Release-tool-specific YAML parsing, `dist/artifacts.json` processing and GitHub API operations belong in adapters, not the core library.

## Documentation

- [Documentation website (GitHub Pages, once enabled)](https://arran4.github.io/gentoo-overlay-publisher/)
- [Documentation source](docs/index.html)
- [GitHub repository metadata and Pages setup](.github/REPOSITORY_METADATA.md)
- [Dedicated smoke-test overlay](https://github.com/arran4/gentoo-overlay-publisher-smoke-overlay)

The initial Pages website is intentionally a **stub**. It will be expanded with a tested Action usage guide, CLI reference, Go API examples, architecture and integration coverage as those interfaces are implemented.

## Development and acceptance

The first implementation will extract, redesign and extend the relevant Gentoo features and tests from [the existing GoReleaser fork contribution](https://github.com/arran4/goreleaser/pull/37). Existing behavior and regression cases should be carried into provider-neutral packages where appropriate, while leaving well-defined extension points for broader ebuild constructs.

Verification should include pure planner tests, golden ebuild output, real Gentoo/Portage validation and separately secured end-to-end PR publishing into the [smoke overlay](https://github.com/arran4/gentoo-overlay-publisher-smoke-overlay). The smoke repository is **not** the implementation repository.

A future, separately reviewed upstream GoReleaser PR should depend on the released library and demonstrate a substantially smaller GoReleaser-specific code contribution, with its production LOC reduction measured rather than assumed.

**No public Go API, Action `uses:` tag or CLI syntax should be treated as stable until its implementation and tests are merged.** The repository's own license and attribution policy must be explicitly settled before source code is extracted.
