# Ansible

**Topic:** [[devops/topics/infrastructure-as-code]]
**Related:** [[devops/concepts/infrastructure-as-code]], [[devops/concepts/terraform-patterns]], [[devops/concepts/secrets-management]]

---

## What it is

Ansible is an agentless, push-based configuration management and automation tool. It connects to target hosts via SSH (or WinRM for Windows) and runs tasks defined in YAML **playbooks**. Unlike Terraform, Ansible is stateless — it does not track what it has done. Each run re-evaluates and re-applies.

**When to use Ansible over Terraform:**
- Configuring software on existing machines (install packages, write config files, manage services)
- Ad-hoc fleet operations (patch all VMs, restart a service across 500 nodes)
- Bootstrapping a VM image (before Packer or after Terraform provisions)
- Environments where immutable infrastructure is not practical (bare metal, legacy VMs)

**When Terraform is the right tool instead:**
- Creating cloud resources (VPCs, EC2, RDS, IAM) — Terraform tracks state and handles lifecycle
- Anything that needs destroy/re-create semantics or drift detection

---

## Core Concepts

### Inventory

The list of hosts Ansible will manage. Can be static (INI/YAML file) or dynamic (pull from AWS, GCP, Azure, or any API).

**Static inventory:**
```ini
[webservers]
web-01.internal ansible_host=10.0.1.10
web-02.internal ansible_host=10.0.1.11

[databases]
db-01.internal  ansible_host=10.0.2.10 ansible_user=ubuntu

[production:children]
webservers
databases

[all:vars]
ansible_ssh_private_key_file=~/.ssh/deploy_key
```

**Dynamic inventory (AWS EC2 plugin):**
```yaml
# aws_ec2.yaml
plugin: amazon.aws.aws_ec2
regions: [us-east-1]
filters:
  tag:Environment: prod
  instance-state-name: running
keyed_groups:
  - key: tags.Role
    prefix: role
```
```bash
ansible-inventory -i aws_ec2.yaml --list   # verify dynamic inventory
ansible -i aws_ec2.yaml role_web -m ping   # ping all EC2s tagged Role=web
```

### Playbook

An ordered list of **plays**. Each play maps a group of hosts to a list of **tasks**. Each task calls an Ansible **module**.

```yaml
# site.yml
- name: Configure web servers
  hosts: webservers
  become: true          # sudo / privilege escalation
  vars:
    app_port: 8080
    deploy_user: appuser

  tasks:
    - name: Install nginx
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Write nginx config
      ansible.builtin.template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
      notify: Restart nginx       # triggers handler below

    - name: Ensure nginx is running and enabled
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

### Idempotency

Every Ansible module is designed to be idempotent:
- `package: state: present` — installs if absent, does nothing if already installed
- `file: state: directory` — creates directory if absent, does nothing if exists
- `template` — writes file if content changed, skips if content matches

Running the same playbook 10 times against the same host should produce 0 changes after the first run. The module's `changed` vs. `ok` return indicates whether actual changes were made.

**Where idempotency breaks:** `command` and `shell` modules are NOT idempotent by default. They always run and always report `changed`. Fix: use `creates` or `removes` arguments, or `when` conditions to guard them.

```yaml
# NOT idempotent — runs every time
- name: Initialize database
  ansible.builtin.command: /usr/local/bin/init-db.sh

# Idempotent — skip if already done
- name: Initialize database
  ansible.builtin.command: /usr/local/bin/init-db.sh
  args:
    creates: /var/lib/myapp/.db_initialized   # skip if this file exists
```

---

## Roles

Roles provide a standardized directory structure for grouping related tasks, templates, files, variables, and handlers.

```
roles/
└── webserver/
    ├── tasks/
    │   └── main.yml        # main task list
    ├── handlers/
    │   └── main.yml        # handlers triggered by notify
    ├── templates/
    │   └── nginx.conf.j2   # Jinja2 templates
    ├── files/
    │   └── index.html      # static files to copy
    ├── vars/
    │   └── main.yml        # role-internal variables (high precedence)
    ├── defaults/
    │   └── main.yml        # default variable values (lowest precedence)
    └── meta/
        └── main.yml        # role metadata, dependencies
```

Using a role in a playbook:

```yaml
- hosts: webservers
  roles:
    - role: webserver
      vars:
        app_port: 8080
    - role: monitoring_agent
```

**Galaxy:** Ansible's public role marketplace. `ansible-galaxy install geerlingguy.nginx` installs a community role.

---

## Variables and Precedence

Ansible variable precedence is complex (22 levels). The practical hierarchy:

| Priority | Source | Example |
|---|---|---|
| Highest | `--extra-vars` CLI flag | `-e "env=prod"` |
| High | `vars:` in the play | Play-level vars block |
| Medium | `group_vars/`, `host_vars/` | Environment-specific vars |
| Low | Role `vars/main.yml` | Role-internal vars |
| Lowest | Role `defaults/main.yml` | Default values callers can override |

`group_vars/` and `host_vars/` are directories at the same level as the inventory file:

```
inventory/
├── hosts.ini
├── group_vars/
│   ├── all.yml          # applies to all hosts
│   ├── webservers.yml   # applies to webservers group
│   └── prod.yml         # applies to prod group
└── host_vars/
    └── web-01.internal.yml  # applies to this specific host
```

---

## Ansible Vault (Secrets Management)

Ansible Vault encrypts sensitive data (passwords, API keys) so they can be stored in version control safely:

```bash
# Encrypt a variables file
ansible-vault encrypt group_vars/prod/secrets.yml

# Create an encrypted file from scratch
ansible-vault create group_vars/prod/secrets.yml

