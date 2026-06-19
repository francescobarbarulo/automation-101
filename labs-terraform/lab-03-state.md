# Lab 03 - State File

## Learning Objectives

- Understand the purpose of the state file (`terraform.tfstate`)
- Observe how Terraform tracks resource **metadata**, particularly **dependencies**
- Verify that Terraform uses the dependencies recorded in the state to destroy resources in the correct order
- Experiment with what happens when resources are removed from the configuration

## Prerequisites

- **Terraform CLI** (version 1.0 or higher) installed
- A **text editor** (VS Code recommended)
- Completion of Lab 01 and Lab 02
- **No cloud account required** — this lab exclusively uses the `local` provider

## Instructions

### Step 1: Create the working directory

```bash
mkdir lab-03-state
cd lab-03-state
```

### Step 2: Create resources with dependencies

Create a `main.tf` file with three resources linked to each other via `depends_on`:

```hcl
resource "local_file" "configurazione" {
  filename = "${path.module}/configurazione.txt"
  content  = "Questo è il file di configurazione base."
}

resource "local_file" "dati" {
  filename = "${path.module}/dati.txt"
  content  = "Questi sono i dati dell'applicazione."

  depends_on = [local_file.configurazione]
}

resource "local_file" "report" {
  filename = "${path.module}/report.txt"
  content  = "Report: configurazione e dati sono pronti."

  depends_on = [local_file.configurazione, local_file.dati]
}
```

The dependency structure is:

```
configurazione ← dati ← report
```

- `dati` depends on `configurazione`
- `report` depends on both

### Step 3: Initialize and apply

```bash
terraform init
terraform apply -auto-approve
```

Terraform will create the three resources respecting the dependency order: first `configurazione`, then `dati`, finally `report`.

### Step 4: Inspect the state file

Now open the `terraform.tfstate` file with your editor. It is a JSON file. Look for the `"resources"` section — you will notice that for each resource Terraform stores:

- **type** and **name** — the identity of the resource
- **instances.attributes** — all current attributes (content, path, hash, etc.)
- **dependencies** — the list of dependencies for that resource

For the `report` resource, you will see something similar to:

```json
{
  "mode": "managed",
  "type": "local_file",
  "name": "report",
  "instances": [
    {
      "attributes": { ... },
      "dependencies": [
        "local_file.configurazione",
        "local_file.dati"
      ]
    }
  ]
}
```

These dependencies are **metadata** that Terraform records in the state. They are used to know in which order to operate on resources, even when the configuration file is no longer available.

You can also use the command:

```bash
terraform state list
```

To list all tracked resources:

```
local_file.configurazione
local_file.dati
local_file.report
```

### Step 5: Observe the destruction order

Run:

```bash
terraform destroy -auto-approve
```

Watch the output carefully. Terraform destroys resources in the **reverse order** of their dependencies:

1. First `local_file.report` (depends on everything)
2. Then `local_file.dati` (depends on `configurazione`)
3. Finally `local_file.configurazione` (no dependencies)

Terraform knows this order thanks to the dependency metadata stored in the state file.

### Step 6: Reapply and remove a resource from the configuration

Reapply the full configuration:

```bash
terraform apply -auto-approve
```

Now **remove** the `report` resource from the `main.tf` file. The file becomes:

```hcl
resource "local_file" "configurazione" {
  filename = "${path.module}/configurazione.txt"
  content  = "Questo è il file di configurazione base."
}

resource "local_file" "dati" {
  filename = "${path.module}/dati.txt"
  content  = "Questi sono i dati dell'applicazione."

  depends_on = [local_file.configurazione]
}
```

Run the plan:

```bash
terraform plan
```

Terraform detects that `local_file.report` is present in the state but **no longer in the configuration**, and proposes to destroy it:

```
  # local_file.report will be destroyed
  # (because local_file.report is not in configuration)
  - resource "local_file" "report" { ... }

Plan: 0 to add, 0 to change, 1 to destroy.
```

Apply:

```bash
terraform apply -auto-approve
```

Terraform knows which resource to delete because it tracks it in the state file, even if it no longer exists in the code.

### Step 7: Remove all remaining resources

Now completely empty `main.tf` (or remove both resources):

```hcl
# main.tf empty
```

Run:

```bash
terraform apply -auto-approve
```

Terraform will destroy `dati` before `configurazione`, respecting the reverse dependency order — even though there is no longer any dependency information in the code. The state file still contains that metadata.

### Key Concepts

| What the state tracks | What it's used for |
|----------------------|-------------------|
| Resource attributes | Terraform compares the current state with the configuration to determine changes |
| Dependencies | Terraform correctly orders resource creation and destruction |
| Resource identities | Terraform knows which resources it manages, even when they are removed from code |

**Without the state file**, Terraform would not know:
- Which resources it previously created
- In which order to destroy them
- That a resource removed from code must be deleted from the infrastructure

### Step 8: Cleanup

Verify that no residual resources remain:

```bash
terraform state list
```

If the list is empty, the lab is complete. You can remove the working directory.

## Next Lab

Continue to [Lab 04 - Working with Terraform](./lab-04-working-with-terraform.md) to learn about validation, formatting, lifecycle rules, data sources, and creating multiple resource instances with `count` and `for_each`.

