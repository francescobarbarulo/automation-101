# Lab 01 - Ansible Introduction for Beginners

## Objectives

By the end of this lab you will be able to:

- Explain what Ansible is and why it is agentless.
- Install Ansible and confirm the version.
- Run your first ad-hoc command against `localhost`.
- Understand the difference between **ad-hoc commands** and **playbooks**.

## Background

Ansible is an open-source automation tool used for configuration management, application
deployment, and orchestration. Its key characteristics:

- **Agentless** - it connects over SSH (or WinRM for Windows). No software runs permanently on the managed hosts.
- **Declarative & idempotent** - you describe the desired state; running the same task twice does not change anything the second time.
- **Push-based** - the control node pushes changes to managed nodes.

## Steps

### 1. Verify the installation

```bash
ansible --version
```

You should see the Ansible core version, the config file in use, and the Python interpreter.

### 2. Create the project directory

```bash
mkdir -p ~/ansible-labs && cd ~/ansible-labs
```

### 3. Create a minimal inventory

Create a file named `inventory.ini`:

```ini
[local]
localhost ansible_connection=local
```

`ansible_connection=local` tells Ansible to run on the control node itself without SSH - perfect for learning.

### 4. Run your first ad-hoc command

The `ping` module checks connectivity (it is **not** ICMP ping - it verifies Ansible can reach the host and run Python):

```bash
ansible local -i inventory.ini -m ping
```

Expected output:

```
localhost | SUCCESS => {
    "ansible_facts": {"discovered_interpreter_python": "/usr/bin/python3"},
    "changed": false,
    "ping": "pong"
}
```

### 5. Try a few more ad-hoc modules

```bash
# Print a message
ansible local -i inventory.ini -m debug -a "msg='Hello from Ansible'"

# Run a shell command
ansible local -i inventory.ini -m command -a "uptime"

# Gather and view system facts
ansible local -i inventory.ini -m setup | head -n 30
```

### 6. Ad-hoc vs. playbook

Ad-hoc commands are great for one-off tasks. For anything repeatable you write a **playbook** (a YAML file). Here is a tiny preview - create `hello.yml`:

```yaml
---
- name: My first playbook
  hosts: local
  tasks:
    - name: Print a greeting
      ansible.builtin.debug:
        msg: "Ansible is working!"
```

Run it:

```bash
ansible-playbook -i inventory.ini hello.yml
```

## Verification

- `ansible local -i inventory.ini -m ping` returns `SUCCESS` / `pong`.
- The playbook run ends with `ok=2 changed=0 failed=0`.

## Challenge

1. Use an ad-hoc command to display the amount of free memory on `localhost` (hint: the `setup` module exposes `ansible_memfree_mb`; filter with `-a "filter=ansible_mem*"`).
2. Rewrite the `uptime` ad-hoc command as a one-task playbook.

## Key takeaways

- Ansible is agentless and idempotent.
- `-i` selects the inventory, `-m` selects the module, `-a` passes arguments.
- `ping` returning `pong` is the canonical "Ansible can reach this host" check.

---

**Next lab:** [Lab 02 - Understanding YAML](02-understanding-yaml.md)
