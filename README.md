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
    uses: ninoverse/.github/.github/workflows/rust-ci.yml@v1.2.3
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
    uses: ninoverse/.github/.github/workflows/rust-audit.yml@v1.2.3
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
    uses: ninoverse/.github/.github/workflows/go-ci.yml@v1.2.3
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
    uses: ninoverse/.github/.github/workflows/go-audit.yml@v1.2.3
```

Repositories with no license policy pass `licenses: false`; `coverage: false`
skips the coverage build.

Every job takes its toolchain from `go.mod`, not from `stable`. That is
deliberate, and it is the one place Go differs from Rust here: clippy ships with
the Rust toolchain, but `golangci-lint` is pinned separately by the calling
repository, and a pinned analyzer only understands the Go releases it was built
against. Its bundled staticcheck parses the standard library's own source, so a
new Go release makes it panic rather than merely miss a lint. Tracking `stable`
would turn every repository red the week Go ships, with no commit in any of
them. Moving to a new Go is a deliberate edit to `go.mod`.

Note that `setup-go@v5` reads only the **`go` directive** from `go.mod` — it
matches `/^go (\d+(\.\d+)*)/` and never looks at `toolchain`. The action's
README on `main` says otherwise; that describes a later version than the one
pinned here.

The `Go <version>` job looks like the MSRV job and is not its counterpart. Rust
needs that job because nothing on the stable toolchain enforces `rust-version`;
Go enforces the `go` directive on every build, in every job, so the floor is
already covered. What that job adds is the two things that are not:

- Under the default `GOTOOLCHAIN=auto`, a dependency requiring a newer Go makes
  the toolchain silently download and use it — the gates stay green while the
  declared floor has quietly become a lie. `GOTOOLCHAIN=local` turns that into a
  failure.
- Passing `go-version` explicitly, rather than reading `go.mod` a seventh time,
  checks the caller's declared floor against the file. They are the same number
  from two sources, so drift between them surfaces here rather than going
  unnoticed.

### Versioning

These workflows are a dependency of every repository that calls them, so they
carry version numbers like any other dependency. Callers pin a full one:

```yaml
uses: ninoverse/.github/.github/workflows/rust-ci.yml@v1.2.3
```

`v1.2.3` is a placeholder, here and in every example below and in each workflow
file's own header. Renovate rewrites `jobs.<id>.uses`; it does not rewrite prose
or comments, so a real version written into an example would go stale and stay
stale. A caller pins a version that exists.

[`bump-version.yml`](.github/workflows/bump-version.yml) cuts the tags on every
push to `main`, from the commit subject, by the same conventional-commit rules
every other repository here releases by. Renovate opens the bump in each caller,
in a group of its own rather than the weekly actions PR — a bump here changes
what the gates *are*, not which version of `actions/checkout` runs them.

**Pin a full version, not a short one.** `@v1` looks like a pin and behaves like
neither thing it resembles. Renovate's `github-actions` versioning prefers the
shortest tag that works, so given `@v1` and a release of `v1.5.3` it resolves
the upgrade and writes `v1` back unchanged; a short ref moves only when a `v2`
appears. It is not a floating tag and not a maintained pin — it is a ref that
quietly stops receiving anything, which is how `v1` here came to sit four
commits behind `main` with no pull request anywhere to show for it.

**Pin a version, not `@main`.** A change to `@main` lands in every repository at
once, with no pull request in any of them.

`v1` still exists and still points at `f612c6f`. It stays there: a landing spot
for repositories not yet moved onto a version, not a major that tracks anything.
A repository still on `@v1` is receiving nothing.

#### What counts as breaking

A version number is only worth reading if it means something, and the
non-obvious half of the answer is that a called workflow's API is wider than its
`inputs:`.

**Major** — the caller has to change, or its branch protection does:

- Removing or renaming an input or secret, or making an optional input required.
- Changing an input's default.
- Renaming a workflow file.
- Renaming a job. A job's `name:` becomes the status check name in every caller,
  and branch protection names its required checks as strings. `Gate 1 — fmt` is
  part of the interface, not a label.
- Needing a permission the caller did not have to grant before. A called
  workflow can only narrow the permissions it is handed, never raise them, so a
  new `contents: write` requirement is a change every caller has to make — the
  `permissions` block on the `rust-release.yml` caller below is one.

**Minor** — a new optional input, a new job, a new output.

**Patch** — everything else, comments and documentation included.

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
| [`rust-release.yml`](.github/workflows/rust-release.yml) | Builds one static binary per target on a tag and publishes them with generated notes |
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

`rust-release.yml` is the other half of a tag for a repository that ships a
binary rather than a deployment: the bump workflow pushes the tag, this one
builds it. Its contract is the runner recipe like the CI workflows' is —
`just dist <target>` — and it derives each target's runner itself, because
every target is built on the architecture it runs on, and a caller pairing the
two by hand could only get it wrong. Assets are named `<binary>-<target>` and
nothing else, since the tag is already in the download URL and anything
fetching one builds that URL by hand.

The `extra-notes` input is markdown the calling repository computes about
itself, prepended above the generated changelog. It is an input and never a
script this workflow runs out of the caller: a shared workflow executing
repository-supplied code inside the run holding the organization's token widens
the blast radius to every repository at once.

```yaml
name: Bump Version and Tag

