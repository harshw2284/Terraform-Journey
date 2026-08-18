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

```hcl
resource aws_vpc my-vpc {
    cidr_block = var.vpc_cidr
  tags = merge(
    {
    Name = "TerraWeek-VPC"
    Environment = var.environment
    Project = var.project_name
  },
  var.extra_tags
  )
}

resource aws_subnet my-subnet {
    cidr_block = var.subnet_cidr
    vpc_id     = aws_vpc.my-vpc.id
  tags = merge(
    {
    Name = "TerraWeek-Public-Subnet"
    Environment = var.environment
    Project = var.project_name
  },
  var.extra_tags
  )  
}

resource aws_internet_gateway my-gateway {
    vpc_id     = aws_vpc.my-vpc.id
}

resource aws_route_table my-table {
    vpc_id     = aws_vpc.my-vpc.id
    route {
      cidr_block = "0.0.0.0/0"
      gateway_id = aws_internet_gateway.my-gateway.id
    }
}

resource aws_route_table_association my-rt-as {
  subnet_id      = aws_subnet.my-subnet.id
  route_table_id = aws_route_table.my-table.id 
}

resource aws_security_group my-sg {
    name        = "TerraWeek-SG"
    description = "Allow SSH and HTTP inbound traffic"
    vpc_id      = aws_vpc.my-vpc.id 

    dynamic "ingress" {   
      for_each = var.allowed_ports
      content {
        from_port   = ingress.value
        protocol    = "tcp"
        to_port     = ingress.value
        cidr_blocks = ["0.0.0.0/0"]
    }
  }

    egress {
      from_port   = 0
      to_port     = 0
      protocol    = "-1"        # semantically equivalent to all ports
      cidr_blocks = ["0.0.0.0/0"]     
    }
}

resource "aws_instance" "my-instance" {
  ami = "ami-0e5497a77ef21b5ac"
  subnet_id = aws_subnet.my-subnet.id
  instance_type = var.Insatnce_type
  associate_public_ip_address = true
  lifecycle {
    create_before_destroy = true
  }
  tags = merge(
  {
    Name = "TerraWeek-Server" 
    Environment = var.environment
    Project = var.project_name
  },
  var.extra_tags 
 )
}

resource "aws_s3_bucket" "my-bucket" {
    bucket = "terra-bucket-hwb"
    depends_on = [ aws_instance.my-instance ]
}
```

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

### ✅ Task 2 : Variable Files and Precedence

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

In Terraform, variable precedence is evaluated in the following order, from lowest to highest priority (values defined further down the list override values defined above them):

1. Environment variables (`TF_VAR_variable_name`)

2. `terraform.tfvars` file

3. `terraform.tfvars.json` file

4. `*.auto.tfvars` or `*.auto.tfvars.json` files (processed in alphabetical order by filename)

5. `-var` and `-var-file` options specified on the command line (in the exact order they are passed)

---

### ✅ Task 3 : Add Outputs

Create an `outputs.tf` file with outputs for:

1. `vpc_id` -- the VPC ID
2. `subnet_id` -- the public subnet ID
3. `instance_id` -- the EC2 instance ID
4. `instance_public_ip` -- the public IP of the EC2 instance
5. `instance_public_dns` -- the public DNS name
6. `security_group_id` -- the security group ID

```hcl
output "vpc_id" {
  description = "The ID of the VPC"  
  value = aws_vpc.my-vpc.id
}

output "subnet_id" {
  description = "The ID of the public subnet"
  value = aws_subnet.my-subnet.id
}

output "instance_id" {
  description = "The ID of the EC2 instance"
  value = aws_instance.my-instance.id
}

output "instance_public_ip" {
  description = "The public IP address assigned to the EC2 instance"
  value = aws_instance.my-instance.public_ip
}

output "instance_public_dns" {
  description = "The public DNS name assigned to the EC2 instance"
  value = aws_instance.my-instance.public_dns
}

output "security_group_id" {
  description = "The ID of the security group"  
  value = aws_security_group.my-sg.id
}
```

Apply your config and verify the outputs are printed at the end:
```bash
terraform apply

# After apply, you can also run:
terraform output                          # Show all outputs
terraform output instance_public_ip       # Show a specific output
terraform output -json                    # JSON format for scripting
```

**Verify:** Does `terraform output instance_public_ip` return the correct IP ?

<img width="659" height="181" alt="Screenshot 2026-08-18 230905" src="https://github.com/user-attachments/assets/449f4904-7520-489e-8d8d-5f9bb8ae2aa3"/>

Yes !

---

### ✅ Task 4 : Use Data Sources

Stop hardcoding the AMI ID. Use a data source to fetch it dynamically.

1. Add a `data "aws_ami"` block that:
   - Filters for Amazon Linux 2 images
   - Filters for `hvm` virtualization and `gp2` root device
   - Uses `owners = ["amazon"]`
   - Sets `most_recent = true`

```hcl
data "aws_ami" "amazon_linux" {
  owners = ["amazon"]
  most_recent = true                           # Terraform will search AWS for AMIs matching that name

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]    # The values here is a search pattern, not the AMI ID.
  }
  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
}
```

