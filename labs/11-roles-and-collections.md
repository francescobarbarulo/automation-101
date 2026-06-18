# Lab 11 - Ansible Roles and Collections

## Objectives

- Understand why roles exist and their standard directory structure.
- Create a role with `ansible-galaxy` and use it in a playbook.
- Install and use roles/collections from **Ansible Galaxy**.
- Manage dependencies with a `requirements.yml`.

## Background

As playbooks grow, you need structure and reuse. A **role** bundles tasks, handlers, templates,
files, variables, and defaults into a standard layout that can be reused across projects and
shared. A **collection** is a distributable package that can contain roles, modules, and
plugins. **Ansible Galaxy** (galaxy.ansible.com) is the public hub for both.

## Steps

### 1. The role directory structure

```bash
mkdir -p roles
ansible-galaxy init roles/webserver
```

This scaffolds:

```
roles/webserver/
├── defaults/main.yml     # default variables (lowest precedence)
├── files/                # static files for copy
├── handlers/main.yml     # handlers
├── meta/main.yml         # role metadata & dependencies
├── tasks/main.yml        # the main task list (entry point)
├── templates/            # Jinja2 templates
├── vars/main.yml         # role variables (higher precedence)
└── README.md
```

Ansible automatically loads `main.yml` from each directory.

### 2. Fill in the role

`roles/webserver/defaults/main.yml`:

```yaml
---
web_root: /tmp/webserver
web_message: "Hello from a role"
```

`roles/webserver/tasks/main.yml`:

```yaml
---
- name: Ensure web root exists
  ansible.builtin.file:
    path: "{{ web_root }}"
    state: directory
    mode: "0755"

- name: Render the index page
  ansible.builtin.template:
    src: index.html.j2
    dest: "{{ web_root }}/index.html"
    mode: "0644"
  notify: reload web
```

`roles/webserver/templates/index.html.j2`:

```jinja
<h1>{{ web_message }}</h1>
<p>Rendered on {{ ansible_facts['hostname'] }}</p>
```

`roles/webserver/handlers/main.yml`:

```yaml
---
- name: reload web
  ansible.builtin.debug:
    msg: "web reloaded"
```

### 3. Use the role in a playbook

Create `roles-demo.yml`:

```yaml
---
- name: Deploy with a role
  hosts: local
  become: true
  roles:
    - webserver
```

Run it:

```bash
ansible-playbook -i inventory.ini roles-demo.yml
```

Notice how clean the playbook is - all the detail lives in the role.

### 4. Overriding role variables

Because `web_message` is a **default**, any higher-precedence source overrides it:

```yaml
- name: Deploy with a role
  hosts: local
  become: true
  roles:
    - role: webserver
      vars:
        web_message: "Customized!"
```

### 5. Roles via `tasks` with `include_role` / `import_role`

Besides the `roles:` keyword you can invoke roles inside `tasks:`:

```yaml
tasks:
  - name: Run the role partway through a play
    ansible.builtin.include_role:
      name: webserver
```

- `import_role` - included **statically** at parse time.
- `include_role` - included **dynamically** at run time (can be looped / conditional).

### 6. Ansible Galaxy - install community content

Install a role:

```bash
ansible-galaxy role install geerlingguy.nginx
ansible-galaxy role list
```

Install a collection:

```bash
ansible-galaxy collection install community.general
ansible-galaxy collection list
```

After installing, reference collection content by its FQCN, e.g.
`community.general.<module>`.

### 7. Pin dependencies with `requirements.yml`

For reproducible projects, list what you need and install in one shot.
`requirements.yml`:

```yaml
---
roles:
  - name: geerlingguy.nginx
    version: "3.1.4"

collections:
  - name: community.general
    version: ">=8.0.0"
```

```bash
ansible-galaxy install -r requirements.yml
```

### 8. Role dependencies (`meta/main.yml`)

A role can declare other roles it depends on:

```yaml
# roles/webserver/meta/main.yml
dependencies:
  - role: common
```

Dependencies run before the role that requires them.

## Verification

- `ansible-galaxy init` created the full directory tree under `roles/webserver/`.
- `roles-demo.yml` produces `/tmp/webserver/index.html` with your hostname.
- Overriding `web_message` via play `vars` changes the rendered page.

## Challenge

1. Create a second role `common` that creates a `/tmp/common.txt` marker file, and make
   `webserver` depend on it via `meta/main.yml`.
2. Move the inventory's `web_root` value into `group_vars` and confirm the role still works.
3. Write a `requirements.yml` pinning one role and one collection, and install it.

## Key takeaways

- Roles give playbooks a standard, reusable structure; `ansible-galaxy init` scaffolds it.
- `defaults/` are easily overridable; `vars/` are stronger; `tasks/main.yml` is the entry point.
- Galaxy + `requirements.yml` make community content and dependencies reproducible.

---

🎉 **Course complete!** You now have a working `ansible-labs/` project covering inventories,
variables, playbooks, modules, handlers, templates, and roles - the full beginner toolkit.
