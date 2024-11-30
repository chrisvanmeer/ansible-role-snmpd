ansible-role-snmpd
==================

Ansible role to install and manage snmpd plus with snmpv3 users.

Requirements
------------

No requirements.

Role Defaults
-------------

See `defaults/main.yml` for all defaults.

Dependencies
------------

No dependencies

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

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

License
-------

BSD

Author Information
------------------

Chris van Meer <chris@atcomputing.nl>
