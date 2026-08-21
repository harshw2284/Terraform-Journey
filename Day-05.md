# Terraform – Day 05 - Terraform Modules: Build Reusable Infrastructure

I have been writing everything in one big `main.tf` file. That works for learning, but in real teams you manage dozens of environments with hundreds of resources. Copy-pasting configs across projects is a recipe for disaster.

Today I learned Terraform modules -- the way to package, reuse, and share infrastructure code. Think of modules as functions in programming. Write once, call many times.

---

### ✅ Task 1: Understand Module Structure

A Terraform module is just a directory with `.tf` files. Create this structure:

```
terraform-modules/
  main.tf                    # Root module -- calls child modules
  variables.tf               # Root variables
  outputs.tf                 # Root outputs
  providers.tf               # Provider config
  modules/
    ec2-instance/
      main.tf                # EC2 resource definition
      variables.tf           # Module inputs
      outputs.tf             # Module outputs
    security-group/
      main.tf                # Security group resource definition
      variables.tf           # Module inputs
      outputs.tf             # Module outputs
```

Create all the directories and empty files. This is the standard layout every Terraform project follows.

**Document:** What is the difference between a "root module" and a "child module"?

The primary difference between a root module and a child module is where Terraform is executed and how the modules are invoked.The root module is the main working directory where you run Terraform CLI commands (like `terraform apply`), serving as the entry point. A child module is any external, reusable package of configuration files that is called from within another module using a `module` block.

---

### ✅ Task 2 : Build a Custom EC2 Module

Create `modules/ec2-instance/`:

1. **`variables.tf`** -- define inputs:
   - `ami_id` (string)
   - `instance_type` (string, default: `"t2.micro"`)
   - `subnet_id` (string)
   - `security_group_ids` (list of strings)
   - `instance_name` (string)
   - `tags` (map of strings, default: `{}`)

```hcl
variable "ami_id" {
    description = "ami_id for ec2 instance"
    type = string
}

variable "instance_type" {
    description = "ec2 instance type"
    default = "t2.micro"
    type = string
}

variable "subnet_id" {
    description = "subnet id"
    type = string
}

variable "security_group_ids" {
    description = "security group ids"
    type = list(string)  
}

variable "instance_name" {
    description = "name of ec2 instance"
    type = string
}

variable "tags" {
    description = "value"
    default = {}
    type = map(string)
}
```

2. **`main.tf`** -- define the resource:
   - `aws_instance` using all the variables
   - Merge the Name tag with additional tags

```hcl
resource "aws_instance" "my-instance" {
  ami           = var.ami_id
  subnet_id     = var.subnet_id
  instance_type = var.instance_type
  vpc_security_group_ids = var.security_group_ids
  tags = merge(
    {
    Name = var.instance_name
    },
    var.tags
  )
}
```

3. **`outputs.tf`** -- expose:
   - `instance_id`
   - `public_ip`
   - `private_ip`

```hcl
output "instance_id" {
  value = aws_instance.my-instance.id
}

output "public_ip" {
  value = aws_instance.my-instance.public_ip
}

output "private_ip" {
  value = aws_instance.my-instance.private_ip
}
```

Do NOT apply yet -- just write the module.

---

### ✅ Task 3 : Build a Custom Security Group Module

Create `modules/security-group/`:

1. **`variables.tf`** -- define inputs:
   - `vpc_id` (string)
   - `sg_name` (string)
   - `ingress_ports` (list of numbers, default: `[22, 80]`)
   - `tags` (map of strings, default: `{}`)

```hcl
variable "vpc_id" {
    description = "VPC id"
    type = string
}

variable "sg_name" {
  description = "name of security group"
  type = string
}

variable "ingress_ports" {
  description = "Incoming ports"
  type = list(number)
}

variable "tags" {
    type = map(string)
}
```

2. **`main.tf`** -- define the resource:
   - `aws_security_group` in the given VPC
   - Use `dynamic "ingress"` block to create rules from the `ingress_ports` list
   - Allow all egress

```hcl
resource "aws_security_group" "my_sg" {
  name        = var.sg_name
  description = "Security group created by module"
  vpc_id      = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    ="-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    {
      Name = var.sg_name
    },
    var.tags
  )
}  
  
```

3. **`outputs.tf`** -- expose:
   - `sg_id`

```hcl
output "sg_id" {
  description = "The ID of the created security group"
  value = aws_security_group.my_sg.id 
}
```

This is your first time using a `dynamic` block -- it loops over a list to generate repeated nested blocks.

