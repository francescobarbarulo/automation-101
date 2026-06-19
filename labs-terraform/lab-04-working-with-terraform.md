# Lab 04 - Working with Terraform

## Learning Objectives

By the end of this lab you will be able to:

- Use the `terraform validate` and `terraform fmt` commands to verify and format configuration
- Use `terraform show` and `terraform output` to query resource state
- Understand the difference between mutable and immutable infrastructure
- Configure lifecycle rules (`create_before_destroy`, `prevent_destroy`, `ignore_changes`)
- Use data sources to reference existing resources
- Use the `count` and `for_each` meta-arguments to create multiple resource instances

## Prerequisites

- **Terraform CLI** (version 1.0 or higher) installed
- A **text editor** (VS Code with HashiCorp Terraform extension recommended)
- Completion of Labs 01, 02, and 03 (knowledge of HCL, variables, outputs, and state)
- **AWS account with configured credentials** — this lab uses the AWS provider to create real cloud resources
- AWS CLI installed and configured (`aws configure`) or environment variables `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` set

## Instructions

### Step 1: Prepare the project directory

Create a new directory for the lab and the initial configuration files:

```bash
mkdir lab-04-exercise
cd lab-04-exercise
```

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

Create the `variables.tf` file:

```hcl
variable "aws_region" {
  description = "La regione AWS in cui creare le risorse"
  type        = string
  default     = "eu-south-1"
}

variable "nome_progetto" {
  description = "Nome del progetto per il tagging"
  type        = string
  default     = "lab04-terraform"
}
```

Create a `main.tf` file with a security group:

```hcl
resource "aws_security_group" "web" {
  name        = "${var.nome_progetto}-web-sg"
  description = "Security group per le istanze web"

  ingress {
    descriptions = "HTTP"
    from_port   = 80
    to_port     = 80
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
    Name = "${var.nome_progetto}-web-sg"
  }
}
```

Initialize the project:

```bash
terraform init
```

### Step 2: Use terraform validate and terraform fmt

The `terraform validate` command checks the syntactic and semantic correctness of the configuration **without** accessing remote services:

```bash
terraform validate
```

It should report an error:

```bash
Error: Unsupported argument
│ 
│   on main.tf line 8, in resource "aws_security_group" "web":
│    8:     to_ports     = 80
```

Go to main.tf and remove the 's' from line 8 (the correct argument is to_port) and use:

```bash
terraform validate
```

If the configuration is valid, you will see:

```
Success! The configuration is valid.
```

The `terraform fmt` command automatically formats `.tf` files according to standard conventions:

```bash
terraform fmt
```

### Step 3: Use terraform show and terraform output

After applying a configuration, you can inspect the current state with:

```bash
terraform show
```

This shows all resources in the current state with their attributes.

To view only the declared outputs:

```bash
terraform output
```

To get a single output value (useful in scripts):

```bash
terraform output -raw ami_id
```

### Step 4: Mutable vs immutable infrastructure

In Terraform it is essential to understand two approaches:

- **Mutable infrastructure**: resources are modified in-place (direct update). Terraform updates attributes without recreating the resource.
- **Immutable infrastructure**: resources are destroyed and recreated when key attributes change. Terraform replaces the entire resource.

Some AWS resource attributes are "force new" — modifying them causes the resource to be recreated (immutable approach). Lifecycle rules allow you to control this behavior.

### Step 5: Configure lifecycle rules

First, destroy the security group created earlier:

```bash
terraform destroy
```

Create a `main.tf` file with a security group that uses `create_before_destroy`:

