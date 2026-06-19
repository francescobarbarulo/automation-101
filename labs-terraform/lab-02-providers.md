# Lab 02 - Using Terraform Providers

## Learning Objectives

By the end of this lab you will be able to:

- Organize a Terraform configuration across multiple files following the standard structure (`main.tf`, `variables.tf`, `providers.tf`, `output.tf`)
- Declare input variables and assign values through different strategies (default values, CLI flags, `.tfvars` files, environment variables)
- Declare and use output values to expose resource attributes
- Use `depends_on` to define explicit dependencies between resources

## Prerequisites

- **Terraform CLI** installed (version 1.0 or higher)
- A **text editor** (VS Code, Vim, or similar)
- Basic knowledge of HCL syntax (completion of Lab 01)
- **No cloud account required** — this lab exclusively uses the `local` provider

## Instructions

### Step 1: Create the multi-file structure

In this step you will create a project directory with the standard Terraform file structure.

1. Create a new directory for the lab:

```bash
mkdir lab-02-providers
cd lab-02-providers
```

2. Create the following empty files:
   (Linux/macOS)

```bash
touch providers.tf main.tf variables.tf output.tf
```
  (Windows)
```bash
ni providers.tf,main.tf,variables.tf,output.tf
```

This is the recommended structure for every Terraform project:

| File | Purpose |
|------|---------|
| `providers.tf` | Configuration of required providers |
| `main.tf` | Resource definitions |
| `variables.tf` | Input variable declarations |
| `output.tf` | Output value declarations |

### Step 2: Configure the provider

Open the `providers.tf` file and add the `local` provider configuration:

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}
```

This block declares that the project requires the `hashicorp/local` provider with a version compatible with 2.x.

### Step 3: Declare input variables

Open the `variables.tf` file and add the following variable declarations:

```hcl
variable "nome_file_principale" {
  description = "Nome del file principale da creare"
  type        = string
  default     = "output-principale.txt"
}

variable "contenuto_file" {
  description = "Contenuto da scrivere nel file principale"
  type        = string
  default     = "Questo file è stato creato con Terraform!"
}

variable "nome_file_secondario" {
  description = "Nome del file secondario (dipendente dal primo)"
  type        = string
  default     = "output-secondario.txt"
}
```

Note how each variable has:
- A **name** identifier
- A **description** explaining its purpose
- A **type** defining the expected value format
- An optional **default** value

### Step 4: Define resources with explicit dependencies

Open the `main.tf` file and add the following resources:

```hcl
resource "local_file" "file_principale" {
  filename = var.nome_file_principale
  content  = var.contenuto_file
}

resource "local_file" "file_secondario" {
  filename = var.nome_file_secondario
  content  = "Questo file dipende da: ${local_file.file_principale.filename}"

  depends_on = [
    local_file.file_principale
  ]
}
```

The `depends_on` meta-argument in the second resource ensures that `file_principale` is created **before** `file_secondario`. Terraform normally determines the order automatically by analyzing references, but `depends_on` allows you to define explicit dependencies when needed — for example, for logical dependencies not expressed in references.

### Step 5: Declare output values

Open the `output.tf` file and add:

```hcl
output "percorso_file_principale" {
  description = "Percorso completo del file principale creato"
  value       = local_file.file_principale.filename
}

output "percorso_file_secondario" {
  description = "Percorso completo del file secondario creato"
  value       = local_file.file_secondario.filename
}

output "id_file_principale" {
  description = "ID della risorsa file principale"
  value       = local_file.file_principale.id
}
```

Outputs expose resource attributes that can be queried after applying with `terraform output`.

### Step 6: Initialize and apply the configuration

1. Initialize the project:

```bash
terraform init
```

2. View the execution plan:

```bash
terraform plan
```

3. Apply the configuration using the default values:

```bash
terraform apply
```

Confirm with `yes` when prompted. Terraform will create the two files in the correct order thanks to `depends_on`.

4. Verify the outputs:

```bash
terraform output
```

### Step 7: Assign variables via CLI flags

You can override default values by passing variables directly from the command line with the `-var` flag:

```bash
terraform apply -var="nome_file_principale=file-da-cli.txt" -var="contenuto_file=Contenuto passato da CLI"
```

This strategy is useful for occasional values or quick tests.

### Step 8: Assign variables via .tfvars file

1. Create a file called `terraform.tfvars`:

```hcl
nome_file_principale  = "file-da-tfvars.txt"
contenuto_file        = "Contenuto caricato dal file terraform.tfvars"
nome_file_secondario  = "secondo-file-tfvars.txt"
```

2. Apply the configuration — Terraform automatically loads `terraform.tfvars`:

```bash
terraform apply
```

You can also create files with custom names (e.g., `dev.tfvars`) and specify them with the `-var-file` flag:

```bash
terraform apply -var-file="dev.tfvars"
```

### Step 9: Assign variables via environment variables

Terraform recognizes environment variables with the `TF_VAR_` prefix. The variable name follows the prefix:

**Linux/macOS:**

```bash
export TF_VAR_nome_file_principale="file-da-env.txt"
export TF_VAR_contenuto_file="Contenuto da variabile d'ambiente"
terraform apply
```

**Windows (PowerShell):**

```powershell
$env:TF_VAR_nome_file_principale = "file-da-env.txt"
$env:TF_VAR_contenuto_file = "Contenuto da variabile d'ambiente"
terraform apply
```

In this case, apply will not make changes because there is a precedence order for defining variable values, and we have already specified the terraform.tfvars file.

### Step 10: Understand variable precedence

Terraform applies variable values according to this precedence order (from lowest to highest):

1. `default` value in the variable declaration
2. `TF_VAR_*` environment variables
3. `terraform.tfvars` file (automatically loaded)
4. `*.auto.tfvars` files (automatically loaded in alphabetical order)
5. File specified with `-var-file`
6. `-var` flag on the command line

The value with the highest precedence wins. This allows you to have base values in the code and override them for specific environments.

### Step 11: Resource cleanup

At the end of the exercise, delete all created resources:

```bash
terraform destroy
```

Confirm with `yes` when prompted.

## Next Lab

Continue to [Lab 03 - State File](./lab-03-state.md) to learn how Terraform tracks resources, metadata, and dependencies through the state file.