---

### ✅ Task 4 : Call Your Modules from Root

In the root `main.tf`, wire everything together:

1. Create a VPC and subnet directly (or reuse your Day 2 config)

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

locals {
  common_tags = {
    Environment = "dev"
    Project     = "terraweek"
    ManagedBy   = "terraform"
  }
}

resource aws_vpc main {
    cidr_block  = "10.0.0.0/16"
    tags = merge(
    {
    Name = "main-vpc"
    },
    local.common_tags
    )
}

resource aws_subnet public {
    cidr_block = "10.0.1.0/24"
    vpc_id     = aws_vpc.main.id
    map_public_ip_on_launch = true
  tags = merge({
    Name = "my-subnet"
  },
  local.common_tags
  )
}

module "web_sg" {
  source        = "./modules/security-group"
  vpc_id        = aws_vpc.main.id
  sg_name       = "terraweek-web-sg"
  ingress_ports = [22, 80, 443]
  tags          = local.common_tags
  
}

module "web_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = aws_subnet.public.id
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-web"
  tags               = local.common_tags
}

module "api_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = aws_subnet.public.id
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-api"
  tags               = local.common_tags
}
```

2. Call the security group module:
```hcl
module "web_sg" {
  source        = "./modules/security-group"
  vpc_id        = aws_vpc.main.id
  sg_name       = "terraweek-web-sg"
  ingress_ports = [22, 80, 443]
  tags          = local.common_tags
}
```

3. Call the EC2 module -- deploy **two instances** with different names using the same module:
```hcl
module "web_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = aws_subnet.public.id
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-web"
  tags               = local.common_tags
}

module "api_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = aws_subnet.public.id
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-api"
  tags               = local.common_tags
}
```

4. Add root outputs that reference module outputs:
```hcl
output "web_server_ip" {
  value = module.web_server.public_ip
}

output "api_server_ip" {
  value = module.api_server.public_ip
}
```

5. Apply:
```bash
terraform init    # Downloads/links the local modules
terraform plan    # Should show all resources from both module calls
terraform apply
```

**Verify:** Two EC2 instances running, same security group, different names. Check the AWS console.

---

### ✅ Task 5 : State Surgery -- mv and rm

Sometimes you need to rename a resource or remove it from state without destroying it in AWS.

1. **Rename a resource in state:**
```bash
terraform state list                              # Note the current resource names
terraform state mv aws_s3_bucket.imported aws_s3_bucket.logs_bucket
```
Update your `.tf` file to match the new name. Run `terraform plan` -- it should show no changes.

```hcl
resource "aws_s3_bucket" "logs_bucket" {
    bucket = "terraweek-import-test-harsh"
}
```

2. **Remove a resource from state (without destroying it):**
```bash
terraform state rm aws_s3_bucket.logs_bucket
```
Run `terraform plan` -- Terraform no longer knows about the bucket, but it still exists in AWS.

3. **Re-import it** to bring it back:
```bash
terraform import aws_s3_bucket.logs_bucket terraweek-import-test-<yourname>
```

**Document:** When would you use `state mv` in a real project? When would you use `state rm`?

In a real-world production project, you use `terraform state mv` when you want to rename or reorganize code without destroying real infrastructure, while you use `terraform state rm` when you want Terraform to stop managing a resource entirely without deleting it.

---

### ✅ Task 6 : Built-in Functions and Conditional Expressions

1. Pin your registry module version explicitly:
   - `version = "5.1.0"` -- exact version
   - `version = "~> 5.0"` -- any 5.x version
   - `version = ">= 5.0, < 6.0"` -- range

2. Run `terraform init -upgrade` to check for newer versions

3. Check the state to see how modules appear:
```bash
terraform state list
```
Notice the `module.vpc.`, `module.web_server.`, `module.web_sg.` prefixes.

4. Destroy everything:
```bash
terraform destroy
```

**Document:** Write down five module best practices:
- Always pin versions for registry modules
- Keep modules focused -- one concern per module
- Use variables for everything, hardcode nothing
- Always define outputs so callers can reference resources
- Add a README.md to every custom module

---

## Note
- `terraform init` must be re-run after adding a new module source
- Module outputs are accessed as `module.<name>.<output>`
- `dynamic` blocks use `content {}` inside to define the repeated block
- Registry modules document all inputs and outputs on registry.terraform.io
- Local modules use `source = "./modules/<name>"`, registry modules use `source = "<org>/<name>/<provider>"`
- `terraform get` downloads modules without full init


---

