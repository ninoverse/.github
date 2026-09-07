# Security Policy

This is the organization-wide default. It applies to every `ninoverse`
repository that does not ship its own `SECURITY.md`.

## Reporting a vulnerability

**Do not open a public issue.**

Use **Report a vulnerability** on the affected repository's Security tab, which
opens a private advisory visible only to the maintainers. If that button is not
present, private reporting has not been enabled on that repository — contact
[@nicolapasqua99](https://github.com/nicolapasqua99) directly instead and ask for
a private channel before sending details.

Please include what you can of: the affected repository and version or commit,
the impact, and the smallest reproduction you have. A partial report is still
worth sending.

Expect an acknowledgement within a week. If the report is valid you will get an
estimated fix timeline, and credit in the published advisory unless you ask
otherwise.

## Supported versions

Only the default branch of each repository is maintained. There are no release
branches to backport to. Projects created from a `ninoverse` template are the
responsibility of whoever created them — a fix here does not reach a fork.

## Scope

In scope, for any repository:

- A dependency or base image pinned to a version with a known advisory
- A CI workflow with excessive `permissions:`, or one that can execute untrusted
  input from a pull request
- A credential, token or key committed to history
- A default that weakens what the repository ships — a container running as
  root, a build leaking a secret into a layer, a lint or license gate that can be
  silently bypassed

Out of scope:

- Advisories against dependencies of *your* project rather than ours. Run the
  repository's own audit gate.
- The absence of a hardening measure that was never claimed. Open a feature
  request instead.
- Findings that require an attacker to already have write access to the
  repository.
