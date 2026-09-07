# Contributing

This is the organization-wide default. GitHub serves it to every `ninoverse`
repository that does not ship its own `CONTRIBUTING.md`; a repository with its
own file overrides this one entirely, so read that instead when it exists.

What follows is process — branches, commits, pull requests. It is deliberately
free of build commands, because those differ per repository. **Each repository's
README is authoritative for how to build, test and verify it.**

## The loop

One branch, one commit, one pull request, merged before the next begins.

The defining constraint: **no stacked PRs.** Every branch is cut from an
up-to-date `main` and merged before the next one is cut. Only one branch is ever
in flight.

```bash
# 1. Start from an up-to-date main
git switch main
git pull --ff-only

# 2. Cut the branch
git switch -c <type>/<short-description>

# 3. Make the change, then run this repository's verification gate
#    (see its README — `just ci`, `make ci`, or the equivalent)

# 4. One commit
git add <the files this change touches>
git commit

# 5. Push and open the PR
git push -u origin <type>/<short-description>
```

Work too large for one commit is split into a **sequence** of PRs, not a stack.
Each does one thing, passes the gate on its own, and is merged before the next
begins. Order them so every PR leaves `main` green — a PR that needs a later PR
to build is in the wrong position.

### Hard rules

| Rule | Why |
|------|-----|
| Cut every branch from `main` | A branch cut from another branch is a stacked PR |
| Never start change N+1 before N is merged | Same reason; only one branch in flight |
| One commit per branch | The PR is the review unit; a merged PR is one commit on `main` |
| Never push to `main` | `main` only advances through merged PRs |
| Delete the branch after merging | A squash merge rewrites the commit, so the local branch is not an ancestor of `main` and will not be cleaned up for you |

## Branch names

```
<type>/<short-description>
```

Lowercase, hyphen-separated, two to five words. No ticket numbers unless the
repository tracks them.

| Prefix | When to use |
|--------|-------------|
| `feat/` | New feature |
| `fix/` | Bug fix |
| `refactor/` | Refactor with no behavior change |
| `chore/` | Tooling, deps, CI, config |
| `docs/` | Documentation only |

## Commits

[Conventional Commits](https://www.conventionalcommits.org/).

```
<type>(<scope>): <description>

[optional body]
```

- Subject line: max 72 characters, lowercase, imperative mood, no trailing period
- Body: wrap at 72 characters, explain *why* rather than *what*
- Append `!` after the type for a breaking change: `feat!: drop the v1 endpoint`

Types are the branch prefixes above plus `style`, `perf`, `test` and `revert`.
Scope is the module, package or layer being changed — whatever that means in
this repository — or one of `ci`, `deps`, `config`.

Avoid: vague subjects (`fix stuff`, `update`, `wip`), unrelated changes in one
commit, and anything resembling a credential.

## Pull requests

Title follows the commit format, under 72 characters. The body follows the
repository's pull request template.

- One logical change per PR, in one commit
- Branch from an up-to-date `main`, so no rebase is needed before review
- The verification gate passes **before** the branch is pushed
- If the PR establishes a new pattern, link the rule file or doc that records it

Because a branch is only pushed once the gate already passes, there is no
work-in-progress state to represent — draft PRs are not used.

| Lines changed | Action |
|--------------|--------|
| < 200 | Normal review |
| 200 – 600 | Say in the description where to start reading |
| > 600 | Consider splitting; at minimum, justify the size |

## Dependency updates

Renovate runs centrally for the whole organization from
[`ninoverse/.github`](https://github.com/ninoverse/.github) — one scheduled run
covers every repository. A repository opts in by committing a one-line
`renovate.json`:

```json
{ "extends": ["github>ninoverse/.github"] }
```

Repositories without that file are **skipped**, not onboarded. Nothing is done
to a repository that has not asked for it.

The shared policy: everything non-breaking arrives as one grouped PR on Monday,
majors wait for approval on the Dependency Dashboard issue, and security fixes
ignore the schedule entirely. Review the changelog rather than rubber-stamping —
a dependency raising *its own* minimum toolchain version is the usual reason a
green repository suddenly goes red.

### One-time setup

Authentication is a **GitHub App** installed on the organization, not a personal
access token. `GITHUB_TOKEN` cannot do this job twice over: it reaches only the
repository it runs in, and pull requests opened with it deliberately do not
trigger other workflows, so CI would never run on a dependency PR — the one
thing that makes these safe to merge.

The app needs these repository permissions:

| Permission | Access |
|---|---|
| Contents | Read and write |
| Pull requests | Read and write |
| Issues | Read and write *(for the Dependency Dashboard)* |
| Workflows | Read and write *(to update `.github/workflows/`)* |

Install it across the organization, then set both of these **at organization
level** so no repository duplicates them:

- `RENOVATE_APP_ID` — an organization *variable*
- `RENOVATE_APP_PRIVATE_KEY` — an organization *secret*

The workflow fails immediately with a named error if either is missing, rather
than failing obscurely inside Renovate. To try it without side effects, run it
from the Actions tab with **dryRun** checked: it logs what it would do across
every repository and opens nothing.

## Releases

Versions are not chosen by hand. A push to `main` is read as a Conventional
Commit and bumped accordingly — `feat` minor, `fix`/`perf`/`refactor`/`chore`
and friends patch, a `!` or `BREAKING CHANGE` major — and a `v*` tag is pushed.
Anything else, including a merge commit that does not follow the format, bumps
nothing. So the commit convention above is not only documentation: it decides
the version number.

Repositories opt in by calling the shared workflows; see the
[README](README.md#release-workflows) for the caller files. Deployment, where a
repository has one, triggers off that tag rather than off the push.

### One-time setup

A second **GitHub App**, separate from Renovate's. Both push, but this one
pushes to `main`, and a dependency bot should not hold that.

| Permission | Access | Why |
|---|---|---|
| Contents | Read and write | Push the release commit and the tag |

Install it across the organization, then set both **at organization level**:

- `RELEASE_APP_ID` — an organization *variable*
- `RELEASE_APP_PRIVATE_KEY` — an organization *secret*

`GITHUB_TOKEN` cannot stand in. A tag pushed with it deliberately does not
trigger other workflows, so a deploy watching for that tag would never fire.

If `main` is protected, the app also needs permission to bypass the rule — a
push ruleset with the app in its bypass list. Without that, every bump run fails
at the push and nothing is ever tagged.

The workflows fail immediately with a named error if either credential is
missing, rather than failing obscurely inside the token action.