on:
  push:
    branches: [main]

jobs:
  bump:
    uses: ninoverse/.github/.github/workflows/rust-bump-version.yml@v1.2.3
    with:
      app-id: ${{ vars.RELEASE_APP_ID }}
    secrets:
      app-private-key: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}
```

```yaml
name: Release

on:
  push:
    tags: ["v[0-9]+.[0-9]+.[0-9]+"]

# Belongs in the caller: a re-run of the same tag must not race the first, and
# it queues rather than cancels because cancelling between the upload and the
# release leaves a tag with some of its assets and no release to hold them.
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false

jobs:
  release:
    # Required, and it goes on the calling job. A called workflow can only
    # downgrade the permissions it is handed, never raise them, so the
    # `contents: write` the release job needs to publish is capped by whatever
    # the caller grants. Leave this out and the run builds every target and
    # then fails the publish with a 403.
    permissions:
      contents: write
    uses: ninoverse/.github/.github/workflows/rust-release.yml@v1.2.3
    with:
      binary: my-tool
      targets: x86_64-unknown-linux-musl,aarch64-unknown-linux-musl,aarch64-apple-darwin
```

This is the one workflow here that needs a write permission, which is why it is
the only caller example that carries a `permissions:` block. A workflow-level
`permissions: contents: read` in the caller is fine, and does not have to be
removed — a job-level block replaces it rather than being capped by it.

A repository with something to say about its own release computes it in a job
of its own and passes it through:

```yaml
jobs:
  notes:
    runs-on: ubuntu-latest
    outputs:
      extra: ${{ steps.notes.outputs.extra }}
    steps: [...]

  release:
    needs: notes
    permissions:
      contents: write
    uses: ninoverse/.github/.github/workflows/rust-release.yml@v1.2.3
    with:
      binary: my-tool
      targets: x86_64-unknown-linux-musl
      extra-notes: ${{ needs.notes.outputs.extra }}
```

```yaml
name: Deploy to Cloud Run on Tag Push

on:
  push:
    tags: ["v[0-9]+.[0-9]+.[0-9]+"]

jobs:
  deploy:
    uses: ninoverse/.github/.github/workflows/release-cloudrun.yml@v1.2.3
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

Four consequences worth knowing before changing `default.json`:

- **One cron is a cadence floor.** Per-repository `schedule:` still narrows, but
  no repository updates more often than the central run fires.
- **One failure point.** A bad preset affects every repository at once, which is
  why the workflow re-runs on a push to `default.json` instead of waiting for
  Monday.
- **The window is the whole of Monday, and has to be.** `schedule:` is evaluated
  when a branch would be created, not when the run fires. A GitHub cron is a
  request, not a promise — scheduled workflows are delayed under load, and the
  one run that has ever fired that way started at 09:26 against a `0 4 * * 1`
  cron. Against the four-hour window this preset used to carry, that run could
  create nothing at all. Renovate's own documentation warns off the shape:
  *"Avoid schedules like 'Run Renovate for an hour each Sunday' as you will run
  into problems."* Narrowing the window again re-creates the bug.
