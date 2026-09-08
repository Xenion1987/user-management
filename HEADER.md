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

## Where users are defined

Ansible *replaces* a list when a more specific scope redefines it, so a single
variable cannot carry global, group and host users at once. The role reads three
lists and merges them by user name:

| Set in                   | Variable                       | Wins over     |
| ------------------------ | ------------------------------ | ------------- |
| `group_vars/all.yml`     | `user_management_users`        | -             |
| `group_vars/<group>.yml` | `user_management_group_users`  | global        |
| `host_vars/<host>.yml`   | `user_management_host_users`   | group, global |

A group or a single host can therefore add accounts - or redefine one global
account - without copying the whole list:

```yaml
# group_vars/all.yml
user_management_users:
  - name: ops
    state: present
    ssh_public_keys: ["ssh-ed25519 AAAAC3Nz... ops"]

# group_vars/dbservers.yml - adds one account for that group only
user_management_group_users:
  - name: postgres-admin
    state: present

# host_vars/db01.yml - gives `ops` a different shell on this host only
user_management_host_users:
  - name: ops
    state: present
    shell: /bin/sh
    ssh_public_keys: ["ssh-ed25519 AAAAC3Nz... ops"]
```

> [!IMPORTANT]
> Whole entries are replaced, not merged key by key. A redefining entry has to
> carry every key it wants - anything left out falls back to the role default,
> not to the value from the less specific level. That is why `ssh_public_keys`
> is repeated in the `host_vars` example above.

Users are processed in name order, which keeps runs deterministic. Names that
appear on more than one level are listed by a `debug` task, visible with `-v`.
