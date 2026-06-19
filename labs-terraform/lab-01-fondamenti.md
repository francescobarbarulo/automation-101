# Lab 01 - Terraform Fundamentals

## Learning Objectives

- Understand the basic structure of a Terraform configuration file
- Create a resource using the local provider
- Execute the fundamental workflow: `terraform init`, `terraform plan`, `terraform apply`
- Modify an existing resource and observe the differences in the execution plan
- Remove created resources with `terraform destroy`

## Prerequisites

- **Terraform CLI** installed (version 1.0 or higher) — [Installation Guide](https://developer.hashicorp.com/terraform/install)
- A **text editor** (VS Code with the HashiCorp Terraform extension recommended)
- Terminal access (Bash, PowerShell, or CMD)
- **No cloud account required** (this lab exclusively uses the local provider)

## Instructions

### Step 1: Create the working directory

Create a new directory for your first Terraform project and navigate into it:

```bash
mkdir lab-01-terraform
cd lab-01-terraform
```

### Step 2: Write the configuration

Create a file called `main.tf` with the following content:

```hcl
resource "local_file" "benvenuto" {
  filename = "${path.module}/benvenuto.txt"
  content  = "Ciao dal mio primo file Terraform!"
}
```

Let's analyze this configuration:

- `resource` — keyword that declares a resource to manage
- `"local_file"` — resource type (provider `local`, resource `file`)
- `"benvenuto"` — logical name of the resource (used internally by Terraform)
- `filename` — path of the file to create
- `content` — text content of the file
- `${path.module}` — expression that resolves to the current directory path

### Step 3: Initialize the directory

Run the initialization command:

```bash
terraform init
```

This command:
- Automatically detects that the configuration requires the `local` provider
- Downloads the provider into the `.terraform/` directory
- Generates the `.terraform.lock.hcl` file that locks the provider version

You should see output similar to:

```
Initializing the backend...
Initializing provider plugins...
- Finding latest version of hashicorp/local...
- Installing hashicorp/local v2.x.x...
Terraform has been successfully initialized!
```

### Step 4: Plan the changes

Run the plan command:

```bash
terraform plan
```

The `plan` command shows what Terraform intends to do **without making any actual changes**. It is a safe preview phase.

In the output you will see:

```
  # local_file.benvenuto will be created
  + resource "local_file" "benvenuto" {
      + content              = "Ciao dal mio primo file Terraform!"
      + filename             = "./benvenuto.txt"
      + id                   = (known after apply)
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

The `+` symbol indicates that the resource will be **created**.

### Step 5: Apply the configuration

Run the apply command:

```bash
terraform apply
```

Terraform will display the plan again and ask for confirmation:

```
Do you want to perform these actions?
  Enter a value: yes
```

Type `yes` and press Enter. Terraform will create the `benvenuto.txt` file.

Verify that the file was created:

```bash
cat benvenuto.txt
```

You should see: `Ciao dal mio primo file Terraform!`

### Step 6: Modify the resource and observe the diff

Now modify the resource content in the `main.tf` file:

```hcl
resource "local_file" "benvenuto" {
  filename = "${path.module}/benvenuto.txt"
  content  = "Ciao dal mio primo file Terraform!\nQuesto file è stato modificato."
}
```

Run the plan again to observe the differences:

```bash
terraform plan
```

In the output you will notice the `-/+` symbol indicating a resource to be **replaced**:

```
  # local_file.benvenuto must be replaced
-/+ resource "local_file" "benvenuto" {
      ~ content              = "Ciao dal mio primo file Terraform!" -> "Ciao dal mio primo file Terraform!\nQuesto file è stato modificato."
      ~ id                   = "..." -> (known after apply)
        # (other attributes unchanged)
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```

This is the core of the Terraform workflow: you can always verify what will change **before** applying.

Apply the changes:

```bash
terraform apply
```

Confirm with `yes` and verify the new file content.

### Step 7: Destroy the resources

To remove all resources managed by Terraform, run:

```bash
terraform destroy
```

Terraform will show what will be deleted:

```
  # local_file.benvenuto will be destroyed
  - resource "local_file" "benvenuto" {
      - content  = "Ciao dal mio primo file Terraform!\nQuesto file è stato modificato."
      - filename = "./benvenuto.txt"
      ...
    }

Plan: 0 to add, 0 to change, 1 to destroy.
```

The `-` symbol indicates that the resource will be **deleted**.

Confirm with `yes`. Terraform will remove the created file and update the state.

## Next Lab

Continue to [Lab 02 - Using Terraform Providers](./lab-02-providers.md) to learn how to organize configurations across multiple files, use input variables, outputs, and explicit dependencies.

