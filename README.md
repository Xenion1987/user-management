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

Manage users and their SSH public key enrollment via Ansible on Linux systems.

## Table of contents

- [Requirements](#requirements)
- [Default Variables](#default-variables)
  - [user_management_default_home_move](#user_management_default_home_move)
  - [user_management_default_home_root](#user_management_default_home_root)
  - [user_management_default_primary_group](#user_management_default_primary_group)
  - [user_management_default_secondary_groups](#user_management_default_secondary_groups)
  - [user_management_default_secondary_groups_append](#user_management_default_secondary_groups_append)
  - [user_management_default_shell](#user_management_default_shell)
  - [user_management_default_ssh_from](#user_management_default_ssh_from)
  - [user_management_group_ssh_from](#user_management_group_ssh_from)
  - [user_management_group_users](#user_management_group_users)
  - [user_management_host_ssh_from](#user_management_host_ssh_from)
  - [user_management_host_users](#user_management_host_users)
  - [user_management_users](#user_management_users)
- [Dependencies](#dependencies)
- [License](#license)
- [Author](#author)

---

## Requirements

- Minimum Ansible version: `2.20`

## Default Variables

### user_management_default_home_move

If set to `true` when used with `home`, attempt to move the user's old home directory to the specified directory if it isn't there already and the old home exists.

**_Required:_** false<br />
**_Type:_** bool<br />

#### Default value

```YAML
user_management_default_home_move: false
```

### user_management_default_home_root

Custom default `$HOME` root path. Omitted if `null`.

**_Required:_** false<br />
**_Type:_** str<br />

#### Default value

```YAML
user_management_default_home_root:
```

### user_management_default_primary_group

Custom default primary user group. Omitted if `null`.

**_Required:_** false<br />
**_Type:_** str<br />

#### Default value

```YAML
user_management_default_primary_group:
```

### user_management_default_secondary_groups

Custom default secondary user groups. Omitted if empty.

**_Required:_** false<br />
**_Type:_** list<br />

#### Default value

```YAML
user_management_default_secondary_groups: []
```

### user_management_default_secondary_groups_append

If `true`, add the user to the groups specified in `groups`. If `false`, user will only be added to the groups specified in `groups`, removing them from all other groups.

**_Required:_** false<br />
**_Type:_** bool<br />

#### Default value

```YAML
user_management_default_secondary_groups_append: false
```

### user_management_default_shell

Default user's shell. Omitted if `null`.

**_Required:_** false<br />
**_Type:_** str<br />

#### Default value

```YAML
user_management_default_shell:
```

### user_management_default_ssh_from

Default, global `from=""` value added to `authorized_keys` for each user having `user_management_users.ssh_public_keys` defined

**_Required:_** false<br />
**_Type:_** list<br />

#### Default value

```YAML
user_management_default_ssh_from: []
```

### user_management_group_ssh_from

`group_vars` specific `from=""` value added to `authorized_keys` for each user having `user_management_users.ssh_public_keys` defined

**_Required:_** false<br />
**_Type:_** list<br />

#### Default value

```YAML
user_management_group_ssh_from: []
```

### user_management_group_users

Users added for one inventory group, typically set in `group_vars/<group>.yml`. Merged on top of `user_management_users` by user name, so a group can add accounts - or redefine a global one - without copying the whole list. Takes the same keys.

**_Required:_** false<br />
**_Type:_** list<br />

#### Default value

```YAML
user_management_group_users: []
```

#### Example usage

```YAML
# Same keys as user_management_users.
user_management_group_users:
  - name: deploy
    state: present
```

### user_management_host_ssh_from

`host_vars` specific `from=""` value added to `authorized_keys` for each user having `user_management_users.ssh_public_keys` defined

**_Required:_** false<br />
**_Type:_** list<br />

#### Default value

```YAML
user_management_host_ssh_from: []
```

### user_management_host_users

Users added for a single host, typically set in `host_vars/<host>.yml`. Merged last and therefore wins over the group and global lists for the same user name. Takes the same keys.

**_Required:_** false<br />
**_Type:_** list<br />

#### Default value

```YAML
user_management_host_users: []
```

#### Example usage

```YAML
# Same keys as user_management_users.
user_management_host_users:
  - name: backup-agent
    state: present
```

### user_management_users

Global baseline list of users to be managed, typically set in `group_vars/all.yml`. Merged by user name with `user_management_group_users` and `user_management_host_users`. See the per-user key table in the README for the keys each entry accepts.

**_Required:_** false<br />
**_Type:_** list<br />

#### Default value

```YAML
user_management_users:
  - name: user_management_john.doe
    absolute_home_path:
    custom_ssh_from: []
    gecos: John Doe,Room 123,212-555-0000,212-555-3456,john.doe@world.org
    groups_append: false
    home_create: false
    home_move: false
    password: $6$hUyQQf65j3czoiSV$gckRY17sJxEKgiHPqXszseZs.6x5Ehu995GixUnPnSB018J4ijM7Xw.3/B6xmVTlqUJzPQhITMH3hpHXwD14r.
    primary_group:
    secondary_groups: []
    ssh_public_keys:
      - ssh-rsa AAAAB3NzakGUg/a4nViDJGhuFty+4yR8fWQ== ansible-role-user-management-1
    state: absent
    userdel_force: false
    userdel_remove: false
```

#### Example usage

```YAML
# Every key an entry accepts. user_management_group_users and
# user_management_host_users take exactly the same keys.
user_management_users:
  # User's Linux login name.
  - name: john.doe  # str, required
    # Whether the account should exist or not, taking action if the
    # state is different from what is stated.
    state: present  # str, required, choices: absent|present
    # Optionally sets the description (aka GECOS) of user account: Full
    # Name, Room Number, Work Phone, Home Phone, Other
    gecos: "John Doe,Room 123,212-555-0000,,john.doe@world.org"  # str, optional
    # If provided, set the user's password to the provided encrypted
    # hash password. To create an account with a locked/disabled
    # password, set this to `!` or `*`. How to generate encrypted
    # passwords: Ansible Documentation -
    # https://docs.ansible.com/ansible/latest/reference_appendices/faq.html#how-do-i-generate-encrypted-passwords-for-the-user-module
    password: "$6$rounds=656000$salt$hash"  # str, optional
    # Optionally sets the user's primary group (takes a group name).
    primary_group: developers  # str, optional
    # List of groups user will be added to. By default, the user is
    # removed from all other groups. Configure `groups_append` to modify
    # this. When set to an empty string `''`, the user is removed from
    # all groups except the primary group.
    secondary_groups: ["docker", "sudo"]  # list, optional
    # If `true`, add the user to the groups specified in groups. If
    # `false`, user will only be added to the groups specified in
    # `secondary_groups`, removing them from all other groups.
    groups_append: true  # bool, optional, choices: False|True
    # Overwrites 'user_management_default_shell'.
    shell: /bin/bash  # str, optional
    # Custom `$HOME` root path. Must be specified as absolute path.
    absolute_home_path: /opt/john.doe  # str, optional
    # Unless set to false, a home directory will be created for the user
    # when the account is created or if the home directory does not
    # exist.
    home_create: true  # bool, optional, choices: False|True
    # If set to `true` when used with `home:`, attempt to move the
    # user's old home directory to the specified directory if it isn't
    # already there and the old home exists.
    home_move: false  # bool, optional, choices: False|True
    # The SSH public key(s), as a list or (since Ansible 1.9) url.
    ssh_public_keys: ["ssh-ed25519 AAAAC3Nz... john.doe"]  # list, optional
    # `from=""` value added to `authorized_keys` if user has
    # `user_management_users.ssh_public_keys` defined. If
    # `user_management_default_ssh_from` or `custom_ssh_from` is defined
    # and not set to `'*'`, all values will be concatenated.
    custom_ssh_from: ["10.1.0.0/16"]  # list, optional
    # This only affects `state=absent`. It forces removal of the user
    # and associated directories on supported platforms.
    userdel_force: false  # bool, optional, choices: False|True
    # This only affects `state=absent`. It attempts to remove
    # directories associated with the user.
    userdel_remove: false  # bool, optional, choices: False|True
```

## Dependencies

None.

## License

MIT

## Author

Xenion1987
