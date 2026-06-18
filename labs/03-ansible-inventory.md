# Lab 03 - Ansible Inventory

## Objectives

- Understand what an inventory is and why Ansible needs one.
- Define hosts and connection parameters.
- Use `ansible-inventory` to inspect what Ansible parses.
- Configure a default inventory via `ansible.cfg`.

## Background

The **inventory** is the list of hosts (and groups of hosts) that Ansible manages. It can be a
static file or generated dynamically. Each host can carry connection variables such as the SSH
user, port, or Python interpreter.

## Steps

### 1. A basic INI inventory

Create `inventory.ini`:

```ini
# A single host reachable over SSH
web01 ansible_host=192.168.56.11 ansible_user=ubuntu

# The control node itself
localhost ansible_connection=local
```

Common host variables:

| Variable                     | Meaning                    |
| ---------------------------- | -------------------------- |
| `ansible_host`               | IP/DNS to connect to       |
| `ansible_user`               | SSH login user             |
| `ansible_port`               | SSH port (default 22)      |
| `ansible_connection`         | `ssh` (default) or `local` |
| `ansible_python_interpreter` | Path to Python on the host |

### 2. Group hosts

```ini
[webservers]
web01 ansible_host=192.168.56.11
web02 ansible_host=192.168.56.12

[dbservers]
db01 ansible_host=192.168.56.21
```

### 3. Inspect the parsed inventory

```bash
# List everything Ansible sees
ansible-inventory -i inventory.ini --list

# Show it as a tree
ansible-inventory -i inventory.ini --graph
```

`--graph` output looks like:

```
@all:
  |--@webservers:
  |  |--web01
  |  |--web02
  |--@dbservers:
  |  |--db01
  |--@ungrouped:
  |  |--localhost
```

### 4. Target hosts and groups

```bash
ansible all          -i inventory.ini -m ping
ansible webservers   -i inventory.ini -m ping
ansible web01        -i inventory.ini -m ping
ansible 'web*'       -i inventory.ini --list-hosts   # pattern match
```

Note the built-in groups: **`all`** (every host) and **`ungrouped`** (hosts in no group).

### 5. Set a default inventory with `ansible.cfg`

Tired of typing `-i inventory.ini`? Create `ansible.cfg` in the project directory:

```ini
[defaults]
inventory = ./inventory.ini
host_key_checking = False
```

Now you can omit `-i`:

```bash
ansible all -m ping
ansible-inventory --graph
```

> `host_key_checking = False` is convenient in labs but should be used carefully in production.

## Verification

- `ansible-inventory --graph` shows your groups and hosts.
- `ansible all -m ping` (no `-i`) works after creating `ansible.cfg`.

## Challenge

1. Add a third group `loadbalancers` with one host.
2. Use `ansible <group> --list-hosts` to confirm membership of each group.
3. Use a wildcard pattern to list only hosts whose name starts with `db`.

## Key takeaways

- The inventory defines _what_ Ansible manages; modules define _what to do_.
- `ansible-inventory --graph/--list` is the way to debug inventory problems.
- `ansible.cfg` removes repetitive `-i` flags and sets project defaults.

---

**Next lab:** [Lab 04 - Inventory Formats](04-inventory-formats.md)
