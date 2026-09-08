# Ansible role: user_management

[![CI](https://github.com/Xenion1987/ansible-role-user-management/actions/workflows/ci.yml/badge.svg)](https://github.com/Xenion1987/ansible-role-user-management/actions/workflows/ci.yml)
[![prek](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/j178/prek/master/docs/assets/badge-v0.json)](https://github.com/j178/prek)
[![Molecule](https://img.shields.io/badge/tested%20with-Molecule-blue.svg)](https://github.com/ansible-community/molecule)
[![Galaxy downloads](https://img.shields.io/ansible/role/d/xenion1987/user_management?label=Galaxy%20downloads&logo=ansible&color=%23096598)](https://galaxy.ansible.com/ui/standalone/roles/xenion1987/user_management)
[![License](https://img.shields.io/github/license/Xenion1987/ansible-role-user-management)](https://github.com/Xenion1987/ansible-role-user-management/blob/main/LICENSE)

> [!IMPORTANT]
> **`README.md` is generated - do not edit it.** The handwritten part lives in
> `HEADER.md`; the variable reference below it is built from the `@var`
> annotations in `defaults/main.yml`. Run `uv run ansible-doctor` to regenerate
> it, or just commit - a prek hook does it for you.

Manage users and their SSH public key enrollment via Ansible on Linux systems.

---

## Example playbook

```yaml
- name: "Play | user_management"
  hosts: all
  roles:
    - role: xenion1987.user_management
      vars:
        user_management_default_ssh_from: ["10.0.0.0/8"]
        user_management_users:
          - name: john.doe
            gecos: John Doe,Room 123,212-555-0000,212-555-3456,john.doe@world.org
            state: present
            ssh_public_keys:
              - "ssh-ed25519 AAAAC3Nz... john.doe"
```

Accounts that only exist on a single host belong in
`user_management_host_users`, set in that host's `host_vars`. That list is
merged on top of `user_management_users`, so a host can add accounts without
copying or overriding the shared list.

---

## Rolling out to many hosts

Per managed user the role issues three module calls (`user`, the `.ssh`
directory, the `authorized_keys` template), plus one `group` call per
*distinct* group rather than per user - group names are flattened and
de-duplicated first. The role also contains no `include_tasks` inside a `loop`:
a dynamic include is rebuilt and re-templated by the controller for every item
on every host, which is what makes that pattern collapse once host counts grow.
`import_tasks` is no way around it either, being static and incompatible with
`loop`.

The remaining cost scales with users x hosts, and there three controller
settings dominate the wall time - none of them part of this role:

| Setting | Why it matters |
| --- | --- |
| `forks` (default `5`) | 200 hosts are otherwise worked through in 40 sequential waves |
| `pipelining = True` | drops several SSH round trips per module call; needs `requiretty` disabled in sudoers |
| `strategy: free` | with the default `linear`, every single task is a barrier across all hosts |

Adding `-o ControlMaster=auto -o ControlPersist=60s` to `ssh_args` helps as
well, as does keeping fact gathering small (`gather_subset: min` or a fact
cache) - this role needs no facts of its own.

To find out where the time actually goes in your environment, run a regular
play with `ANSIBLE_CALLBACKS_ENABLED=ansible.posix.profile_tasks`.

---

## Per-user keys

Every entry of `user_management_users` and `user_management_host_users` accepts
the keys below. They are documented here rather than in the generated section,
because `@var` annotations cannot describe nested keys.

| Key | Type | Required | Choices | Default | Description |
| --- | --- | --- | --- | --- | --- |
| `absolute_home_path` | `str` | `false` |  |  | Custom `$HOME` root path. Must be specified as absolute path. |
| `custom_ssh_from` | `list` | `false` |  | `[]` | `from=""` value added to `authorized_keys` if user has `user_management_users.ssh_public_keys` defined. If `user_management_default_ssh_from` or `custom_ssh_from` is defined and not set to `'*'`, all values will be concatenated. |
| `gecos` | `str` | `false` |  |  | Optionally sets the description (aka GECOS) of user account: Full Name, Room Number, Work Phone, Home Phone, Other |
| `groups_append` | `bool` | `false` | `False`, `True` | `False` | If `true`, add the user to the groups specified in groups. If `false`, user will only be added to the groups specified in `secondary_groups`, removing them from all other groups. |
| `home_create` | `bool` | `false` | `False`, `True` | `True` | Unless set to false, a home directory will be created for the user when the account is created or if the home directory does not exist. |
| `home_move` | `bool` | `false` | `False`, `True` | `False` | If set to `true` when used with `home:`, attempt to move the user's old home directory to the specified directory if it isn't already there and the old home exists. |
| `name` | `str` | `true` |  | `user_management_john.doe` | User's Linux login name. |
| `password` | `str` | `false` |  |  | If provided, set the user's password to the provided encrypted hash password. To create an account with a locked/disabled password, set this to `!` or `*`. How to generate encrypted passwords: [Ansible Documentation](https://docs.ansible.com/ansible/latest/reference_appendices/faq.html#how-do-i-generate-encrypted-passwords-for-the-user-module) |
| `primary_group` | `str` | `false` |  | `user_management_users.name` | Optionally sets the user's primary group (takes a group name). |
| `secondary_groups` | `list` | `false` |  | `[]` | List of groups user will be added to. By default, the user is removed from all other groups. Configure `groups_append` to modify this. When set to an empty string `''`, the user is removed from all groups except the primary group. |
| `shell` | `str` | `false` |  | `user_management_default_shell` | Overwrites 'user_management_default_shell'. |
| `ssh_public_keys` | `list` | `false` |  | `[]` | The SSH public key(s), as a list or (since Ansible 1.9) url. |
| `state` | `str` | `true` | `absent`, `present` | `present` | Whether the account should exist or not, taking action if the state is different from what is stated. |
| `userdel_force` | `bool` | `false` | `False`, `True` | `False` | This only affects `state=absent`. It forces removal of the user and associated directories on supported platforms. |
| `userdel_remove` | `bool` | `false` | `False`, `True` | `False` | This only affects `state=absent`. It attempts to remove directories associated with the user. |

The `from=""` restriction written into `authorized_keys` is the concatenation of
`user_management_default_ssh_from`, `user_management_group_ssh_from`,
`user_management_host_ssh_from` and the entry's own `custom_ssh_from`.

---

## Prerequisites

| Tool                                 | Required         | Purpose                                                                    |
| ------------------------------------ | ---------------- | -------------------------------------------------------------------------- |
| [uv](https://docs.astral.sh/uv/)     | yes              | Python package and project manager (manages Python versions automatically) |
| [Podman](https://podman.io/)         | yes              | Container runtime for Molecule tests                                       |
| [prek](https://github.com/j178/prek) | no (recommended) | Fast Git hook manager (drop-in pre-commit alternative)                     |
| [direnv](https://direnv.net/)        | no               | Auto-load environment variables                                            |

---

## Development Workflow

### Directory structure

| Path                 | Purpose                                             |
| -------------------- | --------------------------------------------------- |
| `tasks/`             | Main role tasks                                     |
| `handlers/`          | Handlers triggered by tasks                         |
| `defaults/`          | Default variables (lowest precedence) + `@var` docs |
| `vars/`              | OS-specific or internal variables                   |
| `files/`             | Static files to deploy                              |
| `templates/`         | Jinja2 templates (`.j2`)                            |
| `meta/`              | Role metadata and argument specs                    |
| `molecule/`          | Test scenarios                                      |
| `HEADER.md`          | Handwritten part of `README.md`                     |
| `vars/`              | Role-internal computed variables                    |
| `.github/workflows/` | CI and release pipelines                            |

### Linting

```sh
uv run yamllint .          # YAML syntax and style (incl. .github/workflows)
uv run ansible-lint        # Best practices for the role and the Molecule playbooks
```

### Testing

```sh
uv run molecule test       # Full test: create, converge, idempotence, verify, destroy
uv run molecule converge   # Only apply the role (keep container running)
uv run molecule verify     # Run verification steps
uv run molecule destroy    # Tear down containers
```

Pick the target distribution with `MOLECULE_DISTRO` (default `debian13`):

```sh
MOLECULE_DISTRO=rockylinux9 uv run molecule test
```

Assertions belong in `molecule/default/verify.yml`, which runs as part of
`molecule test`. Keep at least one real assertion there - an empty verify stage
makes CI pass without checking anything.

### Git hooks (prek)

After installing dev dependencies, install the hooks:

```sh
prek install
```

The hooks then run on every `git commit`:

| Hook                  | Does                                                   |
| --------------------- | ------------------------------------------------------ |
| `trailing-whitespace` | Strips trailing whitespace                             |
| `end-of-file-fixer`   | Ensures a single trailing newline                      |
| `ansible-doctor`      | Regenerates `README.md` from `HEADER.md` + `defaults/` |
| `ansible-lint`        | Lints the role and the Molecule playbooks              |
| `yamllint`            | Lints all YAML                                         |

The lint hooks run the same repo-wide command as CI, so a green commit means a
green lint job.

### Documenting variables

`README.md` is assembled by [ansible-doctor](https://ansible-doctor.geekdocs.de/)
from `HEADER.md` plus `@var` annotations in `defaults/main.yml`:

```yaml
# @var my_variable:description: What this variable controls
# @var my_variable:type: str
# @var my_variable:required: false
# @var my_variable:example: >
# my_variable: some-value
my_variable: "default-value"
```

Valid subtypes are `description`, `type`, `required`, `value`, `example` and
`deprecated`. Two things to avoid:

- There is **no `:default:` subtype** - the default value is read from the YAML
  below the annotation.
- Do **not** use the short form `# @var my_variable: some text`. It is parsed as
  `:value:` and collides with the auto-detected value, which makes
  `ansible-doctor` exit non-zero and blocks the commit.

`meta/argument_specs.yml` carries role-level metadata (`author`, `description`)
and is only populated with `options` when runtime argument validation
(type checks, required fields, choices) is needed.

Regenerate manually with:

```sh
uv run ansible-doctor
```

---

## CI/CD (GitHub Actions)

### CI Pipeline (`.github/workflows/ci.yml`)

1. **Lint** - runs `yamllint` and `ansible-lint` across the whole repository
2. **Test** (depends on lint) - runs `molecule test` on Debian 13, Rocky Linux 9
   and Ubuntu 24.04, against each supported Ansible/Python pair:

   | ansible-core | Python |
   | ------------ | ------ |
   | 2.20         | 3.13   |
   | 2.21         | 3.14   |

   The pairs are explicit because each `ansible-core` release supports a
   specific range of controller Python versions. Keep the oldest pair in sync
   with `min_ansible_version` in `meta/main.yml` and the `ansible-core` floor in
   `pyproject.toml` - only tested versions should be advertised as supported.

3. **test-matrix** - a tiny aggregate job that fails unless every matrix job
   succeeded

Branch protection should require exactly two checks: **`lint`** and
**`test-matrix`**. Both names are stable, so adding a distribution or an
ansible-core version never means editing the required-check list again. Do not
require the individual `test (...)` contexts - renaming the matrix silently
leaves them "expected forever" and blocks every pull request.

No manual changes are required - the role name is derived from the repository
name automatically.

### Release Pipeline (`.github/workflows/release.yml`)

Tag and GitHub release are always created when a commit carries a release
prefix. Publishing to Ansible Galaxy is the only part that needs credentials and
is skipped without them, so the pipeline is useful in a repository that has no
Galaxy presence at all.

A push to `main` cuts a release automatically. The bump is derived from **every
commit since the last release tag**, not just the pushed commit - so a
multi-commit push, a merge-commit strategy, or a run that GitHub drops from the
concurrency queue cannot silently lose a release. The strongest signal wins:

| Commit subject (or message body)                      | Bump  |
| ----------------------------------------------------- | ----- |
| `major:` , any type with `!` (`feat!:`, `fix(api)!:`) | Major |
| `BREAKING CHANGE` / `BREAKING-CHANGE` in the body     | Major |
| `feat:` , `feat(scope):`                              | Minor |
| `fix:` , `fix(scope):`                                | Patch |
| anything else (`chore:`, `docs:`, `ci:`, `refactor:`) | none  |

Type prefixes match case-insensitively; `BREAKING CHANGE` is matched
case-sensitively, as the Conventional Commits spec defines it. Only the commit
*subject* decides the type, so the `* fix: ...` bullets that squash merges put
in the body do not trigger releases of their own.

The base version is the **highest** `v*` semver tag in the repository, not the
closest reachable one, so out-of-order or branch-local tags cannot bump from the
wrong base. Non-semver tags (`nightly-...`) are filtered out before any
arithmetic. Pre-release tags are deliberately not supported - if you need `-rc`
versions, replace this workflow with a dedicated tool rather than extending it.

**Every run writes a plan to the job summary** - base version, how many commits
were inspected, the resulting bump, the target version and which commits drove
it. A run that deliberately produces no release says so, instead of being an
indistinguishable green check.

**Manual runs**: *Actions -> Release Pipeline -> Run workflow* takes two inputs:

| Input     | Default | Purpose                                                      |
| --------- | ------- | ------------------------------------------------------------ |
| `bump`    | `auto`  | Force `patch`/`minor`/`major`, e.g. to recover a lost release |
| `dry_run` | `true`  | Print the plan without tagging or publishing                  |

`dry_run` defaults to on, so a manual trigger can never release by accident.

Release notes are generated by GitHub and shaped by `.github/release.yml`, which
groups PRs into categories and excludes dependency bumps.

> [!NOTE]
> The release workflow is not gated on CI by itself - both workflows trigger
> independently, so the protection comes from the branch. A ruleset on `main`
> requiring the `lint` and `test-matrix` status checks, linear history, and no
> force-push or deletion covers it. Add a pull-request requirement on top if you
> want to rule out direct pushes entirely.

After the release, the role is imported to Ansible Galaxy - but only if
`GALAXY_API_KEY` is configured. Without it the step logs why it is skipping and
succeeds, so tag and GitHub release still happen. The import references the
default branch because Galaxy discovers a role's versions from the repository's
tags.

Both workflows pin their actions to commit SHAs rather than moving tags; the
release job holds `contents: write` and sees the Galaxy token. Dependabot
updates SHA pins just as it does version tags.

### Secrets and Variables

Configure these under **Settings -> Secrets and variables -> Actions**:

| Name               | Type     | Required | Description                                                              |
| ------------------ | -------- | -------- | ------------------------------------------------------------------------ |
| `GALAXY_API_KEY`   | Secret   | no       | Ansible Galaxy API token; only the Galaxy import is skipped without it   |
| `GALAXY_NAMESPACE` | Variable | no       | Alternate Galaxy namespace (defaults to the repository owner)            |

### Dependency updates

`.github/dependabot.yml` keeps the GitHub Actions pins and the `uv.lock` dev
dependencies current. Adjust the schedule or reviewers to taste.

---
