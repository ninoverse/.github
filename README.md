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

**Contract:** the calling repository has a `justfile` exposing `fmt-check`,
`lint`, `test`, `deny` and `audit`. The recipe *names* are the contract, not the
cargo commands behind them — CI calls the recipe so each command has exactly one
definition. The justfile itself cannot be centralised, since neither `just` nor
`cargo` has a remote include, so adopting repositories copy it.

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

**Pin the tag, not `@main`.** A change to `@main` lands in every repository at
once, with no pull request in any of them.

### Adding another ecosystem

Follow the same shape rather than adding a `language` input to these: toolchain
setup, cache action and tool installers are fully disjoint across ecosystems, and
the ecosystem-specific jobs stay separate anyway. `hmi-components-dioxus` settles
it — it is Rust, but gates on `dx check` and `dx build`, so even "Rust" is not
one shape.

Name them `<ecosystem>-ci.yml` and `<ecosystem>-audit.yml`, keep the `Gate N — …`
job names so a red check reads the same in any repository, and keep every gate
job to a single `run:` line invoking the repository's own runner recipe —
`just` for Rust, `make` for Go.

## What cannot live here

GitHub only defaults the community health filenames above. `CODEOWNERS`,
`.gitignore`, `.editorconfig`, linter and formatter configs, and task runner
files are **not** defaultable and must be committed to each repository.