- **A re-run on a Monday delivers; on any other day it only validates.** Outside
  the window a push-triggered run extracts dependencies and opens nothing, so a
  config mistake still surfaces immediately while pull requests wait. On a
  Monday there is no such separation — a push to `default.json` opens whatever
  is due, org-wide. To pull one update forward on another day, tick its checkbox
  under *Awaiting Schedule* on that repository's Dependency Dashboard and re-run
  the workflow.

### agentcfg

A repository whose agent rule files are composed by
[`agent-config-sync`](https://github.com/ninoverse/agent-config-sync) pins the
release it generated them from, as `config_version` in `.agentprofile.yml`. No
built-in manager reads that file, so a custom manager picks the pin up by regex
— the same annotation-free technique that keeps the pinned Renovate version
above current.

It gets a group of its own rather than joining the weekly non-major pull
request. A bump there rewrites the prose an agent obeys, not code it calls,
and folded in among cargo updates it would be one line of churn in a pull
request nobody reads closely — the exact failure the pin exists to prevent.
Majors need nothing extra: the shared rule already holds every major behind
the Dependency Dashboard.

The whole pin is matched including its leading `v`, because that `v` is part of
the tag the release download URL is built from.

Matches nothing in a repository with no `.agentprofile.yml`, which today is
every repository but one.

Bumping the pin alone would leave `main` inconsistent: the profile would name a
release the composed files were not generated from, and the repository's own
`agentcfg check` gate would fail on the very pull request the bot opened. So a
`postUpgradeTasks` entry regenerates them *before the commit is made* —
`executionMode: branch`, so it runs once per branch rather than once per
dependency. The new pin and the files it produces land in one commit.

The commands get no shell, so they are three bare invocations rather than a
pipeline: fetch the pinned binary, mark it executable, run `sync`.
`--create-dirs` is what keeps that at three commands instead of four. The tag
comes from `{{{newValue}}}`, triple-braced so it is not HTML-escaped — and it
resolves correctly here only because the rule above gives agentcfg a group of
its own, since in branch mode the template reads the branch's *first* upgrade.

`allowedCommands` in
[`renovate.yml`](.github/workflows/renovate.yml) is what permits them, and it is
a **global-only** option: a repository cannot add to it from its own
`renovate.json`, so extending this preset can never make the central run execute
something the organization has not already allowed. Each pattern is anchored by
hand, because Renovate tests them unanchored against the command *after* its
template is compiled.

**`.agentcfg/` must be gitignored in every repository that opts in.** Renovate
collects post-upgrade changes from `git status`, so a fetched binary that is not
ignored lands in the bump commit — three megabytes of it, in a pull request
about prose.

#### Auto-merging agentcfg

A repository extending only the base preset auto-merges nothing, which is the
right default for rules a person should read. Three presets relax that, chosen
by a second line in `renovate.json`:

| Preset | Auto-merges |
|---|---|
| `github>ninoverse/.github:agentcfg-automerge-never` | Nothing. The default, written down. |
| `github>ninoverse/.github:agentcfg-automerge-patch` | Patch only |
| `github>ninoverse/.github:agentcfg-automerge-minor` | Patch and minor |

```json
{
  "extends": [
    "github>ninoverse/.github",
    "github>ninoverse/.github:agentcfg-automerge-patch"
  ]
}
```

Majors need nothing added, and no preset here can auto-merge one: the base
preset holds every major behind the Dependency Dashboard. CI still gates every
auto-merge, `agentcfg check` included, so a bump that regenerated badly is never
merged unattended.

This is why the version number has to mean something for prose, and it does.
**Patch** is wording, examples or clarification — no rule changes meaning.
**Minor** adds a fragment, an axis value or a rule; it is additive, so nothing
an agent was already doing becomes wrong. **Major** reverses or removes a rule,
renames a fragment, or changes the profile schema — something a repository was
doing is now wrong, or its profile needs editing.

Setup is a GitHub App installed on the organization, with `RENOVATE_APP_ID` and
`RENOVATE_APP_PRIVATE_KEY` set at organization level. See
[`CONTRIBUTING.md`](CONTRIBUTING.md#one-time-setup).

## What cannot live here

GitHub only defaults the community health filenames above. `CODEOWNERS`,
`.gitignore`, `.editorconfig`, linter and formatter configs, and task runner
files are **not** defaultable and must be committed to each repository.
