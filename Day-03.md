# Terraform – Day 03 - Variables, Outputs, Data Sources and Expressions

My Day 2 config works, but it is full of hardcoded values -- region, CIDR blocks, AMI IDs, instance types, tags. Change the region and everything breaks. Today I will make Terraform configs dynamic, reusable, and environment-aware.

This is the difference between a config that works once and a config you can use across projects.

---

### ✅ Task 1: Extract Variables

Take your Day 2 infrastructure config and refactor it:

1. Create a `variables.tf` file with input variables for:
   - `region` (string, default: your preferred region)
   - `vpc_cidr` (string, default: `"10.0.0.0/16"`)
   - `subnet_cidr` (string, default: `"10.0.1.0/24"`)
   - `instance_type` (string, default: `"t2.micro"`)
   - `project_name` (string, no default -- force the user to provide it)
   - `environment` (string, default: `"dev"`)
   - `allowed_ports` (list of numbers, default: `[22, 80, 443]`)
   - `extra_tags` (map of strings, default: `{}`)

```hcl
variable "region" {
    description = "EC2 Insatnce region"
    default = "us-east-2"
    type = string
}

variable "vpc_cidr" {
    description = "VPC cidr"
    default = "10.0.0.0/16"
    type = string
}

variable "subnet_cidr" {
    description = "subnet cidr"
    default = "10.0.1.0/24"
    type = string
}

variable "Insatnce_type" {
    description = "EC2 Insatnce type"
    default = "t2.micro"
    type = string
}

variable "project_name" {
    description = "Name of project (required)"
    type = string
}

variable "environment" {
    description = "working environment"
    default = "dev"
    type = string
}

variable "allowed_ports" {
    description = "working environment"
    default = [22,80,443]
    type = list(number)
}

variable "extra_tags" {
    description = "A map of additional tags to apply to resources"
    default = {}
    type = map(string)
  
}
```

2. Replace every hardcoded value in `main.tf` with `var.<name>` references



3. Run `terraform plan` -- it should prompt you for `project_name` since it has no default

**Document:** What are the five variable types in Terraform? (`string`, `number`, `bool`, `list`, `map`)

The five core variable types in Terraform are split into primitive (simple) types (string, number, bool) and collection types (list, map).

**Primitive Types**

* `string`: A sequence of Unicode characters representing text. It is always wrapped in double quotes (e.g., `"ami-xyz123"` or `"us-east-1"`).
* `number`: A numeric value that can represent both whole numbers (integers) and fractional values (decimals) (e.g., `3` or `0.0.0.0`).
* `bool`: A boolean value that can only be `true` or `false`. These are commonly used for conditional logic and feature toggles.


**Collection Types**

* `list`: A sequential, ordered collection of values. Items in a list must all share the same data type and are indexed starting at zero (e.g., `["subnet-1", "subnet-2"]`).
* `map`: A collection of key-value pairs where each unique string key maps to a specific value. All values in a single map must be of the same data type (e.g., `{ env = "production", tier = "frontend" }`).

---

### ✅ Task 2 : Install Terraform and Configure AWS

1. Create `terraform.tfvars`:
```hcl
project_name = "terraweek"
environment  = "dev"
instance_type = "t2.micro"
```

2. Create `prod.tfvars`:
```hcl
project_name = "terraweek"
environment  = "prod"
instance_type = "t3.small"
vpc_cidr     = "10.1.0.0/16"
subnet_cidr  = "10.1.1.0/24"
```

3. Apply with the default file:
```bash
terraform plan                              # Uses terraform.tfvars automatically
```

4. Apply with the prod file:
```bash
terraform plan -var-file="prod.tfvars"      # Uses prod.tfvars
```

5. Override with CLI:
```bash
terraform plan -var="instance_type=t2.nano"  # CLI overrides everything
```

6. Set an environment variable:
```bash
export TF_VAR_environment="staging"
terraform plan                              # env var overrides default but not tfvars
```

**Document:** Write the variable precedence order from lowest to highest priority.

---

### ✅ Task 3 : Your First Terraform Config -- Create an S3 Bucket

Create a project directory and write your first Terraform config:

```bash
mkdir terraform-basics && cd terraform-basics
```

Create a file called `main.tf` with:
1. A `terraform` block with `required_providers` specifying the `aws` provider
2. A `provider "aws"` block with your region
3. A `resource "aws_s3_bucket"` that creates a bucket with a globally unique name

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
    }
  }
}

provider "aws" {
    region = "us-east-2"
}

resource aws_s3_bucket my-bucket {
    bucket = "terra-bucket-hwb"
}

