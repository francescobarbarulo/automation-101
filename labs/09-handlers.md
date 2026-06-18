# Lab 09 - Ansible Handlers

## Objectives

- Understand what a handler is and when it runs.
- Trigger handlers with `notify`.
- Control handler timing with `meta: flush_handlers`.
- Use handlers to restart services only when configuration changes.

## Background

A **handler** is a special task that runs **only when notified** by another task, and only if
that task reported `changed`. Handlers are the idiomatic way to "restart the service _only if_
its config actually changed." By default, all notified handlers run **once**, at the **end** of
the play.

## Steps

### 1. The classic pattern

Create `handlers-demo.yml`:

```yaml
---
- name: Handlers demo
  hosts: local
  become: true
  tasks:
    - name: Deploy application config
      ansible.builtin.copy:
        dest: /tmp/app.conf
        content: |
          listen_port = 8080
          log_level = info
      notify: restart app

  handlers:
    - name: restart app
      ansible.builtin.debug:
        msg: ">>> app restarted because config changed <<<"
```

Run it twice:

```bash
ansible-playbook -i inventory.ini handlers-demo.yml
ansible-playbook -i inventory.ini handlers-demo.yml
```

- **First run:** the `copy` task is `changed`, so the handler fires.
- **Second run:** the file is unchanged, so the handler does **not** fire.

This is the whole point: no needless restarts.

### 2. Key rules of handlers

- A handler runs only if at least one notifying task changed.
- Handlers run **after all tasks** in the play, in the **order they are defined** in the
  `handlers:` section - _not_ in notification order.
- A handler notified multiple times still runs **once**.
- The notify name must match the handler's `name` exactly.

### 3. Notifying multiple handlers

```yaml
tasks:
  - name: Update config
    ansible.builtin.copy:
      dest: /tmp/web.conf
      content: "tuned = true\n"
    notify:
      - reload web
      - clear cache

handlers:
  - name: reload web
    ansible.builtin.debug: { msg: "web reloaded" }
  - name: clear cache
    ansible.builtin.debug: { msg: "cache cleared" }
```

### 4. Running handlers immediately

Sometimes you must run pending handlers before continuing (e.g. restart a service before the
next task depends on it):

```yaml
- name: Force pending handlers to run now
  ansible.builtin.meta: flush_handlers
```

### 5. A realistic example (service restart)

On a real managed host this is the everyday use:

```yaml
tasks:
  - name: Install nginx config
    ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: restart nginx

handlers:
  - name: restart nginx
    ansible.builtin.service:
      name: nginx
      state: restarted
```

The service restarts **only** when the rendered config differs from what's on disk.

### 6. Listen: many tasks, one handler topic

`listen` lets several handlers respond to one notification topic:

```yaml
tasks:
  - ansible.builtin.copy: { dest: /tmp/a, content: "a\n" }
    notify: "config changed"

handlers:
  - name: reload service A
    ansible.builtin.debug: { msg: "A reloaded" }
    listen: "config changed"
  - name: reload service B
    ansible.builtin.debug: { msg: "B reloaded" }
    listen: "config changed"
```

## Verification

- First run of `handlers-demo.yml` shows the handler message; second run does not (`changed=0`).
- The recap shows handlers running at the very end of the play.

## Challenge

1. Add a second task that modifies a different file and notifies the same handler. Confirm the
   handler still runs only once.
2. Use `meta: flush_handlers` to force a handler to run mid-play, and observe the new ordering.
3. Convert the demo to use `listen` with two handlers on one topic.

## Key takeaways

- Handlers run only when notified by a `changed` task, once, at the end of the play.
- They keep automation idempotent: restart/reload only on real changes.
- `flush_handlers` and `listen` give you control over timing and fan-out.

---

**Next lab:** [Lab 10 - Ansible Templates](10-templates.md)