```hcl
resource "aws_security_group" "web" {
  name        = "${var.nome_progetto}-web-sg"
  description = "Security group per le istanze web"

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
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
    Name = "${var.nome_progetto}-web-sg"
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

The three main lifecycle rules are:

| Rule | Effect |
|------|--------|
| `create_before_destroy` | Creates the new resource before deleting the old one (reduces downtime) |
| `prevent_destroy` | Prevents accidental destruction of the resource (Terraform will emit an error) |
| `ignore_changes` | Ignores changes to specific attributes (useful for values modified externally) |

### Step 6: Use Data Sources

Data sources allow you to **read** information about existing resources without managing them. Add to your `main.tf`:

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

This data source searches for the most recent Amazon Linux AMI. You can reference the result as `data.aws_ami.amazon_linux.id` in resources.

Add an output in the `output.tf` file to display the information read from the data source:

```hcl
output "ami_id" {
  description = "ID dell'AMI Amazon Linux più recente"
  value       = data.aws_ami.amazon_linux.id
}

output "ami_name" {
  description = "Nome dell'AMI Amazon Linux trovata"
  value       = data.aws_ami.amazon_linux.name
}

output "ami_description" {
  description = "Descrizione dell'AMI"
  value       = data.aws_ami.amazon_linux.description
}
```

After running `terraform apply`, you can query these values:

```bash
terraform output ami_id
terraform output ami_name
```

This demonstrates how data sources allow you to **extract information** from existing infrastructure and make it available through outputs.

The key difference compared to a `resource`:
- `resource` — Terraform **creates and manages** the resource
- `data` — Terraform **reads** information about an existing resource

### Step 7: Use count to create multiple instances

The `count` meta-argument allows you to create a variable number of instances of the same resource. Add the variable to `variables.tf`:

```hcl
variable "numero_istanze" {
  description = "Numero di istanze da creare"
  type        = number
  default     = 2
}
```

Then add the EC2 resource (VM on AWS) inside `main.tf`:

```hcl
resource "aws_instance" "web" {
  count = var.numero_istanze

  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.web.id]

  tags = {
    Name = "${var.nome_progetto}-web-${count.index + 1}"
  }
}
```

With `count`, each instance is accessible by index: `aws_instance.web[0]`, `aws_instance.web[1]`, etc.
As you can see from the execution plan after running `terraform apply`, the AMI is retrieved from the data source set up in step 6, both instances depend on the security group, and the number of instances created is 2, as specified in the variable value.
You can access the AWS console to verify that the instances were created correctly and are running.

### Step 8: Use for_each to create resources from a map

The `for_each` meta-argument is more flexible than `count` — it allows iterating over a map or a set.
Define the environments map variable in the `variables.tf` file:

```hcl
variable "ambienti" {
  description = "Mappa degli ambienti"
  type        = map(string)
  default = {
    dev     = "development"
    staging = "staging"
    prod    = "production"
  }
}
```

And define the S3 resource in `main.tf`:

```hcl
resource "aws_s3_bucket" "ambienti" {
  for_each = var.ambienti

  bucket = "${var.nome_progetto}-${each.key}-bucket"

  tags = {
    Name        = "${var.nome_progetto}-${each.key}"
    Environment = each.value
  }
}
```

Running `terraform apply` will create 3 resources, one for each variable defined in the map.
With `for_each`, each instance is identified by key: `aws_s3_bucket.ambienti["dev"]`, `aws_s3_bucket.ambienti["prod"]`, etc.

**Advantages of `for_each` over `count`:**
- Removing an element from the map only deletes that specific resource
- With `count`, removing an element in the middle causes subsequent resources to be recreated

### Step 9: Apply and verify

Apply the complete configuration:

```bash
terraform apply
```

After applying, verify the outputs:

```bash
terraform output
terraform show
```

### Step 10: Cleanup

At the end of the exercise, remove all created resources:

```bash
terraform destroy
```

**Note:** If you have resources with `prevent_destroy = true`, you must first comment out or remove that rule from the code before you can run destroy.

## Next Lab

Continue to [Lab 05 - Remote State](./lab-05-remote-state.md) to learn how to store Terraform state remotely in S3 with locking, enabling team collaboration and improved security.

