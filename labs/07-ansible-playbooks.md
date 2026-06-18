# Lab 07 - Ansible Playbooks

## Objectives

- Understand the anatomy of a playbook: plays, tasks, modules.
- Use `become` for privilege escalation.
- Apply loops, conditionals, and tags.
- Run playbooks with `--check` (dry run) and `--diff`.

## Background

A **playbook** is a YAML file containing one or more **plays**. A play maps a set of **hosts**
to a list of **tasks**, and each task calls a **module**. Playbooks are the primary way you use
Ansible.

## Steps

### 1. Anatomy of a play

```yaml
---
- name: Configure web servers # play name
  hosts: webservers # which hosts
  become: true # run tasks with sudo
  vars:
    package: nginx
  tasks: # list of tasks
    - name: Install the web server # task name
      ansible.builtin.apt: # module
        name: "{{ package }}" # module arguments
        state: present
```

### 2. A complete, runnable playbook

For this lab we target `local`. Create `site.yml`:

```yaml
---
- name: Set up a simple web page
  hosts: local
  become: true
  tasks:
    - name: Ensure a content directory exists
      ansible.builtin.file:
        path: /tmp/ansible-web
        state: directory
        mode: "0755"

    - name: Create an index page
      ansible.builtin.copy:
        dest: /tmp/ansible-web/index.html
        content: "Served by Ansible on {{ ansible_facts['hostname'] }}\n"
        mode: "0644"

    - name: Show the file we created
      ansible.builtin.command: cat /tmp/ansible-web/index.html
      register: page
      changed_when: false

    - name: Display it
      ansible.builtin.debug:
        msg: "{{ page.stdout }}"
```

Run it:

```bash
ansible-playbook -i inventory.ini site.yml
```

Read the **recap** at the end: `ok=`, `changed=`, `failed=`. Run it a **second time** - note
that `changed=0` for the file tasks. That is **idempotency** in action.

### 3. Privilege escalation with `become`

`become: true` runs tasks as root via sudo. If sudo needs a password:

```bash
ansible-playbook -i inventory.ini site.yml --ask-become-pass
```

### 4. Loops

```yaml
- name: Install several packages
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - curl
    - htop
```

### 5. Conditionals

```yaml
- name: Only on Debian-family systems
  ansible.builtin.debug:
    msg: "apt is available here"
  when: ansible_facts['os_family'] == "Debian"
```

### 6. Tags - run part of a playbook

```yaml
- name: Install packages
  ansible.builtin.apt:
    name: git
    state: present
  tags: [packages]
```

```bash
ansible-playbook -i inventory.ini site.yml --tags packages
ansible-playbook -i inventory.ini site.yml --skip-tags packages
```

### 7. Dry run and diff

```bash
# Show what WOULD change without changing anything
ansible-playbook -i inventory.ini site.yml --check

# Show file content differences
ansible-playbook -i inventory.ini site.yml --check --diff
```

### 8. Always check syntax first

```bash
ansible-playbook site.yml --syntax-check
```

## Verification

- First run shows `changed`; second run shows `changed=0` (idempotent).
- `/tmp/ansible-web/index.html` contains your hostname.
- `--check` reports changes without making them.

## Challenge

1. Extend `site.yml` with a loop that creates three files in `/tmp/ansible-web`.
2. Add a `when` condition so a task runs only on a specific OS family.
3. Tag the file-creation tasks `content` and run only them with `--tags content`.

## Key takeaways

- Playbook = plays; play = hosts + tasks; task = a module call.
- `become` escalates privileges; `loop`/`when` add iteration and logic.
- `--syntax-check`, `--check`, and `--diff` make runs safe and predictable.

---

**Next lab:** [Lab 08 - Ansible Modules and Plugins](08-modules-and-plugins.md)
