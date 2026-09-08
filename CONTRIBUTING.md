# Contributing

## Setup

```sh
uv sync --group dev
prek install
```

`prek install` wires up the hooks that regenerate `README.md` and run the
linters, so local commits and CI check the same things.

## Before you push

```sh
uv run yamllint .
uv run ansible-lint
uv run molecule test
```

`molecule test` defaults to Debian 13. Override the target with
`MOLECULE_DISTRO=rockylinux9` (or `ubuntu2404`) to reproduce a specific CI job.
CI runs every distro against ansible-core 2.20/Python 3.13 and 2.21/Python 3.14.

## Documentation

`README.md` is generated - **do not edit it**.

- Prose, badges and guides live in `HEADER.md`.
- Variable documentation lives as `@var` annotations next to the variables in
  `defaults/main.yml`.

Regenerate with `uv run ansible-doctor` (the prek hook does this automatically
when `defaults/`, `meta/`, `HEADER.md` or `.ansibledoctor.yml` change).

Valid `@var` subtypes: `description`, `type`, `required`, `value`, `example`,
`deprecated`. There is no `:default:` subtype, and the short form
`# @var name: text` is invalid - it writes to `:value:`, collides with the
auto-detected value and makes `ansible-doctor` exit non-zero.

## Commits determine the release

The release pipeline derives the version bump from **every commit that landed on
`main` since the last release tag**, and applies the strongest one:

| Commit subject (or body)                              | Bump  |
| ----------------------------------------------------- | ----- |
| `major:` , any type with `!` (`feat!:`, `fix(api)!:`) | Major |
| `BREAKING CHANGE` / `BREAKING-CHANGE` in the body     | Major |
| `feat:` , `feat(scope):`                              | Minor |
| `fix:` , `fix(scope):`                                | Patch |
| anything else                                         | none  |

With squash merges that means the PR title is what counts. Type prefixes match
case-insensitively and accept scopes; `BREAKING CHANGE` is case-sensitive per
spec. Only the subject decides the type, so `* fix: ...` bullets in a squash
body do not each trigger a release.

Because the range is "since the last tag" rather than "the pushed commit", a
prefix is never lost to a multi-commit push or a merge commit - it stays in
range and the next run picks it up.

A commit without a recognised prefix produces no release, which is the right
outcome for `chore:`, `docs:`, `ci:` and `refactor:`. Every run records what it
decided, and why, in its job summary.

To release manually - or to recover one that was never cut - use
*Actions -> Release Pipeline -> Run workflow*: `bump` forces a level, `dry_run`
(on by default) prints the plan without tagging.

Tag and GitHub release are always created for a recognised prefix. Only the
Ansible Galaxy import needs `GALAXY_API_KEY`; without it that step logs why it
skips and the release itself still happens.

## Supported versions

Only versions that CI actually tests may be advertised. If you change the test
matrix in `.github/workflows/ci.yml`, update both:

- `min_ansible_version` in `meta/main.yml` (currently `2.20`)
- the `ansible-core` floor in `pyproject.toml`
