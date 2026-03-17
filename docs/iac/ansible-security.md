# Ansible Security

## Ansible Vault — Encrypting Secrets

### Basic Usage

```bash
# Create encrypted file
ansible-vault create secrets.yml

# Encrypt existing file
ansible-vault encrypt vars/production.yml

# Edit encrypted file
ansible-vault edit secrets.yml

# Decrypt file
ansible-vault decrypt secrets.yml

# Run playbook with vault password
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

### Encrypting Individual Variables

```yaml
# vars/main.yml
db_host: db.example.com
db_user: appuser
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  6462313236...encrypted-data...
```

```bash
# Encrypt a single string
ansible-vault encrypt_string 's3cure_p@ss' --name 'db_password'
```

## Secure Playbook Practices

### Least Privilege

```yaml
# Run tasks with minimum required privilege
- name: Configure application
  hosts: app_servers
  become: false  # Don't escalate by default

  tasks:
    - name: Deploy application files
      ansible.builtin.copy:
        src: app/
        dest: /opt/app/
        owner: appuser
        group: appgroup
        mode: '0750'

    # Only escalate when necessary
    - name: Restart service
      ansible.builtin.systemd:
        name: myapp
        state: restarted
      become: true
      become_user: root
```

### Secure File Permissions

```yaml
- name: Set secure file permissions
  ansible.builtin.file:
    path: /etc/myapp/config.yml
    owner: appuser
    group: appgroup
    mode: '0640'  # Owner read/write, group read, others none

- name: Set secure directory permissions
  ansible.builtin.file:
    path: /etc/myapp/
    state: directory
    owner: appuser
    group: appgroup
    mode: '0750'
    recurse: true
```

### No Secrets in Logs

```yaml
- name: Set database password
  ansible.builtin.lineinfile:
    path: /etc/myapp/db.conf
    regexp: '^DB_PASSWORD='
    line: "DB_PASSWORD={{ db_password }}"
  no_log: true  # Prevent logging sensitive data
```

## Security Hardening Roles

### SSH Hardening

```yaml
- name: Harden SSH configuration
  ansible.builtin.template:
    src: sshd_config.j2
    dest: /etc/ssh/sshd_config
    owner: root
    group: root
    mode: '0600'
    validate: '/usr/sbin/sshd -t -f %s'
  notify: restart sshd

# sshd_config.j2
# PermitRootLogin no
# PasswordAuthentication no
# PubkeyAuthentication yes
# X11Forwarding no
# MaxAuthTries 3
# ClientAliveInterval 300
# ClientAliveCountMax 2
# AllowUsers {{ ssh_allowed_users | join(' ') }}
# Protocol 2
```

### Firewall Configuration

```yaml
- name: Configure firewall rules
  ansible.builtin.ufw:
    rule: allow
    port: "{{ item.port }}"
    proto: "{{ item.proto }}"
    src: "{{ item.src | default('any') }}"
  loop:
    - { port: '22', proto: 'tcp', src: '10.0.0.0/8' }
    - { port: '443', proto: 'tcp' }
  become: true

- name: Enable UFW with default deny
  ansible.builtin.ufw:
    state: enabled
    policy: deny
    direction: incoming
  become: true
```

## Ansible Security Scanning

### ansible-lint

```bash
# Install and run
pip install ansible-lint
ansible-lint playbook.yml

# Custom rules for security
# .ansible-lint
warn_list:
  - no-changed-when
  - command-instead-of-module

enable_list:
  - no-log-password  # Ensure no_log on password tasks
```

## Ansible Security Checklist

- [ ] All secrets encrypted with Ansible Vault
- [ ] `no_log: true` on tasks with sensitive data
- [ ] Minimum privilege (`become` only when needed)
- [ ] File permissions explicitly set (no defaults)
- [ ] SSH key-based authentication (no passwords)
- [ ] Vault password stored securely (not in repo)
- [ ] Playbooks tested with `--check` mode first
- [ ] ansible-lint running in CI pipeline
