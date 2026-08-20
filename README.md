# Create Ansible user

An Ansible Galaxy role for Debian, Ubuntu, and Enterprise Linux distributions
such as RHEL, Rocky Linux, AlmaLinux, Oracle Linux, and CentOS. It creates a
key-only automation user with a home directory, login shell, locked password,
and unrestricted passwordless sudo access. By default, the role then proves
that the account works by opening a fresh Ansible connection as the new user
and executing a module with privilege escalation.

## Requirements

- Ansible Core 2.14 or newer
- Debian, Ubuntu, or an Enterprise Linux-compatible managed host
- Gathered facts and privilege escalation
- An SSH connection plugin usable by the Ansible controller

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
create_ansible_user_login_test_enabled: true
create_ansible_user_login_test_connect_timeout: 10
create_ansible_user_login_test_timeout: 30
```

The role adds every configured key to `authorized_keys` and preserves all other
existing entries. Removing a key from the variable does not remove it from the
host.

## Login verification

The role reuses the current inventory credential; it never reads or stores a
private key itself. After configuring the account, it resets Ansible's current
connection and reconnects to the same inventory host with
`ansible_user=create_ansible_user_name`. Because the password of the new account
is locked, a normal SSH setup must authenticate with one of the public keys in
the fully managed `authorized_keys` file.

`wait_for_connection` verifies that Ansible can connect and execute a module.
The role then runs `id -u` with `become: true`, forces the `sudo` method without
a become password, and requires UID 0. Finally, it resets the connection again
so subsequent tasks use the play's original connection settings.

The test retains the current inventory host's `ansible_host`, `ansible_port`,
proxy, SSH agent, private-key setting, and other connection options. It is not
run in Ansible check mode. To opt out completely:

```yaml
create_ansible_user_login_test_enabled: false
```

Disabling the test means that the role verifies files and sudoers syntax only;
it no longer proves that the account can authenticate.

If configuring sudo or verifying the login fails, the role removes only keys
that were newly added during that run. Keys present before the run remain
untouched, and previous file ownership and permissions are restored. An empty
file created by the failed run is removed.

Only the credential active for the current Ansible run is tested. Additional
public keys in `create_ansible_user_ssh_public_keys` are installed but cannot be
cryptographically tested without running Ansible with their corresponding
credentials.

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
  roles:
    - role: peedy2495.create_ansible_user
```

## License

MIT
