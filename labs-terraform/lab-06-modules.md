# Lab 06 - Modules

## Learning Objectives

By the end of this lab you will be able to:

- Understand the concept of a module in Terraform and its advantages (reusability, organization, encapsulation)
- Create a local Terraform module with exposed input variables and outputs
- Call a local module from the root configuration passing the required parameters
- Use a module from the Terraform Registry in your own configuration
- Compose infrastructure by combining local modules and registry modules

## Prerequisites

- **Terraform CLI** (version 1.0 or higher) installed
- A **text editor** (VS Code with HashiCorp Terraform extension recommended)
- Completion of Labs 01-05 (knowledge of HCL, providers, state, lifecycle, remote state)
- **AWS account with configured credentials** — this lab uses the AWS provider to create real cloud resources
- AWS CLI installed and configured (`aws configure`) or environment variables `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` set

## Instructions

### Step 1: Understand Terraform modules

A **module** in Terraform is simply a directory containing `.tf` files. Every Terraform configuration is technically a module (the "root module").

Modules allow you to:
- **Reuse** configurations without duplicating code
- **Organize** complex infrastructure into logical components
- **Encapsulate** implementation details by exposing only variables and outputs

There are two main types of modules:
- **Local modules**: defined in your project, referenced via relative path
- **Registry modules**: published in the Terraform Registry, downloaded automatically

### Step 2: Create the project structure

Create the directory structure for the lab:

```bash
mkdir -p lab-06-modules/modules/security-group
cd lab-06-modules
```

The final structure will be:

```
lab-06-modules/
├── main.tf
├── variables.tf
├── providers.tf
├── output.tf
└── modules/
    └── security-group/
        ├── main.tf
        ├── variables.tf
        └── output.tf
```

### Step 3: Configure the provider

Create the `providers.tf` file in the root directory:

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

### Step 4: Create the local module — variables.tf

Enter the module directory and create the `modules/security-group/variables.tf` file:

```hcl
variable "sg_name" {
  description = "Il nome da assegnare al security group"
  type        = string
}

variable "sg_description" {
  description = "La descrizione del security group"
  type        = string
  default     = "Security group gestito da Terraform"
}

variable "vpc_id" {
  description = "L'ID della VPC in cui creare il security group"
  type        = string
}

variable "ingress_port" {
  description = "La porta di ingresso da permettere (es. 80, 443)"
  type        = number
}

variable "aws_region" {
  description = "La regione AWS in cui operare"
  type        = string
  default     = "eu-south-1"
}
```

These are the module's **input variables** — the parameters the user must provide when calling the module.

### Step 5: Create the local module — main.tf

Create the `modules/security-group/main.tf` file:

```hcl
resource "aws_security_group" "this" {
  name        = var.sg_name
  description = var.sg_description
  vpc_id      = var.vpc_id

  ingress {
    description = "Consenti traffico in entrata sulla porta ${var.ingress_port}"
    from_port   = var.ingress_port
    to_port     = var.ingress_port
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Consenti tutto il traffico in uscita"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name      = var.sg_name
    Terraform = "true"
  }
}
```

### Step 6: Create the local module — output.tf

Create the `modules/security-group/output.tf` file:

```hcl
output "security_group_id" {
  description = "L'ID del security group creato"
  value       = aws_security_group.this.id
}

output "security_group_name" {
  description = "Il nome del security group creato"
  value       = aws_security_group.this.name
}
```

The module's **outputs** make values available to the calling configuration.

### Step 7: Declare root variables

Create the `variables.tf` file in the root directory:

```hcl
variable "aws_region" {
  description = "La regione AWS"
  type        = string
  default     = "eu-south-1"
}

variable "vpc_name" {
  description = "Il nome della VPC"
  type        = string
  default     = "lab06-vpc"
}

variable "vpc_cidr" {
  description = "Il blocco CIDR per la VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnets" {
  description = "Lista dei CIDR per le subnet pubbliche"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnets" {
  description = "Lista dei CIDR per le subnet private"
  type        = list(string)
  default     = ["10.0.101.0/24", "10.0.102.0/24"]
}

variable "availability_zones" {
  description = "Le availability zone da utilizzare"
  type        = list(string)
  default     = ["eu-south-1a", "eu-south-1b"]
}

variable "sg_name" {
  description = "Il nome del security group"
  type        = string
  default     = "lab06-sg"
}

variable "sg_description" {
  description = "La descrizione del security group"
  type        = string
  default     = "Security group creato dal modulo locale nel Lab 06"
}

variable "ingress_port" {
  description = "La porta di ingresso per il security group"
  type        = number
  default     = 80
}
```

### Step 8: Call the module from the Terraform Registry

Create the `main.tf` file in the root directory. First, call a module from the Terraform Registry — we will use the official VPC module:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.0"

  name = var.vpc_name
  cidr = var.vpc_cidr

  azs             = var.availability_zones
  public_subnets  = var.public_subnets
  private_subnets = var.private_subnets

  enable_nat_gateway = false
  enable_vpn_gateway = false

  tags = {
    Terraform   = "true"
    Environment = "lab06"
  }
}
```

The syntax for registry modules is `source = "<NAMESPACE>/<NAME>/<PROVIDER>"`. The module is automatically downloaded during `terraform init`.

### Step 9: Call the local module

Add the local module call to the `main.tf` file:

```hcl
module "security_group" {
  source = "./modules/security-group"

  sg_name        = var.sg_name
  sg_description = var.sg_description
  vpc_id         = module.vpc.vpc_id
  ingress_port   = var.ingress_port
}
```

Note how:
- `source = "./modules/security-group"` uses a **relative path** for the local module
- `vpc_id = module.vpc.vpc_id` passes an output from the VPC module as input to the security group module — this is module **composition**

### Step 10: Declare root outputs

Create the `output.tf` file in the root directory:

```hcl
output "vpc_id" {
  description = "L'ID della VPC creata dal modulo registry"
  value       = module.vpc.vpc_id
}

output "public_subnet_ids" {
  description = "Gli ID delle subnet pubbliche"
  value       = module.vpc.public_subnets
}

output "security_group_id" {
  description = "L'ID del security group creato dal modulo locale"
  value       = module.security_group.security_group_id
}

output "security_group_name" {
  description = "Il nome del security group"
  value       = module.security_group.security_group_name
}
```

### Step 11: Initialize and apply

Initialize the project (this will download the module from the registry):

```bash
terraform init
```

In the output you will see that Terraform downloads the VPC module from the registry:

```
Downloading terraform-aws-modules/vpc/aws 6.x.x for vpc...
```

View the plan and apply:

```bash
terraform plan
terraform apply
```

### Step 12: Cleanup

At the end of the lab:

```bash
terraform destroy
```

## Next Lab

Continue to [Lab 07 - Workspaces](./lab-07-workspaces.md) to learn how to manage multiple environments (dev, prod) from a single configuration using Terraform workspaces.

