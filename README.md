# .github

Organization-wide defaults for [`ninoverse`](https://github.com/ninoverse).

GitHub serves the community health files below to **every** repository in the
organization that does not ship its own copy — public and private alike. There
is nothing to install and no per-repository commit: a repository with no
`CONTRIBUTING.md` of its own displays this one.

| File | Applies to |
|------|-----------|
| [`CONTRIBUTING.md`](CONTRIBUTING.md) | The branch → commit → PR loop. Process only; build commands stay in each repository's README. |
| [`SECURITY.md`](SECURITY.md) | Private disclosure process and what is in scope. |
| [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md) | Contributor Covenant 2.1, adapted. |
| [`.github/ISSUE_TEMPLATE/`](.github/ISSUE_TEMPLATE/) | Bug and feature forms, plus the security contact link. |
| [`.github/PULL_REQUEST_TEMPLATE.md`](.github/PULL_REQUEST_TEMPLATE.md) | What / Why / How / Testing. |

## Overriding

Override is **all-or-nothing per file**. A repository that ships its own
`CONTRIBUTING.md` inherits nothing from this one — the files are not merged.

Override when a repository needs materially different content, not to add a
paragraph. `claude-mit-rust-template` keeps its own `CONTRIBUTING.md` and pull
request template because both are dense with Rust and `just` specifics that do
not generalize; it inherits everything else.

## Reusable workflows

Workflows are **not** defaultable — a repository never inherits one. They can be
*called* instead, which costs each repository a short caller file but keeps the
job definitions in one place.

| Workflow | Gates |
|---|---|
| [`rust-ci.yml`](.github/workflows/rust-ci.yml) | `fmt` · `clippy` · `test` · `deny` · MSRV · coverage artifact |
| [`rust-audit.yml`](.github/workflows/rust-audit.yml) | `cargo audit` · `cargo deny check advisories` |
| [`go-ci.yml`](.github/workflows/go-ci.yml) | `fmt` · `vet`+`lint` · `test -race` · `licenses`+`vuln` · go.mod floor · coverage artifact |
| [`go-audit.yml`](.github/workflows/go-audit.yml) | `govulncheck` |

**The contract is the runner recipe, never the tool behind it.** CI calls the
recipe so each command — and each tool version pin — keeps exactly one
definition, in the calling repository. The runner file itself cannot be
centralized, since neither `just` nor `make` has a remote include, so adopting
repositories copy it.

| Ecosystem | Runner | Recipes the workflows call |
|---|---|---|
| Rust | `justfile` | `fmt-check` `lint` `test` `deny` `audit` |
| Go | `Makefile` | `fmt-check` `vet` `lint` `test-race` `vuln` `licenses` `cover`, plus `tools-lint` `tools-test` `tools-vuln` `tools-licenses` |

Go needs the `tools-*` split because its linters arrive by `go install` rather
than as prebuilt binaries: each gate installs only what it uses, and the pinned
`golangci-lint` version stays in the Makefile where `make lint` can also see it.

### Rust

`.github/workflows/ci.yml` in the calling repository:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

# Belongs in the caller: inside a called workflow, `github.workflow` still
# resolves to the caller's name.
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    uses: ninoverse/.github/.github/workflows/rust-ci.yml@v1
    with:
      msrv: "1.85" # must match [workspace.package].rust-version
```

`.github/workflows/audit.yml`:

```yaml
name: Audit

on:
  schedule:
    - cron: "0 6 * * 1"
  push:
    branches: [main]
    paths: ["**/Cargo.toml", "**/Cargo.lock", "deny.toml"]
  pull_request:
    paths: ["**/Cargo.toml", "**/Cargo.lock", "deny.toml"]
  workflow_dispatch:

jobs:
  audit:
    uses: ninoverse/.github/.github/workflows/rust-audit.yml@v1
```

Repositories with no `deny.toml` pass `deny: false`; `coverage: false` skips the
coverage build.

### Go

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    uses: ninoverse/.github/.github/workflows/go-ci.yml@v1
    with:
      go-version: "1.25" # must match the `go` directive in go.mod
```

```yaml
name: Audit

on:
  schedule:
    - cron: "0 6 * * 1"
  push:
    branches: [main]
    paths: ["go.mod", "go.sum"]
  pull_request:
    paths: ["go.mod", "go.sum"]
  workflow_dispatch:

jobs:
  audit:
    uses: ninoverse/.github/.github/workflows/go-audit.yml@v1
```

Repositories with no license policy pass `licenses: false`; `coverage: false`
skips the coverage build.

The `Go <version>` job is the MSRV job's counterpart, and it needs the same care:
it sets `GOTOOLCHAIN=local` so the `toolchain` directive in `go.mod` cannot pull
a newer compiler and quietly build on that instead — the exact failure mode
`RUSTUP_TOOLCHAIN` prevents on the Rust side. It is the only job that ignores
that directive, and so the only one testing the `go` floor at all.

Every other job takes its toolchain from `go.mod`, not from `stable`. That is
deliberate and it is the one place Go differs from Rust here: clippy ships with
the Rust toolchain, but `golangci-lint` is pinned separately by the calling
repository, and a pinned analyzer only understands the Go releases it was built
against. Its bundled staticcheck parses the standard library's own source, so a
new Go release makes it panic rather than merely miss a lint. Tracking `stable`
would turn every repository red the week Go ships, with no commit in any of
them. Moving to a new Go is a deliberate edit to `go.mod`.

**Pin the tag, not `@main`.** A change to `@main` lands in every repository at
once, with no pull request in any of them.

### Adding another ecosystem

Follow the same shape rather than adding a `language` input to these: toolchain
setup, cache action and tool installers are fully disjoint across ecosystems, and
the ecosystem-specific jobs stay separate anyway. `hmi-components-dioxus` settles
it — it is Rust, but gates on `dx check` and `dx build`, so even "Rust" is not
one shape.

Name them `<ecosystem>-ci.yml` and `<ecosystem>-audit.yml`, and keep the
`Gate N — …` job names so a red check reads the same in any repository. Every
gate job should reduce to invoking the repository's own runner recipe; anything
above that line is setup, and anything the recipe could own instead — a tool
version, a flag — belongs in the runner file, not here.

## Release workflows

| Workflow | Does |
|---|---|
| [`rust-bump-version.yml`](.github/workflows/rust-bump-version.yml) | Bumps `[workspace.package].version` from the commit type, commits, pushes a `v*` tag |
| [`go-bump-version.yml`](.github/workflows/go-bump-version.yml) | Derives the next version from the existing tags and pushes a `v*` tag |
| [`release-cloudrun.yml`](.github/workflows/release-cloudrun.yml) | `gcloud run deploy --source .` on a tag |

The two bump workflows are split by ecosystem for the same reason the CI ones
are, though the seam is different. The conventional-commit parser is identical;
what differs is that a Go module has no version field — SemVer git tags *are*
the version — so it writes nothing, while the Rust one edits `Cargo.toml`,
commits, and must then skip its own commit on the next push. Branching on that
inside a workflow that pushes to the default branch is worse than duplicating
twenty lines of parser.

`release-cloudrun.yml` is not split, because it genuinely is one shape:
`--source .` hands the repository to Cloud Build, which builds the root
Dockerfile, and nothing in the workflow knows what is inside it.

```yaml
name: Bump Version and Tag

on:
  push:
    branches: [main]

jobs:
  bump:
    uses: ninoverse/.github/.github/workflows/rust-bump-version.yml@v1
    with:
      app-id: ${{ vars.RELEASE_APP_ID }}
    secrets:
      app-private-key: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}
```

```yaml
name: Deploy to Cloud Run on Tag Push

on:
  push:
    tags: ["v[0-9]+.[0-9]+.[0-9]+"]

jobs:
  deploy:
    uses: ninoverse/.github/.github/workflows/release-cloudrun.yml@v1
    with:
      service: my-service
      project: ${{ vars.GCP_PROJECT }}
      region: ${{ vars.GCP_REGION }}
    secrets:
      gcp-service-account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
```

**Secrets do not cross into a called workflow on their own.** Every secret is
declared in the called workflow and passed explicitly by the caller, as above.
`secrets: inherit` would also work, but it hands over every secret the caller
has rather than the one the workflow asked for.

**A GitHub App is required, and `GITHUB_TOKEN` is not a substitute.** A tag
pushed with `GITHUB_TOKEN` deliberately does not trigger other workflows, so the
Cloud Run deploy watching for that tag would never fire. See
[CONTRIBUTING.md](CONTRIBUTING.md#releases) for what the app needs and which
organization variable and secret hold its credentials.

## Dependency updates

[`default.json`](default.json) is the shared Renovate preset, and
[`.github/workflows/renovate.yml`](.github/workflows/renovate.yml) runs Renovate
**once for the whole organization** rather than once per repository. Actions is
unmetered for public repositories, so updating the private ones costs nothing.

A repository opts in with a one-line `renovate.json`:

```json
{ "extends": ["github>ninoverse/.github"] }
```

Repositories without that file are skipped, not onboarded. Renovate is
manager-based rather than language-based — it detects `Cargo.toml`, `go.mod`,
`package.json`, `Dockerfile` and workflow files independently in each repository
— so one preset serves every ecosystem. Rules that name a Rust dependency simply
never match elsewhere.

Two consequences worth knowing before changing `default.json`:

- **One cron is a cadence floor.** Per-repository `schedule:` still narrows, but
  no repository updates more often than the central run fires.
- **One failure point.** A bad preset affects every repository at once, which is
  why the workflow re-runs on a push to `default.json` instead of waiting for
  Monday.

Setup is a GitHub App installed on the organization, with `RENOVATE_APP_ID` and
`RENOVATE_APP_PRIVATE_KEY` set at organization level. See
[`CONTRIBUTING.md`](CONTRIBUTING.md#one-time-setup).

## What cannot live here

GitHub only defaults the community health filenames above. `CODEOWNERS`,
`.gitignore`, `.editorconfig`, linter and formatter configs, and task runner
files are **not** defaultable and must be committed to each repository.
