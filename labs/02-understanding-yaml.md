# Lab 02 - Understanding YAML

## Objectives

- Read and write valid YAML.
- Understand scalars, lists, and dictionaries.
- Recognize how indentation conveys structure.
- Validate YAML and avoid the most common beginner mistakes.

## Background

Ansible playbooks, inventories, and variable files are written in **YAML** ("YAML Ain't Markup
Language"). YAML is whitespace-sensitive and designed to be human-readable.

Core rules:

- **Indentation uses spaces, never tabs.** Two spaces per level is the convention.
- A YAML file usually starts with `---`.
- Comments start with `#`.

## Steps

### 1. Scalars (single values)

Create `yaml-demo.yml`:

```yaml
---
name: web01
port: 8080
enabled: true
ratio: 0.75
empty_value: null
```

- Strings usually need no quotes. Quote them when they contain special characters (`:`, `#`, `{`) or to force a type, e.g. `version: "3.10"` (without quotes `3.10` could be read as a float).
- Booleans: `true`/`false` (also `yes`/`no`, but prefer `true`/`false`).

### 2. Lists (sequences)

```yaml
packages:
  - nginx
  - git
  - htop

# Inline (flow) form:
ports: [80, 443, 8080]
```

### 3. Dictionaries (mappings)

```yaml
user:
  name: alice
  uid: 1001
  shell: /bin/bash

# Inline form:
address: { city: Rome, zip: "00100" }
```

### 4. Nesting lists and dictionaries

This is the structure Ansible uses constantly - a list of dictionaries:

```yaml
users:
  - name: alice
    groups:
      - sudo
      - docker
  - name: bob
    groups:
      - users
```

### 5. Multi-line strings

```yaml
# Block scalar - keeps newlines (|)
motd: |
  Welcome to the server.
  Authorized users only.

# Folded scalar - folds newlines into spaces (>)
description: >
  This is one long line
  written across several
  source lines.
```

### 6. Validate your YAML

```bash
python3 -c "import yaml,sys; print(yaml.safe_load(open('yaml-demo.yml')))"
```

If it prints a Python dict, your YAML is valid. If you get a `yaml.scanner.ScannerError`, fix the indentation.

Install the linter for better feedback:

```bash
python3 -m pip install --user yamllint
yamllint yaml-demo.yml
```

## Common mistakes

| Mistake                        | Fix                                                |
| ------------------------------ | -------------------------------------------------- |
| Using tabs                     | Use spaces only                                    |
| Inconsistent indentation       | Pick 2 spaces and stay consistent                  |
| Unquoted value with `:`        | Quote it: `msg: "time: now"`                       |
| Missing space after `-` or `:` | `- item` and `key: value`, not `-item`/`key:value` |
| `version: 3.10` becoming `3.1` | Quote version numbers                              |

## Verification

- `yamllint yaml-demo.yml` reports no errors.
- The Python validation command prints a dictionary.

## Challenge

Write a YAML file describing **two** servers. Each server has a `name`, a `role`
(string), a list of `packages`, and a nested `network` dictionary with `ip` and `gateway`.
Validate it with `yamllint`.

## Key takeaways

- Indentation (spaces) defines structure.
- Three building blocks: scalars, lists, dictionaries - combined arbitrarily.
- Validate early; most Ansible "syntax" errors are really YAML errors.

---

**Next lab:** [Lab 03 - Ansible Inventory](03-ansible-inventory.md)
