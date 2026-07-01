---
tags: [terraform, iac, devops, cloud]
---
# Terraform

Terraform is a declarative **Infrastructure as Code (IaC)** tool. You describe the desired end state of your infrastructure in configuration files, and Terraform figures out the plan to get there.

---

## CORE CONCEPTS

```
Configuration (.tf files)
└── Provider (aws, azurerm, google, ...)
    └── Resource (e.g. aws_instance, azurerm_storage_account)

State file (terraform.tfstate)
└── Terraform's record of what it believes actually exists

Plan
└── Diff between desired config and current state
    └── Apply — executes the plan against the real infrastructure
```

- **Provider** — plugin that talks to an API (AWS, Azure, GCP, etc.)
- **Resource** — a single infrastructure object managed by Terraform (VM, bucket, network...)
- **Data source** — read-only lookup of existing infrastructure not managed by this config
- **Module** — reusable, packaged group of resources
- **State** — the source of truth Terraform uses to map config to real-world objects

---

## CLI SETUP

```bash
# Verify installation
terraform -version

# Initialize a working directory (downloads providers, sets up backend)
terraform init

# Format files to canonical style
terraform fmt

# Validate configuration syntax
terraform validate
```

---

## CORE WORKFLOW

```bash
# Show what changes would be made
terraform plan

# Save a plan to a file, then apply exactly that plan
terraform plan -out=tfplan
terraform apply tfplan

# Apply changes (prompts for confirmation)
terraform apply

# Apply without confirmation prompt
terraform apply -auto-approve

# Destroy all managed infrastructure
terraform destroy
```

---

## CONFIGURATION BASICS

```hcl
# main.tf

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = "ami-0123456789abcdef0"
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }
}
```

### VARIABLES

```hcl
# variables.tf
variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "environment" {
  type = string
}
```

```hcl
# usage
resource "aws_instance" "web" {
  instance_type = var.instance_type
}
```

```bash
# Pass values at runtime
terraform apply -var="environment=prod"

# Or via a file
terraform apply -var-file="prod.tfvars"
```

### OUTPUTS

```hcl
# outputs.tf
output "instance_ip" {
  value = aws_instance.web.public_ip
}
```

### DATA SOURCES

```hcl
data "aws_ami" "latest_ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-*"]
  }
}
```

---

## STATE

```bash
# List resources tracked in state
terraform state list

# Show details of a resource in state
terraform state show aws_instance.web

# Move/rename a resource in state (without destroying it)
terraform state mv aws_instance.web aws_instance.app

# Remove a resource from state (without destroying it)
terraform state rm aws_instance.web

# Import an existing resource into state
terraform import aws_instance.web i-0123456789abcdef0
```

### REMOTE STATE

Local state (`terraform.tfstate`) doesn't scale beyond one person. Use a remote backend for locking and shared access.

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tfstate-bucket"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-state-lock"   # state locking
    encrypt        = true
  }
}
```

---

## MODULES

Package resources into a reusable unit.

```hcl
# modules/vpc/main.tf
variable "cidr_block" {
  type = string
}

resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
}

output "vpc_id" {
  value = aws_vpc.this.id
}
```

```hcl
# root main.tf
module "network" {
  source     = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id = module.network.vpc_id
  cidr_block = "10.0.1.0/24"
}
```

---

## MULTIPLE ENVIRONMENTS

### WORKSPACES

Lightweight isolation of state within the same config.

```bash
terraform workspace new staging
terraform workspace select staging
terraform workspace list
```

### DIRECTORY-PER-ENVIRONMENT (COMMON PATTERN)

```
environments/
├── dev/
│   ├── main.tf
│   └── terraform.tfvars
├── staging/
│   ├── main.tf
│   └── terraform.tfvars
└── prod/
    ├── main.tf
    └── terraform.tfvars
```

Preferred over workspaces when environments diverge significantly, since each has its own state and can be reviewed independently.

---

## META-ARGUMENTS

```hcl
# Repeat a resource N times
resource "aws_instance" "web" {
  count         = 3
  ami           = data.aws_ami.latest_ubuntu.id
  instance_type = "t3.micro"
}

# Repeat a resource keyed by map/set
resource "aws_instance" "web" {
  for_each      = toset(["a", "b", "c"])
  ami           = data.aws_ami.latest_ubuntu.id
  instance_type = "t3.micro"

  tags = {
    Name = "web-${each.key}"
  }
}

# Force replacement instead of in-place update
resource "aws_instance" "web" {
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = false
    ignore_changes        = [tags]
  }
}
```

---

## PATTERNS

### DEPENDENCY ORDERING

Terraform builds a dependency graph automatically from references (`aws_subnet.public.vpc_id` implies subnet depends on VPC). Use `depends_on` only when the dependency isn't visible through a reference.

```hcl
resource "aws_instance" "web" {
  depends_on = [aws_iam_role_policy.web]
}
```

### SECRETS

Never hardcode secrets in `.tf` files. Pull them from a secrets manager at apply time.

```hcl
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db/password"
}
```

### CI/CD

```bash
# Non-interactive plan, fails if changes are needed and none were expected
terraform plan -detailed-exitcode

# Lock provider versions for reproducible builds
terraform providers lock -platform=linux_amd64
```

---

## GITIGNORE

```
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars
crash.log
```

See also [[Docker]] and [[Azure Cloud]].
