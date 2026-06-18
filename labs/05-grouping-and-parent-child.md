# Lab 05 - Grouping and Parent-Child Relationships

## Objectives

- Organize hosts into meaningful groups.
- Build **groups of groups** (parent-child) with `children`.
- Understand how variables are inherited and overridden across the hierarchy.
- Target nested groups in commands.

## Background

Real environments group hosts many ways at once: by role (web, db), by location (eu, us), by
lifecycle (prod, staging). Ansible lets a host belong to many groups, and lets groups contain
other groups, forming a hierarchy. Child group hosts inherit parent group variables.

## Steps

### 1. A multi-dimensional INI inventory

Create `inventory.ini`:

```ini
[web]
web01
web02

[db]
db01

# Parent group "prod" contains the web and db groups
[prod:children]
web
db

[prod:vars]
env=production
dns_server=10.0.0.53
```

`[prod:children]` makes `web` and `db` **child groups** of `prod`. Every host in `web` and `db`
is therefore also in `prod` and inherits `env` and `dns_server`.

### 2. The YAML equivalent

`inventory.yml`:

```yaml
---
all:
  children:
    prod:
      vars:
        env: production
        dns_server: 10.0.0.53
      children:
        web:
          hosts:
            web01:
            web02:
        db:
          hosts:
            db01:
```

### 3. Visualize the hierarchy

```bash
ansible-inventory -i inventory.ini --graph
```

```
@all:
  |--@prod:
  |  |--@web:
  |  |  |--web01
  |  |  |--web02
  |  |--@db:
  |  |  |--db01
```

### 4. Confirm inherited variables

```bash
ansible-inventory -i inventory.ini --host web01
```

`web01` should show `env=production` and `dns_server=10.0.0.53`, inherited from `prod` even
though it was defined only on the parent.

### 5. Variable precedence across the hierarchy

Add a more specific group var. In `inventory.ini` append:

```ini
[web:vars]
env=staging
```

Re-check:

```bash
ansible-inventory -i inventory.ini --host web01
```

Now `web01` shows `env=staging`. **The child group's value wins over the parent's.** General
rule (least to most specific): `all` → parent group → child group → host.

### 6. Target groups in commands

```bash
ansible prod -i inventory.ini --list-hosts   # all hosts under prod
ansible web  -i inventory.ini --list-hosts   # just web hosts
```

### 7. Hosts in multiple groups

A host can be in several independent groups. Add region grouping:

```ini
[eu]
web01
db01

[us]
web02
```

Now `web01` is in `web`, `prod`, and `eu` simultaneously - and inherits variables from all of
them.

## Verification

- `--graph` shows `prod` containing `web` and `db` as nested groups.
- `web01` inherits `dns_server` from `prod` and `env=staging` from `web`.

## Challenge

1. Add a `staging` parent group containing a new `web` host `web03`.
2. Give `prod` and `staging` different `env` values and confirm each host resolves correctly.
3. Make `db01` a member of both `prod` and a new `backup` group.

## Key takeaways

- `children` builds group hierarchies; child hosts inherit parent variables.
- A host can belong to many groups at once.
- More specific scope overrides less specific: host > child group > parent group > all.

---

**Next lab:** [Lab 06 - Ansible Variables and Facts](06-variables-and-facts.md)
