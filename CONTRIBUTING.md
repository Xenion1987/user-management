# Contributing

## Setup

```sh
uv sync --group dev
prek install
```

`prek install` wires up the hooks that regenerate `README.md` and run the
linters, so local commits and CI check the same things.

| Tool                                 | Required         | Purpose                                       |
| ------------------------------------ | ---------------- | --------------------------------------------- |
| [uv](https://docs.astral.sh/uv/)     | yes              | Manages the Python version and dependencies   |
| [Podman](https://podman.io/)         | yes              | Container runtime for the Molecule tests      |
| [prek](https://github.com/j178/prek) | no (recommended) | Git hook manager, drop-in pre-commit stand-in |
| [direnv](https://direnv.net/)        | no               | Auto-loads the virtualenv on `cd`             |

## Repository layout

| Path                 | Purpose                                             |
| -------------------- | --------------------------------------------------- |
| `tasks/`             | Role tasks                                          |
| `defaults/`          | Default variables (lowest precedence) + `@var` docs |
| `vars/`              | Role-internal computed variables                    |
| `templates/`         | Jinja2 templates (`.j2`)                            |
| `meta/`              | Role metadata and argument specs                    |
| `molecule/`          | Test scenario                                       |
| `HEADER.md`          | Handwritten part of `README.md`                     |
| `.github/workflows/` | CI and release pipelines                            |

## Before you push

```sh
uv run yamllint .          # all YAML, including .github/workflows
uv run ansible-lint        # the role and the Molecule playbooks
uv run molecule test       # create, converge, idempotence, verify, destroy
```

`molecule test` defaults to Debian 13. Override the target with
`MOLECULE_DISTRO=rockylinux9` (or `ubuntu2404`) to reproduce a specific CI job.
CI runs every distro against ansible-core 2.20/Python 3.13 and 2.21/Python 3.14.

For a faster loop, `molecule converge` applies the role and keeps the container,
`molecule verify` re-runs only the assertions, `molecule destroy` cleans up.
Assertions belong in `molecule/default/verify.yml` - an empty verify stage makes
CI pass without checking anything.

The prek hooks run on every `git commit`:

| Hook                  | Does                                                   |
| --------------------- | ------------------------------------------------------ |
| `trailing-whitespace` | Strips trailing whitespace                             |
| `end-of-file-fixer`   | Ensures a single trailing newline                      |
| `ansible-doctor`      | Regenerates `README.md` from `HEADER.md` + `defaults/` |
| `ansible-lint`        | Lints the role and the Molecule playbooks              |
| `yamllint`            | Lints all YAML                                         |

The lint hooks run the same repo-wide commands as CI, so a green commit means a
green lint job.

## Documentation

`README.md` is generated - **do not edit it**.

- Prose, badges and usage guides live in `HEADER.md`.
- Variable documentation lives as `@var` annotations next to the variables in
  `defaults/main.yml`:

```yaml
# @var my_variable:description: What this variable controls
# @var my_variable:type: str
# @var my_variable:required: false
# @var my_variable:example: >
# my_variable: some-value
my_variable: "default-value"
```

Regenerate with `uv run ansible-doctor` (the prek hook does this automatically
when `defaults/`, `meta/`, `HEADER.md` or `.ansibledoctor.yml` change).

Valid `@var` subtypes: `description`, `type`, `required`, `value`, `example`,
`deprecated`. There is no `:default:` subtype, and the short form
`# @var name: text` is invalid - it writes to `:value:`, collides with the
auto-detected value and makes `ansible-doctor` exit non-zero.

`@var` cannot describe *nested* keys. The keys of a `user_management_users`
entry are therefore documented inside that variable's `:example:` block, which
ansible-doctor renders into README.md - so the reference lives next to the
variable it belongs to. `meta/argument_specs.yml` carries only role metadata;
populate its `options` again if runtime argument validation is wanted back.

## How the role is built

Per managed user the role issues three module calls (`user`, the `.ssh`
directory, the `authorized_keys` template), and the `.ssh` task is skipped for
users that enroll no keys. Groups are created from a flattened, de-duplicated
list of names, so shared groups cost one call instead of one per user.

There is deliberately no `include_tasks` inside a `loop`: a dynamic include is
rebuilt and re-templated by the controller for every item on every host, which
collapses once host counts grow. `import_tasks` is no way around it either,
being static and incompatible with `loop`.

`vars/main.yml` concatenates the global, group and host user lists and merges
them by name via `groupby`, so the most specific level wins and users are
processed in a deterministic order. A `debug` task with `verbosity: 1` reports
names that occur on more than one level.

The remaining cost scales with users x hosts. Three controller settings dominate
the wall time of a large rollout, none of them part of this role: `forks`
(default `5`), `pipelining = True` (needs `requiretty` disabled in sudoers) and
`strategy: free` - with the default `linear`, every task is a barrier across all
hosts. `ControlPersist` in `ssh_args` helps too. To find out where the time
actually goes, run a regular play with
`ANSIBLE_CALLBACKS_ENABLED=ansible.posix.profile_tasks`.

## CI pipeline

`.github/workflows/ci.yml` runs three jobs: **lint** (`yamllint` and
`ansible-lint` repo-wide), **test** (`molecule test` per distro and
ansible-core/Python pair) and **test-matrix**, a tiny aggregate job that fails
unless every matrix job succeeded.

| ansible-core | Python |
| ------------ | ------ |
| 2.20         | 3.13   |
| 2.21         | 3.14   |

The pairs are explicit because each `ansible-core` release supports a specific
range of controller Python versions.

> [!IMPORTANT]
> Branch protection must require exactly two checks: **`lint`** and
> **`test-matrix`**. Both names are stable, so adding a distribution or an
> ansible-core version never means editing the required-check list again. Do not
> require the individual `test (...)` contexts - renaming the matrix leaves them
> "expected forever" and blocks every pull request.

Actions are pinned to commit SHAs rather than moving tags; the release job holds
`contents: write` and sees the Galaxy token. Dependabot updates SHA pins just as
it does version tags.

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
range and the next run picks it up. The base version is the highest `v*` semver
tag in the repository, not the closest reachable one, so out-of-order or
branch-local tags cannot bump from the wrong base. Pre-release tags are not
supported.

A commit without a recognised prefix produces no release, which is the right
outcome for `chore:`, `docs:`, `ci:` and `refactor:`. Every run writes its plan
to the job summary - base version, commits inspected, resulting bump, target
version - so a deliberate no-op is distinguishable from a lost release.

To release manually - or to recover one that was never cut - use
*Actions -> Release Pipeline -> Run workflow*: `bump` forces a level, `dry_run`
(on by default) prints the plan without tagging.

Tag and GitHub release are always created for a recognised prefix. Only the
Ansible Galaxy import needs `GALAXY_API_KEY`; without it that step logs why it
skips and the release itself still happens. Release notes are generated by
GitHub and shaped by `.github/release.yml`, which groups pull requests and
excludes dependency bumps.

> [!NOTE]
> The release workflow is not gated on CI by itself - both workflows trigger
> independently, so the protection comes from the branch ruleset.

### Credentials and variables

Configured under *Settings -> Actions*:

| Name               | Kind     | Required | Description                                                            |
| ------------------ | -------- | -------- | ---------------------------------------------------------------------- |
| `GALAXY_API_KEY`   | Secret   | no       | Ansible Galaxy API token; only the Galaxy import is skipped without it |
| `GALAXY_NAMESPACE` | Variable | no       | Alternate Galaxy namespace (defaults to the repository owner)          |

`.github/dependabot.yml` keeps the GitHub Actions pins and the `uv.lock` dev
dependencies current, labelled so they stay out of the release notes.

## Supported versions

Only versions that CI actually tests may be advertised. If you change the test
matrix in `.github/workflows/ci.yml`, update both:

- `min_ansible_version` in `meta/main.yml` (currently `2.20`)
- the `ansible-core` floor in `pyproject.toml`
