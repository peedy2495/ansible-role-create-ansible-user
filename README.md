# Create Ansible user

An Ansible Galaxy role for Debian, Ubuntu, and Enterprise Linux distributions
such as RHEL, Rocky Linux, AlmaLinux, Oracle Linux, and CentOS. It creates a
key-only automation user with a home directory, login shell, locked password,
and unrestricted passwordless sudo access. By default, the role then proves
that the account works by opening a fresh SSH connection from the controller
and running `sudo -n true` as the new user.

## Requirements

- Ansible Core 2.14 or newer
- Debian, Ubuntu, or an Enterprise Linux-compatible managed host
- Gathered facts and privilege escalation
- The OpenSSH client on the Ansible controller when login testing is enabled

## Installation

```bash
ansible-galaxy role install peedy2495.create_ansible_user
```

## Variables

At least one public key is required:

```yaml
create_ansible_user_ssh_public_keys:
  - ssh-ed25519 AAAA... controller@example
```

Defaults:

```yaml
create_ansible_user_name: ansible
create_ansible_user_comment: Ansible automation account
create_ansible_user_shell: /bin/bash
create_ansible_user_home: "/home/{{ create_ansible_user_name }}"
create_ansible_user_state_file_enabled: false
create_ansible_user_login_test_enabled: true
create_ansible_user_login_test_host: "{{ ansible_host | default(inventory_hostname) }}"
create_ansible_user_login_test_port: "{{ ansible_port | default(22) }}"
create_ansible_user_login_test_identity_file: ""
create_ansible_user_login_test_connect_timeout: 10
create_ansible_user_login_test_retries: 3
```

The role fully manages `authorized_keys`; removing a key from the variable also
removes it from the host.

## Login verification

The login test runs on the Ansible controller after `authorized_keys` and sudo
have been configured. It disables password and interactive authentication,
opens a new SSH session as `create_ansible_user_name`, and executes
`sudo -n true`. Thus, a successful test verifies both public-key login and
passwordless privilege escalation.

By default, OpenSSH tries keys from its usual identity files and the SSH agent.
If the matching key is elsewhere, set its controller-side path explicitly:

```yaml
create_ansible_user_login_test_identity_file: /secure/path/ansible_ed25519
```

The host and port default to the current inventory connection's `ansible_host`
and `ansible_port`. Override them when the controller must use another route.
The test is not run in Ansible check mode. To opt out completely:

```yaml
create_ansible_user_login_test_enabled: false
```

Disabling the test means that the role verifies files and sudoers syntax only;
it no longer proves that the account can authenticate.

## State marker

When `create_ansible_user_state_file_enabled` is enabled, `.ssh/.state` contains:

- `nok` while configuration or verification is incomplete.
- `ok` after `authorized_keys` was read back, sudo was configured, and the
  enabled login test succeeded.

Following security roles should use the same sequence: write `nok`, apply their
changes, verify the effective configuration, then write `ok`. The marker is a
workflow status, not proof of security compliance.

## Credentials per target group

Use separate inventory groups and `group_vars` files:

```yaml
# inventory/group_vars/customer_a.yml
create_ansible_user_ssh_public_keys:
  - ssh-ed25519 AAAA... customer-a-controller
```

```yaml
# inventory/group_vars/customer_b.yml
create_ansible_user_ssh_public_keys:
  - ssh-ed25519 AAAA... customer-b-controller
```

Avoid assigning a host to multiple groups that define this variable. If groups
overlap, set the final value in `host_vars`. Use Ansible Vault where required.

## Usage

```yaml
- name: Create the Ansible automation account
  hosts: servers
  gather_facts: true
  become: true
  vars:
    create_ansible_user_ssh_public_keys: "{{ management_ansible_public_keys }}"
    create_ansible_user_login_test_identity_file: /secure/path/ansible_ed25519
  roles:
    - role: peedy2495.create_ansible_user
```

## License

MIT
