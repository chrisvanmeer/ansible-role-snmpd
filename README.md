# ansible-role-snmpd

[![CI](https://github.com/chrisvanmeer/ansible-role-snmpd/actions/workflows/ci.yml/badge.svg)](https://github.com/chrisvanmeer/ansible-role-snmpd/actions/workflows/ci.yml)
[![Ansible Galaxy](https://img.shields.io/galaxy/role/chrisvanmeer/snmpd)](https://galaxy.ansible.com/chrisvanmeer/snmpd/)

Ansible role to install and manage `snmpd`, including SNMPv3 users.

This role was partially forked / used from the [robertdebock.snmpd](https://github.com/robertdebock/ansible-role-snmpd)
role. However, certain parts were lacking and it deviates far enough from the original to grant a forked version.

## Requirements

No hard requirements. Only the standard Ansible Core modules (`ansible.builtin`) are used, so no
collections need to be installed alongside the role. The declared minimum is
`min_ansible_version` from [`meta/main.yml`](meta/main.yml), which CI verifies on every run.

## Role Variables

The role is config driven. Every directive is left unset by default, so only what you define ends up
in the rendered `snmpd.conf`. The following variables are supported:

### Directives

| Variable | Type | Description |
|---|---|---|
| `snmpd_syslocation` | string | `syslocation` directive. |
| `snmpd_syscontact` | string | `syscontact` directive. |
| `snmpd_sysservices` | number | `sysServices` directive. |
| `snmpd_agent_address` | string | `agentaddress` directive, e.g. `udp:161`. |
| `snmpd_master_agentx` | boolean | Enable `master agentx`. |
| `snmpd_views` | list | `view` directives (`name`, `type`, `subtree`, optional `mask`). |
| `snmpd_security_names` | list | `com2sec` directives (`name`, `source`, `community`). |
| `snmpd_groups` | list | `group` directives (`name`, `security_model`, `security_name`). |
| `snmpd_accesses` | list | `access` directives (`group`, `context`, `security_model`, `security_level`, `prefix`, `read`, `write`, `notif`). |
| `snmpd_dontlogtcpwrappersconnects` | string | `dontLogTCPWrappersConnects` directive (`yes`/`no`). |
| `snmpd_processes` | list | `proc` directives (`name`, optional `maximum`, `minimum`). |
| `snmpd_scripts` | list | `exec` directives (`name`, `program`, `arguments`). |
| `snmpd_disks` | list | `disk` directives (`path`, optional `minimum`). |
| `snmpd_load` | dict | `load` directive (`one_minute_average`, `five_minute_average`, `fifteen_minute_average`). |

Note that the net-snmp `proc` directive takes its limits in the order `proc NAME [MAX [MIN]]`, and
`disk` takes `disk PATH [MIN]`. The role renders `maximum` before `minimum` for `proc` so that the
resulting line matches that order.

### SNMPv3 users

| Variable | Type | Description |
|---|---|---|
| `snmpd_v3_users_global` | list | SNMPv3 users applied to a whole inventory. |
| `snmpd_v3_users_group` | list | SNMPv3 users applied at group level. |
| `snmpd_v3_users_host` | list | SNMPv3 users applied per host. |

All three `snmpd_v3_users_*` lists are combined into `snmpd_v3_users`. Each user entry supports:

```yaml
- username: snmp
  auth_proto: SHA        # MD5, SHA, SHA-224, SHA-256, SHA-384, SHA-512
  auth_pass: secret
  priv_proto: AES        # DES, AES, AES-128, AES-192, AES-256
  priv_pass: secret
  readonly: true         # optional, default false (rwuser)
  authpriv: true         # optional, restrict to authPriv security level
```

User names may only contain letters, digits, dots, underscores and dashes, and each name may be
defined only once across the three lists. Both are validated by the role before any change is made.

Store the passwords in `group_vars`/`host_vars` (ideally encrypted with `ansible-vault`).

### Operational defaults

| Variable | Default | Description |
|---|---|---|
| `snmpd_packages` | derived from `ansible_os_family` | Package(s) that provide the daemon. |
| `snmpd_service` | `snmpd` | Service name used for start, stop, restart and boot time enabling. |
| `snmpd_config_file` | `/etc/snmp/snmpd.conf` | Rendered main configuration file. |
| `snmpd_v3_users_file` | `/var/lib/snmp/snmpd.conf` on Debian, `/var/lib/net-snmp/snmpd.conf` elsewhere | net-snmp persistence file holding the `createUser` lines. |

Override these in your play, `group_vars` or `host_vars` when a platform does not follow these
conventions.

## Example Playbook

```yaml
- name: Install and manage snmpd
  hosts: all
  become: true

  roles:
    - role: chrisvanmeer.snmpd
      snmpd_syslocation: Amsterdam
      snmpd_syscontact: ops@example.com
      snmpd_v3_users_global:
        - username: snmp
          auth_proto: SHA
          auth_pass: <ansible-vault encrypted string>
          priv_proto: AES
          priv_pass: <ansible-vault encrypted string>
          readonly: true
          authpriv: true
```

## Notes on SNMPv3

- SNMPv3 `createUser` definitions are written to the net-snmp persistence file
  (`snmpd_v3_users_file`) while `snmpd` is stopped, after which the service is started again.
  `rouser`/`rwuser` lines are rendered into `snmpd_config_file`.
- Both the configuration file and the persistence file are written with mode `0600` because they
  contain community strings and user credentials.
- The variable validation runs on the Ansible controller with privilege escalation disabled. It is
  applied through a `block` in `tasks/assert.yml` on purpose: a `delegate_to` on an `import_tasks`
  is silently ignored, so it would not have any effect there.
- The role does not send any e-mail. If you add notification logic in the future, do so from the
  Ansible controller (`run_once`, `delegate_to: localhost`, `become: false`) so the control node's
  mail relay is used instead of the (usually unreachable) relay of the managed host.

### Changing the credentials of an existing user

On its first start `snmpd` replaces the plaintext `createUser` lines in the persistence file with
`usmUser` lines that hold hashed keys. The role treats a user name that is present in either form as
already provisioned, which is what keeps the role idempotent and stops it from piling up duplicate
entries.

The consequence is that **changing `auth_pass` or `priv_pass` for a user that already exists does not
rotate that user's credentials**. To rotate, remove the user from the persistence file and run the
role again; `snmpd` then re-hashes the new credentials:

```bash
# on the managed host, as root
sed -i '/"snmp"/d' /var/lib/snmp/snmpd.conf
```

## Testing

The role ships [Molecule](https://ansible.readthedocs.io/projects/molecule/) scenarios for Debian 12
and Ubuntu 22.04. Each scenario runs a full `molecule test` sequence: syntax check, converge,
idempotence check and verification. Verification asserts the rendered configuration directive by
directive, the presence of each SNMPv3 user, the file modes, and finally queries the running daemon
over both SNMPv2c and SNMPv3 (`authPriv`).

The playbooks are shared between the scenarios from `molecule/shared/`, so both platforms are
exercised with exactly the same configuration and the same assertions.

```bash
molecule test --all                  # every scenario
molecule test --scenario-name debian # a single scenario
molecule converge -s ubuntu          # just apply
```

The scenarios need a Docker daemon and never reach out to Ansible Galaxy, because the role is
installed from the local checkout.

## Continuous integration

[`.github/workflows/ci.yml`](.github/workflows/ci.yml) runs on every push and pull request and has
three stages:

1. **Lint** — `yamllint` and `ansible-lint` at the `production` profile.
2. **Compatibility** — parses the role with the `min_ansible_version` declared in `meta/main.yml`.
3. **Test** — the Molecule suite on Debian and Ubuntu.

When a `vMAJOR.MINOR.PATCH` tag is pushed and all three stages pass, the role is imported into
Ansible Galaxy as `chrisvanmeer.snmpd` using the `GALAXY_API_KEY` secret. Push a tag to publish:

```bash
git tag -a v1.1.0 -m "Release v1.1.0"
git push origin v1.1.0
```

## Installation

```bash
ansible-galaxy role install chrisvanmeer.snmpd
```

## License

BSD

## Author Information

Chris van Meer <chris@atcomputing.nl>
