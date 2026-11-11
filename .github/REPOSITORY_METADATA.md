# Repository About metadata and GitHub Pages

The GitHub repository's **About** settings and topics are not automatically derived from README or documentation files. Apply these values on the repository's main page using the About gear icon.

## Suggested description

> Generate and publish Gentoo ebuilds for existing binary releases with a Go library and GitHub Action.

## Suggested topics

`gentoo`, `gentoo-linux`, `gentoo-overlay`, `ebuild`, `portage`, `golang`, `github-actions`, `release-automation`, `package-management`

These are **repository topics**, not Git release tags. Do not create a `v1` release or Action major-version tag until a working, tested Action is available.

## Website

Once GitHub Pages is enabled, set the repository's Website field to:

https://arran4.github.io/gentoo-overlay-publisher/

This URL is the conventional project-site address. It is **not confirmed live** merely because a workflow file exists.

## Enable Pages

1. Repository **Settings → Pages → Build and deployment → Source: GitHub Actions**.
2. The initial Pages workflow PR is merged. Run the `Pages` workflow on `main` (or update documentation on `main`) to trigger deployment.
3. Visit the Pages URL and verify it loads. In **Settings → Environments**, restrict the `github-pages` environment to trusted branches if appropriate.

The `.github/workflows/pages.yml` workflow uploads `docs/` directly and deploys with the official Pages actions. It runs on changes to documentation or the deployment workflow, and can be triggered manually. No Node, site generator or Go toolchain is necessary for the stub.

Keep credentialed overlay publication tests separate from this public documentation deployment.
