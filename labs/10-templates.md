# Lab 10 - Ansible Templates

## Objectives

- Render configuration files from **Jinja2** templates with the `template` module.
- Use variables, facts, conditionals, and loops inside templates.
- Apply filters and default values.
- Understand `template` vs `copy`.

## Background

A **template** is a text file containing Jinja2 expressions (`{{ }}`) and statements
(`{% %}`). The `ansible.builtin.template` module renders it on the control node using the
host's variables and facts, then copies the result to the managed host. This is how you generate
host-specific config files from one source.

## Steps

### 1. Project layout

By convention templates live in a `templates/` directory and end in `.j2`:

```bash
mkdir -p templates
```

### 2. A first template

Create `templates/motd.j2`:

```jinja
Welcome to {{ ansible_facts['hostname'] }}
OS: {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}
Managed by Ansible - do not edit by hand.
```

Playbook `templates-demo.yml`:

```yaml
---
- name: Render templates
  hosts: local
  become: true
  tasks:
    - name: Render the MOTD
      ansible.builtin.template:
        src: templates/motd.j2
        dest: /tmp/motd.rendered
        mode: "0644"

    - name: Show the result
      ansible.builtin.command: cat /tmp/motd.rendered
      register: out
      changed_when: false
    - ansible.builtin.debug:
        var: out.stdout_lines
```

Run it:

```bash
ansible-playbook -i inventory.ini templates-demo.yml
```

### 3. Variables and the `default` filter

Templates can reference any variable. Provide a fallback when one may be undefined:

```jinja
server_name = {{ server_name | default('localhost') }}
worker_count = {{ workers | default(4) }}
```

### 4. Conditionals in templates

```jinja
{% if enable_ssl | default(false) %}
listen 443 ssl;
{% else %}
listen 80;
{% endif %}
```

### 5. Loops in templates

A common pattern - generate an upstream block from a list. Add to your play:

```yaml
vars:
  backends:
    - 10.0.0.11:8080
    - 10.0.0.12:8080
    - 10.0.0.13:8080
```

`templates/upstream.conf.j2`:

```jinja
upstream app {
{% for server in backends %}
    server {{ server }};
{% endfor %}
}
```

Render it:

```yaml
- name: Render upstream config
  ansible.builtin.template:
    src: templates/upstream.conf.j2
    dest: /tmp/upstream.conf
```

### 6. Useful filters

```jinja
{{ name | upper }}              {# UPPERCASE #}
{{ path | basename }}           {# last path component #}
{{ items | join(', ') }}        {# list -> comma string #}
{{ password | default('') | length }}
{{ ansible_facts['memtotal_mb'] | int }}
```

### 7. Whitespace control

Jinja can leave blank lines. Trim them with `-`:

```jinja
{% for s in backends -%}
server {{ s }};
{% endfor -%}
```

### 8. `template` vs `copy`

| Use `copy` when...      | Use `template` when...               |
| ----------------------- | ------------------------------------ |
| The file is static      | The file contains variables/logic    |
| No host-specific values | Per-host values, loops, conditionals |

Both are idempotent: the file is only rewritten (and handlers only notified) when the rendered
content differs from what is already on the host - pair `template` with a handler from Lab 09 to
restart a service on config changes.

## Verification

- `/tmp/motd.rendered` contains your real hostname and OS.
- The upstream config lists all three backend servers, one per line.

## Challenge

1. Create a template for an `nginx`-style site config that uses a `server_name` variable, a
   `listen` port with a `default`, and a `for` loop over a `locations` list.
2. Use the `default` filter so the playbook still renders when an optional variable is unset.
3. Add a handler that "restarts" the service only when the template changes.

## Key takeaways

- `template` = Jinja2 rendered with host vars/facts, then copied to the host.
- Templates support variables, `default`, conditionals, loops, and filters.
- Use `template` for dynamic config, `copy` for static files; both are idempotent.

---

**Next lab:** [Lab 11 - Ansible Roles and Collections](11-roles-and-collections.md)
