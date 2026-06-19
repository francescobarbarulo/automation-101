# Lab 05 - Remote State

## Learning Objectives

By the end of this lab you will be able to:

- Understand the limitations of local state and the advantages of remote state
- Configure an S3 backend to store Terraform state centrally
- Use native S3 locking (without DynamoDB) to prevent concurrent modifications
- Migrate a local state to a remote backend
- Verify that the state is actually stored in the S3 bucket

## Prerequisites

- **Terraform CLI** (version 1.10 or higher — required for native S3 locking)
- A **text editor** (VS Code with HashiCorp Terraform extension recommended)
- Completion of Labs 01-04
- **AWS account with configured credentials** — this lab requires creating an S3 bucket
- AWS CLI installed and configured (`aws configure`) or environment variables `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` set

## Instructions

### Step 1: Why remote state?

In previous labs, Terraform always saved the state in a local file (`terraform.tfstate`). This works for individual exercises, but in a real-world context it presents problems:

| Problem | Description |
|---------|-------------|
| Collaboration | Multiple people cannot work on the same infrastructure if the state is on a single computer |
| Security | The state contains sensitive data and should not reside on a laptop |
| Concurrency | Two people running `apply` simultaneously can corrupt the state |
| Backup | If the local file is lost, Terraform loses track of the infrastructure |

The solution is to store the state in a **remote backend**. In this lab we will use **S3** with:
- **Versioning** — to recover previous versions of the state in case of error
- **Encryption** — to protect sensitive data contained in the state
- **Native S3 locking** — to prevent concurrent modifications (available from Terraform 1.10+, without needing DynamoDB)

### Step 2: Create the S3 bucket for the backend

Before configuring the backend, we need to create the S3 bucket. Create a separate directory for this preparatory configuration:

```bash
mkdir -p lab-05-exercise/setup-backend
cd lab-05-exercise/setup-backend
```

For simplicity put everything in a single `main.tf` file:

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
  region = "eu-south-1"
}

# S3 bucket for state
resource "aws_s3_bucket" "state" {
  bucket = "terraform-state-dario-lab05"  # Replace with a globally unique name!

  tags = {
    Name    = "Terraform State Bucket"
    Purpose = "Remote state storage"
  }
  force_destroy = true # force empty bucket before deletion
}

# Enable versioning (to recover previous state versions)
resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "state" {
  bucket = aws_s3_bucket.state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

**Important:** replace `"NOME-BUCKET-UNICO"` with a globally unique name (e.g., `"terraform-state-yourname-lab05"`).

Apply this configuration:

```bash
terraform init
terraform apply
```

Note the name of the created bucket — you will use it in the next step.

### Step 3: Create the main configuration with local backend

Go back to the main lab directory:

```bash
cd ..
```

Create a `main.tf` file with a simple EC2 resource, **without** backend configuration (for now we use local state):

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
  region = "eu-south-1"
}

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

resource "aws_instance" "esempio" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  tags = {
    Name        = "lab05-remote-state"
    Environment = "lab"
  }
}

output "instance_id" {
  value = aws_instance.esempio.id
}
```

Initialize and apply:

```bash
terraform init
terraform apply
```

Verify that the state is local:

```bash
ls terraform.tfstate
```

The file exists in the current directory — this is the **local** state.

### Step 4: Configure the S3 backend with native locking

Now add the `backend` block inside the `terraform` block in your `main.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }

  backend "s3" {
    bucket       = "NOME-BUCKET-UNICO"       # The bucket created in Step 2
    key          = "lab05/terraform.tfstate"  # The path in the bucket
    region       = "eu-south-1"
    use_lockfile = true                      # Native S3 locking
    encrypt      = true
  }
}
```

Backend parameters:
- `bucket` — the name of the S3 bucket where the state will be saved
- `key` — the path (key) of the S3 object that will contain the state
- `region` — the bucket region
- `use_lockfile = true` — enables native S3 locking (Terraform creates a `.tflock` file next to the state)
- `encrypt` — enables encryption at rest

**Note:** in previous Terraform versions (< 1.10) a DynamoDB table was required for locking. From Terraform 1.10+ locking is native to S3 and DynamoDB is no longer necessary.

### Step 5: Migrate the state to the remote backend

Run `terraform init` again. Terraform will detect the backend change and ask if you want to migrate the state:

```bash
terraform init
```

You will see a message similar to:

```
Initializing the backend...
Do you want to copy existing state to the new backend?
  Enter a value: yes
```

Type `yes`. Terraform will copy the local state to the S3 bucket.

After migration, verify that the local `terraform.tfstate` file has been emptied (it contains only a reference to the remote backend).

### Step 6: Verify the remote state

Verify that the state was actually uploaded to the S3 bucket:

```bash
aws s3 ls s3://NOME-BUCKET-UNICO/lab05/
```

You should see the `terraform.tfstate` file.

You can also download and inspect the state (first 20 lines):

(Linux)
```bash
aws s3 cp s3://NOME-BUCKET-UNICO/lab05/terraform.tfstate - | head -20
```
(Windows)
```bash
aws s3 cp s3://NOME-BUCKET-UNICO/lab05/terraform.tfstate - | Select-Object -First 20
```

The content will be identical to what you had locally.

### Step 7: Verify native locking

To see locking in action, you can run a `terraform plan` and simultaneously (in another terminal) another `terraform plan`.

The second command will show an error:

```
Error: Error acquiring the state lock
```

Terraform creates a lock file (`lab05/terraform.tfstate.tflock`) in the S3 bucket during each operation. This file prevents concurrent access to the state, protecting the infrastructure from simultaneous modifications — all without needing DynamoDB.

### Step 8: Operate with remote state

Now the workflow is identical to before — `plan`, `apply`, `destroy` work exactly as with local state, but the state is centralized:

```bash
terraform plan
terraform show
terraform state list
```

The difference is that the state is read from and written to S3 on every operation, and the lock is acquired/released automatically.

### Step 9: Cleanup

First destroy the resources managed by the main configuration:

```bash
terraform destroy
```

Then, go back to the setup directory to destroy the S3 bucket:

```bash
cd setup-backend
terraform destroy
```

**Note:** the S3 bucket must be empty before it can be deleted. If the destroy fails, empty the bucket manually:

```bash
aws s3 rm s3://NOME-BUCKET-UNICO --recursive
```

Then retry `terraform destroy`.

## Next Lab

Continue to [Lab 06 - Modules](./lab-06-modules.md) to learn how to create reusable local modules and consume modules from the Terraform Registry.

