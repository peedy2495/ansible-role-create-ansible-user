# Create Ansible user

Slim role for Ubuntu and Rocky Linux. It creates a key-only automation user
with a home directory, login shell, locked password, and unrestricted
passwordless sudo access. The role requires gathered facts and privilege
escalation; it contains no playbook.

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
```

The role fully manages `authorized_keys`; removing a key from the variable also
removes it from the host.

## State marker

When `create_ansible_user_state_file_enabled` is enabled, `.ssh/.state` contains:

- `nok` while configuration or verification is incomplete.
- `ok` after `authorized_keys` was read back and verified and sudo was
  configured successfully.

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
  ansible.builtin.include_role:
    name: ansible-role-create-ansible-user
    apply:
      become: true
  vars:
    create_ansible_user_ssh_public_keys: "{{ management_ansible_public_keys }}"
```
