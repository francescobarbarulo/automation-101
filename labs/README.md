# Ansible for Beginners - Lab Guides

Hands-on labs for the **Ansible for Beginners** course. Each lab is self-contained, builds on the previous one, and ends with verification steps and a short challenge.

## Prerequisites

- A control node running Linux (Ubuntu 22.04+, Fedora, or similar) or WSL2 on Windows.
- Python 3.9+ installed.
- One or more managed hosts you can reach over SSH. If you have none, the labs explain how to use `localhost` and lightweight Docker/Multipass targets.
- A regular user with `sudo` privileges.

## Install Ansible (control node only)

```bash
python3 -m pip install --user ansible
ansible --version
```

Ansible is **agentless**: nothing is installed on the managed hosts beyond Python and SSH, which most Linux systems already have.

## Lab Index

| #   | Lab                                                        | Topic                              |
| --- | ---------------------------------------------------------- | ---------------------------------- |
| 01  | [Ansible Introduction](01-ansible-introduction.md)         | Ansible Introduction for Beginners |
| 02  | [Understanding YAML](02-understanding-yaml.md)             | YAML syntax                        |
| 03  | [Ansible Inventory](03-ansible-inventory.md)               | Inventory basics                   |
| 04  | [Inventory Formats](04-inventory-formats.md)               | INI vs YAML inventories            |
| 05  | [Grouping & Parent-Child](05-grouping-and-parent-child.md) | Groups and group-of-groups         |
| 06  | [Variables and Facts](06-variables-and-facts.md)           | Variables, facts, precedence       |
| 07  | [Ansible Playbooks](07-ansible-playbooks.md)               | Writing and running playbooks      |
| 08  | [Modules and Plugins](08-modules-and-plugins.md)           | Core modules and plugins           |
| 09  | [Handlers](09-handlers.md)                                 | Handlers and notifications         |
| 10  | [Templates](10-templates.md)                               | Jinja2 templating                  |
| 11  | [Roles and Collections](11-roles-and-collections.md)       | Roles and Ansible Galaxy           |

## How to use these labs

1. Work through them in order - later labs reuse the inventory and project layout from earlier ones.
2. Type the commands yourself rather than copy-pasting; muscle memory matters.
3. Each lab ends with a **Challenge** - do it before moving on.

## Suggested project layout

By the end of the course your working directory should look like this:

```
ansible-labs/
├── ansible.cfg
├── inventory.ini
├── group_vars/
├── host_vars/
├── playbooks/
├── templates/
└── roles/
```

Create the base directory now:

```bash
mkdir -p ~/ansible-labs && cd ~/ansible-labs
```