# Edit in-place (decrypts → editor → re-encrypts)
ansible-vault edit group_vars/prod/secrets.yml

# Run playbook with vault password
ansible-playbook site.yml --vault-password-file ~/.vault_pass
ansible-playbook site.yml --ask-vault-pass

# Encrypt a single value (embed in a YAML file)
ansible-vault encrypt_string 'super_secret_password' --name 'db_password'
```

Encrypted file looks like:
```
$ANSIBLE_VAULT;1.1;AES256
66386439653236336462626566653062663...
```

**In CI:** store the vault password as a CI secret; write to a temp file before running Ansible; delete after.

---

## Jinja2 Templates

Ansible templates use Jinja2 for variable substitution and logic:

```jinja
# templates/app.conf.j2
[server]
host = {{ ansible_hostname }}
port = {{ app_port | default(8080) }}
env  = {{ environment }}

{% if environment == "prod" %}
log_level = WARNING
workers   = {{ ansible_processor_vcpus * 2 }}
{% else %}
log_level = DEBUG
workers   = 2
{% endif %}

[database]
{% for db in databases %}
replica_{{ loop.index }} = {{ db.host }}:{{ db.port }}
{% endfor %}
```

Common Jinja2 filters: `| default(value)`, `| upper`, `| lower`, `| join(',')`, `| selectattr('active', 'eq', true)`.

---

## Key Modules Reference

| Module | Purpose | Idempotent |
|---|---|---|
| `package` / `apt` / `yum` | Install/remove packages | Yes |
| `template` | Write Jinja2 template to file | Yes (content-based) |
| `copy` | Copy static file to host | Yes (content-based) |
| `file` | Create directories, set permissions, symlinks | Yes |
| `service` | Start/stop/enable system services | Yes |
| `user` | Create/modify system users | Yes |
| `lineinfile` | Ensure a specific line exists in a file | Yes |
| `blockinfile` | Ensure a block of text exists in a file | Yes |
| `cron` | Manage cron jobs | Yes |
| `git` | Clone/update a git repo | Yes (if rev matches) |
| `command` | Run shell command (no shell features) | No — use `creates`/`removes` |
| `shell` | Run shell command (with pipes, redirects) | No — use guards |
| `uri` | Make HTTP requests (wait for service health) | Depends on usage |
| `wait_for` | Wait for port/file/condition | Yes |
| `debug` | Print variable values during run | Yes (no-op) |

---

## Ansible in CI/CD Pipeline

```yaml
# .gitlab-ci.yml excerpt
deploy:
  stage: deploy
  image: cytopia/ansible:latest
  script:
    - echo "$VAULT_PASSWORD" > /tmp/.vault_pass
    - ansible-playbook
        -i inventories/prod/
        --vault-password-file /tmp/.vault_pass
        -e "app_version=${CI_COMMIT_SHORT_SHA}"
        site.yml
  after_script:
    - rm -f /tmp/.vault_pass
  only:
    - main
```

---

## Terraform + Ansible: The Standard Pairing

```
Step 1: Terraform provisions cloud infrastructure
  → EC2 instances, VPC, RDS, IAM roles, security groups
  → Output: instance IPs written to a file or SSM Parameter Store

Step 2: Ansible configures the provisioned machines
  → Uses dynamic inventory to discover Terraform-managed EC2s (by tag)
  → Installs application runtime, writes config files, starts services

Modern alternative (immutable):
  → Packer builds AMI with all software pre-installed (runs Ansible at image build time)
  → Terraform launches EC2 with that AMI (no Ansible at deploy time)
  → Faster deployments, no SSH needed in production
```

---

## Common Interview Q&A

**Q: How is Ansible different from Terraform?**
A: Ansible is agentless, stateless, and procedural — it executes tasks in order without remembering what it did before. Best for OS-level configuration (packages, files, services) on existing machines. Terraform is declarative and stateful — it tracks resource state and manages full lifecycle (create, update, destroy) for cloud infrastructure. They are complementary: Terraform provisions, Ansible configures. In modern immutable infrastructure, Packer + Ansible bake an AMI; Terraform deploys it.

**Q: How do you make a `command` module task idempotent?**
A: Use the `creates` argument to specify a file whose existence indicates the command already ran; Ansible skips the task if the file exists. Alternatively, use the `when` conditional with a `stat` module check: `when: not result.stat.exists`. For more complex idempotency, wrap the command in a script that exits 0 without side effects if already done.

**Q: How do you handle secrets in Ansible without committing them to Git in plaintext?**
A: Ansible Vault encrypts secrets at rest. `ansible-vault encrypt` produces ciphertext that is safe to commit. The vault password is provided at runtime via `--vault-password-file` or `--ask-vault-pass`. For CI/CD, the vault password is stored as a CI secret (environment variable). Alternative: use `lookup('hashi_vault', ...)` to fetch secrets from HashiCorp Vault at playbook runtime, keeping no secret data in the repo at all.

**Q: 500 servers need an emergency security patch. How do you use Ansible to do this safely?**
A: Write a playbook using the `package` module with `state: latest` scoped to the specific CVE package. Test on 1 host first: `--limit web-01.internal`. Run with `--forks 5` (5 parallel hosts) rather than default 5 to control blast radius. Use `serial: 10%` at the play level to process 10% of inventory at a time with a pass/fail gate between batches. Monitor the run with `--diff` to see exactly what changes. After the patch, a second task verifies the service is still running.

---

## Sources
- [[devops/concepts/infrastructure-as-code]]
- [[devops/topics/infrastructure-as-code]]
- [[devops/concepts/secrets-management]]
