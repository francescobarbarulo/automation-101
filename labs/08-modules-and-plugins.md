# Lab 08 - Ansible Modules and Plugins

## Objectives

- Understand what modules are and how to discover them.
- Use the most common core modules.
- Read module documentation with `ansible-doc`.
- Understand the role of plugins (and how they differ from modules).

## Background

**Modules** are the units of work Ansible executes - each task calls one. Ansible ships
thousands of modules (in `ansible.builtin` and in collections). **Plugins** extend Ansible
itself (how it connects, filters data, looks up values, etc.) rather than performing a task on a
host.

## Steps

### 1. Discover and read module docs

```bash
# List available modules (a lot!)
ansible-doc -l | head

# Read full docs + examples for a module
ansible-doc ansible.builtin.copy

# Just the examples / snippet
ansible-doc -s ansible.builtin.file
```

`ansible-doc` is your offline reference - use it constantly.

### 2. Essential core modules

Create `modules-demo.yml` (targets `local`, uses `become`):

```yaml
---
- name: Core modules tour
  hosts: local
  become: true
  tasks:
    - name: file - manage files/dirs/permissions
      ansible.builtin.file:
        path: /tmp/demo
        state: directory
        mode: "0755"

    - name: copy - push content/files
      ansible.builtin.copy:
        dest: /tmp/demo/hello.txt
        content: "hello\n"

    - name: lineinfile - ensure a line is present
      ansible.builtin.lineinfile:
        path: /tmp/demo/hello.txt
        line: "added by lineinfile"

    - name: command - run a binary (no shell)
      ansible.builtin.command: ls -l /tmp/demo
      register: listing
      changed_when: false

    - name: debug - print data
      ansible.builtin.debug:
        var: listing.stdout_lines
```

Run it:

```bash
ansible-playbook -i inventory.ini modules-demo.yml
```

### 3. Package and service modules

```yaml
- name: Install a package (Debian/Ubuntu)
  ansible.builtin.apt:
    name: htop
    state: present
    update_cache: true

- name: Cross-distro package install
  ansible.builtin.package:
    name: htop
    state: present

- name: Ensure a service is running and enabled
  ansible.builtin.service:
    name: ssh
    state: started
    enabled: true
```

`ansible.builtin.package` is generic; `apt`/`yum`/`dnf` are distro-specific.

### 4. `command` vs `shell`

- `command` - runs a program directly. Safer. **No** shell features (`|`, `>`, `&&`, `$VAR`).
- `shell` - runs through `/bin/sh`, so pipes and redirects work, but be careful with untrusted input.

```yaml
- name: This needs a pipe, so use shell
  ansible.builtin.shell: "cat /etc/passwd | wc -l"
  register: users
  changed_when: false
```

> Prefer purpose-built modules over `command`/`shell` whenever one exists - they are idempotent.

### 5. Plugins (conceptual + practical)

Plugins power features you already use in expressions:

- **Filter plugins** - transform data in Jinja2: `{{ name | upper }}`, `{{ list | length }}`.
- **Lookup plugins** - pull external data: `{{ lookup('env', 'HOME') }}`.
- **Connection plugins** - `ssh`, `local`, `docker` (set via `ansible_connection`).
- **Callback plugins** - change console output (e.g. the `yaml` or `minimal` stdout callback).

Try a lookup and a filter in `plugins-demo.yml`:

```yaml
---
- name: Plugins demo
  hosts: local
  tasks:
    - ansible.builtin.debug:
        msg: >
          Home is {{ lookup('env', 'HOME') }};
          uppercase: {{ 'ansible' | upper }};
          list length: {{ [1,2,3] | length }}
```

### 6. Where modules live: collections

Modern Ansible groups modules into **collections** using the
`namespace.collection.module` naming you have seen (e.g. `ansible.builtin.copy`,
`community.general.<module>`). `ansible.builtin` is always available; others are installed from
Galaxy (covered in Lab 11).

## Verification

- `ansible-doc ansible.builtin.copy` displays documentation.
- `modules-demo.yml` creates `/tmp/demo/hello.txt` with two lines.
- The plugins demo prints your home directory and a length of `3`.

## Challenge

1. Use `ansible-doc` to find a module that creates a symbolic link, then write a task that does.
2. Replace a `shell` pipe with a combination of a module + filter where possible.
3. Use the `lookup('file', ...)` plugin to read a local file into a variable.

## Key takeaways

- Modules do work on hosts; plugins extend Ansible's own behavior.
- `ansible-doc` is the authoritative, offline reference.
- Prefer specific idempotent modules over raw `command`/`shell`.

---

**Next lab:** [Lab 09 - Ansible Handlers](09-handlers.md)
