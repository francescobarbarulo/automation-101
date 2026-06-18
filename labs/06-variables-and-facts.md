# Lab 06 - Ansible Variables and Facts

## Objectives

- Define variables in playbooks, inventory, and var files.
- Use variables with Jinja2 (`{{ }}`).
- Gather and use **facts** (auto-discovered host data).
- Register the output of a task into a variable.
- Understand variable precedence at a practical level.

## Background

**Variables** let you parameterize automation so the same playbook works across hosts and
environments. **Facts** are variables Ansible discovers automatically about each host (OS,
IP, memory, etc.). **Registered variables** capture a task's result for later use.

## Steps

### 1. Variables defined in a playbook

Create `vars-demo.yml`:

```yaml
---
- name: Variables demo
  hosts: local
  vars:
    greeting: "Hello"
    package_name: nginx
  tasks:
    - name: Use a variable
      ansible.builtin.debug:
        msg: "{{ greeting }}, we will install {{ package_name }}"
```

Run it (using the Lab 01 inventory):

```bash
ansible-playbook -i inventory.ini vars-demo.yml
```

> Note: when a value _starts_ with `{{ }}`, quote the whole string: `msg: "{{ var }}"`.

### 2. Variables from the command line

`--extra-vars` (`-e`) has the highest everyday precedence:

```bash
ansible-playbook -i inventory.ini vars-demo.yml -e "package_name=apache2"
```

### 3. Variables from files (`group_vars` / `host_vars`)

As seen in Lab 04, files in `group_vars/<group>.yml` and `host_vars/<host>.yml` are loaded
automatically. You can also include a vars file explicitly:

`myvars.yml`:

```yaml
---
app_port: 8080
app_user: webadmin
```

```yaml
- name: Use external vars
  hosts: local
  vars_files:
    - myvars.yml
  tasks:
    - ansible.builtin.debug:
        msg: "{{ app_user }} listens on {{ app_port }}"
```

### 4. Facts

Facts are gathered automatically at the start of a play (unless disabled). Explore them:

```bash
ansible local -i inventory.ini -m setup | less
```

Use facts in a play - create `facts-demo.yml`:

```yaml
---
- name: Facts demo
  hosts: local
  tasks:
    - name: Show OS and memory
      ansible.builtin.debug:
        msg: >
          OS: {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }},
          Free RAM: {{ ansible_facts['memfree_mb'] }} MB,
          Hostname: {{ ansible_facts['hostname'] }}
```

Common facts (modern dictionary syntax): `ansible_facts['distribution']`,
`ansible_facts['os_family']`, `ansible_facts['default_ipv4']['address']`,
`ansible_facts['processor_vcpus']`, `ansible_facts['hostname']`.

> **Modern vs. legacy fact syntax.** Older material uses top-level variables like
> `ansible_os_family` or `ansible_hostname`. These only exist when the
> `inject_facts_as_vars` setting is on (the default, for now). The modern, recommended form
> reads facts out of the single `ansible_facts` dictionary - e.g. `ansible_facts['os_family']`
>
> - which avoids cluttering the variable namespace and keeps working when injection is turned
>   off. Prefer the `ansible_facts[...]` form in new code. Bracket (`['key']`) and dot
>   (`ansible_facts.os_family`) access are equivalent; brackets are safer for keys that collide
>   with Python/Jinja attributes.

> Speed tip: add `gather_facts: false` to a play that doesn't need facts.

### 5. Registered variables

Capture a command's output and act on it:

```yaml
---
- name: Register demo
  hosts: local
  tasks:
    - name: Run a command
      ansible.builtin.command: date +%A
      register: today

    - name: Show captured output
      ansible.builtin.debug:
        msg: "Today is {{ today.stdout }}"

    - name: Conditional based on a fact
      ansible.builtin.debug:
        msg: "This is a Debian-family system"
      when: ansible_facts['os_family'] == "Debian"
```

`register` stores a dict with keys like `stdout`, `rc` (return code), and `changed`.

### 6. Variable precedence (practical summary)

From lowest to highest priority (simplified):

1. Role defaults (`roles/x/defaults/main.yml`)
2. Inventory group vars
3. Inventory host vars
4. Play `vars`
5. `vars_files`
6. Registered vars / set_facts
7. `--extra-vars` (`-e`) - **always wins**

When in doubt, `-e` overrides everything.

## Verification

- `facts-demo.yml` prints your real OS, hostname, and free memory.
- The registered variable `today.stdout` prints the current weekday.
- `-e "package_name=apache2"` changes the output of `vars-demo.yml`.

## Challenge

1. Write a play that prints `ansible_facts['default_ipv4']['address']` only `when` the host is in
   the `Debian` OS family.
2. Register the output of `whoami` and print it.
3. Override a `group_vars` value using `-e` and confirm `-e` wins.

## Key takeaways

- Variables come from many sources; `-e` has the final say.
- Facts are free host metadata gathered automatically - use them in conditions and messages.
- `register` turns task output into a variable for later steps.

---

**Next lab:** [Lab 07 - Ansible Playbooks](07-ansible-playbooks.md)
