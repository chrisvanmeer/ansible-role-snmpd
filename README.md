ansible-role-snmpd
==================

Ansible role to install and manage snmpd plus with snmpv3 users.

This role was partially forked / used from the [robertdebock.snmpd](https://github.com/robertdebock/ansible-role-snmpd)
role. However, I found that certain parts are lacking and deviates far enough from the original to grant a forked version.

Requirements
------------

No requirements.

Role Defaults
-------------

See `defaults/main.yml` for all defaults that are available in this role.

I've commented out almost every one to make the role more flexible. Needless to say, you would have
to fit thise into either `group_vars` and/or `host_vars` to make this work.

Dependencies
------------

No dependencies

Example Playbook
----------------

    - name: Install and manage snmpd
      hosts: all
      become: true

      roles:
        - role: chrisvanmeer.snmpd
          snmpd_v3_users_global:
            - username: snmp
              auth_proto: SHA
              auth_pass: <ansible-vault encrypted string>
              priv_proto: AES
              priv_pass: <ansible-vault encrypted string>
              readonly: true
              authpriv: true

License
-------

BSD

Author Information
------------------

Chris van Meer <chris@atcomputing.nl>