2. Replace the hardcoded AMI in your `aws_instance` with `data.aws_ami.amazon_linux.id`

3. Add a `data "aws_availability_zones"` block to fetch available AZs in your region

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}
```

4. Use the first AZ in your subnet: `data.aws_availability_zones.available.names[0]`

Apply and verify -- your config now works in any region without changing the AMI.

**Document:** What is the difference between a `resource` and a `data` source?

A resource and a data source serve opposite roles regarding infrastructure lifecycle management in Terraform:

* **Resource** (`resource`): Creates, updates, or deletes infrastructure components (e.g., an EC2 instance, VPC, or S3 bucket). Terraform directly manages the lifecycle and state of these objects.

**Data Source** (`data`): Reads or fetches read-only information about existing infrastructure created outside the current Terraform workspace or configuration (e.g., querying the latest official AMI ID or existing VPCs). Terraform does not create, modify, or destroy data sources.

| Feature | Resource (`resource`) | Data Source (`data`) |
| :--- | :--- | :--- |
| **Primary Purpose** | Provision and manage infrastructure lifecycle | Query and read existing infrastructure data |
| **Action** | Creates, modifies, or destroys real-world objects | Read-only lookup |
| **Management** | Full lifecycle management via `terraform destroy` / `apply` | Never modified or deleted by Terraform |
| **Example Use Case** | Deploying a new `aws_instance` | Fetching the latest Amazon Linux 2 AMI ID with `aws_ami` |

---

### ✅ Task 5 : Use Locals for Dynamic Values

1. Add a `locals` block:
```hcl
locals {
  name_prefix = "${var.project_name}-${var.environment}"
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

2. Replace all Name tags with `local.name_prefix`:
   - VPC: `"${local.name_prefix}-vpc"`
   - Subnet: `"${local.name_prefix}-subnet"`
   - Instance: `"${local.name_prefix}-server"`

3. Merge common tags with resource-specific tags:
```hcl
tags = merge(local.common_tags, {
  Name = "${local.name_prefix}-server"
})
```

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_ami" "amazon_linux" {
  owners = ["amazon"]
  most_recent = true                           # Terraform will search AWS for AMIs matching that name

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]    # The values here is a search pattern, not the AMI ID.
  }
  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
}

locals {
  name_prefix = "${var.project_name}-${var.environment}"
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource aws_vpc my-vpc {
    cidr_block = var.vpc_cidr
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  },
  var.extra_tags
  )
}

resource aws_subnet my-subnet {
    cidr_block = var.subnet_cidr
    vpc_id     = aws_vpc.my-vpc.id
    availability_zone = data.aws_availability_zones.available.names[0]
  tags = merge(local.common_tags,{
    Name = "${local.name_prefix}-subnet"
  },
  var.extra_tags
  )  
}

resource aws_internet_gateway my-gateway {
    vpc_id     = aws_vpc.my-vpc.id
}

resource aws_route_table my-table {
    vpc_id       = aws_vpc.my-vpc.id
    route {
      cidr_block = "0.0.0.0/0"
      gateway_id = aws_internet_gateway.my-gateway.id
    }
}

resource aws_route_table_association my-rt-as {
  subnet_id      = aws_subnet.my-subnet.id
  route_table_id = aws_route_table.my-table.id 
}

resource aws_security_group my-sg {
    name        = "TerraWeek-SG"
    description = "Allow SSH and HTTP inbound traffic"
    vpc_id      = aws_vpc.my-vpc.id 
    dynamic "ingress" {

      for_each = var.allowed_ports
      content {
        from_port   = ingress.value
        protocol    = "tcp"
        to_port     = ingress.value
        cidr_blocks = ["0.0.0.0/0"]
    }
  }
    egress {
      from_port   = 0
      to_port     = 0
      protocol    = "-1"           # semantically equivalent to all ports
      cidr_blocks = ["0.0.0.0/0"]     
    }
}

resource "aws_instance" "my-instance" {
  ami = data.aws_ami.amazon_linux.id
  subnet_id = aws_subnet.my-subnet.id
  instance_type = var.instance_type
  associate_public_ip_address = true
  lifecycle {
    create_before_destroy = true
  }
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-server"
  },
  var.extra_tags 
 )
}

resource "aws_s3_bucket" "my-bucket" {
    bucket = "terra-bucket-hwb"
    depends_on = [ aws_instance.my-instance ]
}
```

Apply and check the tags in the AWS console -- every resource should have consistent tagging.

<img width="1572" height="690" alt="image" src="https://github.com/user-attachments/assets/ee445881-f2af-4fba-acb3-25cbcda5e3ac" />

---

### ✅ Task 6 : Built-in Functions and Conditional Expressions

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
- `terraform.tfvars` is loaded automatically. Any other `.tfvars` file needs `-var-file`
- Variable precedence (low to high): default -> `terraform.tfvars` -> `*.auto.tfvars` -> `-var-file` -> `-var` flag -> `TF_VAR_*` env vars
- `terraform console` is an interactive REPL for testing expressions and functions
- Data sources are read-only -- they fetch information, they don't create resources
- `merge()` combines two maps -- great for tags
- `terraform output -json` is useful when piping output into other scripts
