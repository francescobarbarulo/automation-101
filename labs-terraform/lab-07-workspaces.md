# Lab 07 - Workspaces

## Learning Objectives

By the end of this lab you will be able to:

- Understand the concept of workspace in Terraform and its purpose in multi-environment management
- Create and switch between workspaces using Terraform commands
- Use the `terraform.workspace` variable to parameterize configurations based on the active environment
- Deploy the same configuration to distinct environments (dev and prod) with differentiated attributes

## Prerequisites

- **Terraform CLI** (version 1.0 or higher) installed
- A **text editor** (VS Code with HashiCorp Terraform extension recommended)
- Completion of Labs 01-06 (knowledge of HCL, providers, state, lifecycle, remote state, modules)
- **AWS account with configured credentials** — this lab uses the AWS provider to create real cloud resources
- AWS CLI installed and configured (`aws configure`) or environment variables `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` set

## Instructions

### Step 1: Understand Terraform workspaces

**Workspaces** in Terraform allow you to manage multiple state instances for the same configuration. Each workspace has its own separate state file.

Common use cases:
- Manage multiple environments (dev, staging, prod) with a single configuration
- Test changes in an isolated workspace before applying them to the main environment
- Manage infrastructure for different tenants or teams

By default, Terraform uses a workspace called `default`.

### Step 2: Create the project directory

```bash
mkdir lab-07-workspaces
cd lab-07-workspaces
```

### Step 3: Configure the provider

Create the `providers.tf` file:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

### Step 4: Declare variables

Create the `variables.tf` file:

```hcl
variable "aws_region" {
  description = "La regione AWS in cui creare le risorse"
  type        = string
  default     = "eu-south-1"
}

variable "nome_progetto" {
  description = "Nome del progetto per il tagging delle risorse"
  type        = string
  default     = "lab07-workspaces"
}
```

### Step 5: Create the workspace-aware configuration

Create the `main.tf` file. The key is using `terraform.workspace` to differentiate behavior:

```hcl
locals {
  # Instance type map per workspace
  instance_type = {
    dev  = "t3.micro"
    prod = "t3.small"
  }

  # Instance count map per workspace
  instance_count = {
    dev  = 1
    prod = 2
  }

  # Environment map for tagging
  environment = {
    dev  = "development"
    prod = "production"
  }
}
```

`terraform.workspace` returns the name of the active workspace as a string. By using it as a key in maps, you can parameterize any attribute.

Add the resources to the `main.tf` file:

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_security_group" "istanza" {
  name        = "${var.nome_progetto}-${terraform.workspace}-sg"
  description = "Security group per ${local.environment[terraform.workspace]}"

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.nome_progetto}-${terraform.workspace}-sg"
    Environment = local.environment[terraform.workspace]
    Workspace   = terraform.workspace
  }
}

resource "aws_instance" "server" {
  count = local.instance_count[terraform.workspace]

  ami           = data.aws_ami.amazon_linux.id
  instance_type = local.instance_type[terraform.workspace]

  vpc_security_group_ids = [aws_security_group.istanza.id]

  tags = {
    Name        = "${var.nome_progetto}-${terraform.workspace}-${count.index + 1}"
    Environment = local.environment[terraform.workspace]
    Workspace   = terraform.workspace
  }
}
```

Observe how each resource uses `terraform.workspace` to:
- Differentiate names (avoiding conflicts between environments)
- Select different parameters from local maps
- Tag resources with the correct environment

### Step 6: Declare outputs

Create the `output.tf` file:

```hcl
output "workspace_corrente" {
  description = "Il nome del workspace Terraform attivo"
  value       = terraform.workspace
}

output "ambiente" {
  description = "L'ambiente corrispondente al workspace"
  value       = local.environment[terraform.workspace]
}

output "instance_type" {
  description = "Il tipo di istanza utilizzato"
  value       = local.instance_type[terraform.workspace]
}

output "instance_ids" {
  description = "Gli ID delle istanze EC2 create"
  value       = aws_instance.server[*].id
}
```

### Step 7: Initialize and create workspaces

Initialize the project:

```bash
terraform init
```

View the current workspace:

```bash
terraform workspace show
```

The output will be `default`. Now create workspaces for the two environments:

```bash
terraform workspace new dev
terraform workspace new prod
```

List all available workspaces:

```bash
terraform workspace list
```

The output will be similar to:

```
  default
* prod
  dev
```

The asterisk (`*`) indicates the active workspace.

### Step 8: Deploy in the dev workspace

Switch to the `dev` workspace:

```bash
terraform workspace select dev
```

Apply the configuration:

```bash
terraform apply
```

Observe that:
- **1 instance** is created (as defined in `instance_count["dev"]`)
- The type is `t3.micro`
- Tags contain `Environment = "development"` and `Workspace = "dev"`

Verify the outputs:

```bash
terraform output
```

### Step 9: Deploy in the prod workspace

Switch to the `prod` workspace:

```bash
terraform workspace select prod
```

Apply the same configuration:

```bash
terraform apply
```

Observe the differences compared to the dev workspace:
- **2 instances** are created (as defined in `instance_count["prod"]`)
- The type is `t3.small`
- Tags contain `Environment = "production"` and `Workspace = "prod"`

Each workspace maintains its own independent state. Resources created in the `dev` workspace are not visible to or affected by the `prod` workspace.

### Step 10: Compare environments

Switch between workspaces to compare:

```bash
terraform workspace select dev
terraform output

terraform workspace select prod
terraform output
```

You can also check where Terraform stores the separate states:

```bash
ls terraform.tfstate.d/
```

You will see a directory for each non-default workspace:

```
dev/
prod/
```

### Step 11: Cleanup

To delete resources, you must run destroy in **each workspace** separately:

```bash
terraform workspace select dev
terraform destroy

terraform workspace select prod
terraform destroy
```

After destroying the resources, you can delete the workspaces:

```bash
terraform workspace select default
terraform workspace delete dev
terraform workspace delete prod
```

## End

Congratulations! You have completed all 7 Terraform labs. You now have hands-on experience with the core Terraform workflow, providers, state management, lifecycle rules, remote state, modules, and workspaces.

