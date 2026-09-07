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

## What cannot live here

GitHub only defaults the filenames above. `CODEOWNERS`, `.gitignore`,
`.editorconfig`, linter and formatter configs, and task runner files are **not**
defaultable and must be committed to each repository.

Workflows are not defaultable either. They can be *called* instead — reusable
workflows are added here separately.