```

Run the Terraform lifecycle:
```bash
terraform init      # Download the AWS provider
terraform plan      # Preview what will be created
terraform apply     # Create the bucket (type 'yes' to confirm)
```

Go to the AWS S3 console and verify your bucket exists.

**Document:** What did `terraform init` download? What does the `.terraform/` directory contain ?

When you run `terraform init`, it primarily downloads provider plugins and external modules required by your configuration, storing them locally in a hidden `.terraform` directory. it contains local cache data required for your workspace, including downloaded provider plugins, remote module copies, current workspace settings, and backend migration metadata

---

### ✅ Task 4 : Add an EC2 Instance

In the same `main.tf`, add:
1. A `resource "aws_instance"` using AMI `ami-0f5ee92e2d63afc18` (Amazon Linux 2 in ap-south-1 -- use the correct AMI for your region)
2. Set instance type to `t2.micro`
3. Add a tag: `Name = "TerraWeek-Day1"`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
    }
  }
}

provider "aws" {
    region = "us-east-2"
}

resource aws_s3_bucket my-bucket {
    bucket = "terra-bucket-hwb"
}

resource "aws_instance" "my-instance" {
    ami = "ami-048f644e868baa0e8"
    instance_type = "t2.micro"  
    tags = {
        Name = "TerraWeek-Day1"
    }
}

```

Run:
```bash
terraform plan      # You should see 1 resource to add (bucket already exists)
terraform apply
```

Go to the AWS EC2 console and verify your instance is running with the correct name tag.

**Document:** How does Terraform know the S3 bucket already exists and only the EC2 instance needs to be created ?

Terraform knows the S3 bucket exists through its state file (terraform.tfstate), which tracks all managed resources and their real-world metadata.

When you run terraform plan or terraform apply, Terraform performs a three-way comparison:

* **State File Check**: It looks up terraform.tfstate to see what resources were created during previous runs.

* **Provider Refresh**: It queries the AWS API to confirm the S3 bucket is still present and matches the recorded state.

* **Configuration Drift Analysis**: It compares your updated main.tf against the refreshed state file:


---

### ✅ Task 5 : Understand the State File

Terraform tracks everything it creates in a state file. Time to inspect it.

1. Open `terraform.tfstate` in your editor -- read the JSON structure
2. Run these commands and document what each returns:
```bash
terraform show                          # Human-readable view of current state
terraform state list                    # List all resources Terraform manages
terraform state show aws_s3_bucket.<name>   # Detailed view of a specific resource
terraform state show aws_instance.<name>
```

3. Answer these questions in your notes:
   - What information does the state file store about each resource ?

     The Terraform state file stores a JSON-formatted mapping between your resource configurations and real-world infrastructure objects. For each resource, it records unique identifiers,         current attribute values, dependency relationships, and provider metadata to track real infrastructure status and plan future changes
     
   - Why should you never manually edit the state file ?

     because it is a highly structured JSON document that requires strict formatting, and even minor errors can corrupt your entire infrastructure deployment. Terraform relies on absolute         precision to track your real-world resources safely.
     
   - Why should the state file not be committed to Git ?

     because it poses severe security risks, breaks team collaboration, and causes merge conflicts. Version control systems are designed for code, whereas the state file functions as a            dynamic database.

---

### ✅ Task 6 : Modify, Plan, and Destroy

1. Change the EC2 instance tag from `"TerraWeek-Day1"` to `"TerraWeek-Modified"` in your `main.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
    }
  }
}

provider "aws" {
    region = "us-east-2"
}

resource aws_s3_bucket my-bucket {
    bucket = "terra-bucket-hwb"
}

resource "aws_instance" "my-instance" {
    ami = "ami-048f644e868baa0e8"
    instance_type = "t2.micro"  
    tags = {
        Name = "TerraWeek-Modified"
    }
}
```

2. Run `terraform plan` and read the output carefully:
   - What do the `~`, `+`, and `-` symbols mean ?
   - In the output of terraform plan, these symbols indicate the planned changes Terraform will make to your infrastructure:

     * `+` (Add): Terraform will create a brand new resource or attribute that does not currently exist in your target environment.

     * `-` (Destroy): Terraform will delete/destroy an existing resource or remove a specific attribute from state.

     * `~` (Update in-place): Terraform will modify an existing resource in-place without destroying and recreating it.

   - Is this an in-place update or a destroy-and-recreate ?
     * Changing the EC2 instance tag from "TerraWeek-Day1" to "TerraWeek-Modified" is an in-place update (~).

     * Updating tags in AWS EC2 only modifies the resource's metadata, so Terraform updates the instance without interrupting or recreating it.
     
4. Apply the change
5. Verify the tag changed in the AWS console
6. Finally, destroy everything:
```bash
terraform destroy
```
6. Verify in the AWS console -- both the S3 bucket and EC2 instance should be gone

---

## Note
- S3 bucket names must be globally unique -- use something like `terraweek-<yourname>-2026`
- AMI IDs are region-specific -- search "Amazon Linux 2 AMI" in your region's EC2 launch wizard
- `terraform fmt` auto-formats your `.tf` files -- run it before committing
- `terraform validate` checks for syntax errors without connecting to AWS
- The `.terraform/` directory contains downloaded provider plugins
- Add `*.tfstate`, `*.tfstate.backup`, and `.terraform/` to your `.gitignore`
