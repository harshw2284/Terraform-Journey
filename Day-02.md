# Terraform – Day 01 - Introduction to Terraform and Your First AWS Infrastructure

Yesterday I created standalone resources. But real infrastructure is connected -- a server lives inside a subnet, a subnet lives inside a VPC, a security group controls what traffic gets in. Today I will build a complete networking stack on AWS and learn how Terraform figures out what to create first.

Understanding dependencies is what separates a Terraform beginner from someone who can build production infrastructure.

---

### ✅ Task 1: Explore the AWS Provider

1. Create a new project directory: `terraform-aws-infra`
2. Write a `providers.tf` file:
   - Define the `terraform` block with `required_providers` pinning the AWS provider to version `~> 5.0`
   - Define the `provider "aws"` block with your region

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
    region = "us-east-2"
}
```

3. Run `terraform init` and check the output -- what version was installed ?

<img width="766" height="130" alt="image" src="https://github.com/user-attachments/assets/8976ba95-74a0-47da-98aa-9e13ba286644" />

hashicorp/aws v5.100.0 version was installed.

4. Read the provider lock file `.terraform.lock.hcl` -- what does it do?

<img width="982" height="650" alt="image" src="https://github.com/user-attachments/assets/ea157a6a-45f8-4410-9098-126ecf3c4693" />

The .terraform.lock.hcl file is a dependency lock file created automatically by Terraform. It records the exact versions and cryptographic checksums of provider plugins used in your configuration. This ensures that every team member, server, and CI/CD pipeline installs identical provider binaries, preventing unexpected updates or breakages

**Document:** What does `~> 5.0` mean? How is it different from `>= 5.0` and `= 5.0.0`?

In Terraform provider version constraints, ~>, >=, and = control how Terraform selects provider updates during terraform init:

* `~> 5.0` (Pessimistic Constraint Operator): Allows updates only to the rightmost version component.

  * `~> 5.0` (2 components specified) allows any version from 5.0.0 up to, but not including, 6.0.0 (e.g., 5.1.0, 5.99.0). It locks the major version to prevent breaking changes while automatically pulling in minor updates and bug fixes.

  * `Note:` If written as ~> 5.0.0 (3 components specified), it would allow updates to patch releases only (e.g., 5.0.1, 5.0.99), stopping before 5.1.0.

* `>= 5.0` (Greater Than or Equal To): Allows version 5.0.0 or any higher version, including future major version releases like 6.0.0 or 7.0.0. This risks breaking your infrastructure code if a future major release removes features or changes syntax.

* `= 5.0.0` (Exact Pinning): Strictly locks the provider to version 5.0.0. Terraform will only install this exact version and will not accept any patch fixes or minor updates.

| Constraint | Lowest Allowed | Example Highest Allowed | Allows Breaking Changes? |
| :--- | :--- | :--- | :--- |
| `~> 5.0` | `5.0.0` | `5.x.x` (below `6.0.0`) | **No** (protects major version) |
| `>= 5.0` | `5.0.0` | Any version (`6.0`, `7.0`, etc.) | **Yes** (unrestricted upper bound) |
| `= 5.0.0` | `5.0.0` | `5.0.0` only | **No** (blocks all updates) |

---

### ✅ Task 2 : Build a VPC from Scratch

Create a `main.tf` and define these resources one by one:

1. `aws_vpc` -- CIDR block `10.0.0.0/16`, tag it `"TerraWeek-VPC"`
2. `aws_subnet` -- CIDR block `10.0.1.0/24`, reference the VPC ID from step 1, enable public IP on launch, tag it `"TerraWeek-Public-Subnet"`
3. `aws_internet_gateway` -- attach it to the VPC
4. `aws_route_table` -- create it in the VPC, add a route for `0.0.0.0/0` pointing to the internet gateway
5. `aws_route_table_association` -- associate the route table with the subnet

```hcl
resource aws_vpc my-vpc {
    cidr_block = "10.0.0.0/16"
  tags = {
    Name = "TerraWeek-VPC"
  }
}

resource aws_subnet my-subnet {
    cidr_block = "10.0.1.0/24"
    vpc_id     = aws_vpc.my-vpc.id
  tags = {
    Name = "TerraWeek-Public-Subnet"
  }  
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
```

Run `terraform plan` -- you should see 5 resources to create.

**Verify:** Apply and check the AWS VPC console. Can you see all five resources connected ?

Yes !

---

### ✅ Task 3 : Understand Implicit Dependencies

Look at your `main.tf` carefully:

1. The subnet references `aws_vpc.main.id` -- this is an implicit dependency
2. The internet gateway references the VPC ID -- another implicit dependency
3. The route table association references both the route table and the subnet

Answer these questions:
1. How does Terraform know to create the VPC before the subnet ?

Terraform knows to create the VPC before the subnet through implicit dependencies, which it determines by parsing your configuration code to build a Directed Acyclic Graph (DAG) before applying changes.

* Attribute Reference: In your `aws_subnet` block, you pass the VPC's ID by referencing an attribute from the VPC resource (for example, vpc_id = `aws_vpc.main.id`).
* Dependency Graph: When Terraform parses this expression, it automatically infers that `aws_subnet` relies on `aws_vpc.main`. It places `aws_vpc.main` upstream in the execution graph.
* Execution Ordering: During terraform apply, Terraform uses this graph to determine the correct sequence—creating the parent resource (`aws_vpc`) first so its exported attributes (such as the generated VPC ID) become available to populate the dependent resource (aws_subnet).

2. What would happen if you tried to create the subnet before the VPC existed ?

If you attempted to create the subnet before the VPC existed, the operation would fail at one of two stages:

* Static Validation / Interpolation Error: Because the subnet resource configuration expects a reference to an existing VPC's ID (such as `vpc_id = aws_vpc.main.id`), Terraform cannot complete its execution plan or resolve the reference if no VPC resource or ID value exists to pass into that argument.
* API Deployment Error: If you manually bypassed Terraform's dependency graph or hardcoded a non-existent VPC ID string (e.g., `vpc_id = "vpc-12345678"`), the Cloud Provider's API (such as AWS) would immediately reject the request. AWS requires a valid, active `VPC ID` to attach a subnet to, so the API would throw an error like `InvalidVpcID.NotFound`, causing terraform apply to fail.

3. Find all implicit dependencies in your config and list them

| Resource | Depends on | Why |
|---|---|---|
| `aws_subnet.my-subnet` | `aws_vpc.my-vpc` | `vpc_id = aws_vpc.my-vpc.id` |
| `aws_internet_gateway.my-gateway` | `aws_vpc.my-vpc` | `vpc_id = aws_vpc.my-vpc.id` |
| `aws_route_table.my-table` | `aws_vpc.my-vpc` | `vpc_id = aws_vpc.my-vpc.id` |
| `aws_route_table.my-table` | `aws_internet_gateway.my-gateway` | `gateway_id = aws_internet_gateway.my-gateway.id` |
| `aws_route_table_association.my-rt-as` | `aws_subnet.my-subnet` | `subnet_id = aws_subnet.my-subnet.id` |
| `aws_route_table_association.my-rt-as` | `aws_route_table.my-table` | `route_table_id = aws_route_table.my-table.id` |

---

### ✅ Task 4 : Add a Security Group and EC2 Instance

Add to your config:

1. `aws_security_group` in the VPC:
   - Ingress rule: allow SSH (port 22) from `0.0.0.0/0`
   - Ingress rule: allow HTTP (port 80) from `0.0.0.0/0`
   - Egress rule: allow all outbound traffic
   - Tag: `"TerraWeek-SG"`

```hcl
resource aws_security_group my-sg {
    name = "TerraWeek-SG"
    description = "Allow ssh and http inbound traffic"
    vpc_id      = aws_vpc.my-vpc.id 

  ingress {    
    from_port   = 22
    protocol    = "tcp"
    to_port     = 22
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress { 
    from_port   = 80
    protocol    = "tcp"
    to_port     = 80
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"        # semantically equivalent to all ports
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

2. `aws_instance` in the subnet:
   - Use Amazon Linux 2 AMI for your region
   - Instance type: `t2.micro`
   - Associate the security group
   - Set `associate_public_ip_address = true`
   - Tag: `"TerraWeek-Server"`

```hcl
resource "aws_instance" "my-instance" {
  ami = "ami-048f644e868baa0e8"
  subnet_id = aws_subnet.my-subnet.id
  instance_type = "t2.micro"
  associate_public_ip_address = true

  tags = {
    Name = "TerraWeek-Server"
  }
}
```

Apply and verify -- your EC2 instance should have a public IP and be reachable.

<img width="1316" height="624" alt="image" src="https://github.com/user-attachments/assets/b5e74129-42b1-4420-9590-27d3b1e40c11" />

---

### ✅ Task 5 : Explicit Dependencies with depends_on

Sometimes Terraform cannot detect a dependency automatically.

1. Add a second `aws_s3_bucket` resource for application logs
2. Add `depends_on = [aws_instance.main]` to the S3 bucket -- even though there is no direct reference, you want the bucket created only after the instance
3. Run `terraform plan` and observe the order

```hcl
resource "aws_s3_bucket" "my-bucket" {
    bucket = "terra-bucket-hwb"
    depends_on = [ aws_instance.my-instance ]
}
```

Now visualize the entire dependency tree:
```bash
terraform graph | dot -Tpng > graph.png
```
If you don't have `dot` (Graphviz) installed, use:
```bash
terraform graph
```
and paste the output into an online Graphviz viewer.

<img width="1304" height="203" alt="graph" src="https://github.com/user-attachments/assets/b72db1a2-e97a-4f3e-aba5-172555f69950" />

**Document:** When would you use `depends_on` in real projects? Give two examples.

You use `depends_on` when a resource relies on another resource or configuration to exist or complete initialization first, but no direct attribute reference exists between them in the Terraform HCL code for Terraform to automatically detect an implicit dependency.

Real project uses:

* Forcing an IAM Policy Attachment Before Creating a Server

Sometimes an EC2 instance or server profile needs an IAM policy attached, but the server code does not directly use any attribute from the policy resource. Without depends_on, Terraform might build the server before the permissions are active

* Delaying a Script Execution Until a Database is Ready

When using a local or remote script provisioner that connects to a newly created database, you must make sure the database setup is complete. The script block might not reference a specific output value of the database cluster, so depends_on forces the correct sequence

---


### ✅ Task 6 : Lifecycle Rules and Destroy


1. Add a `lifecycle` block to your EC2 instance:
```hcl
lifecycle {
  create_before_destroy = true
}
```

```hcl
resource "aws_instance" "my-instance" {
  ami = "ami-0e5497a77ef21b5ac"
  subnet_id = aws_subnet.my-subnet.id
  instance_type = "t2.micro"
  associate_public_ip_address = true
  lifecycle {
    create_before_destroy = true
  }
  tags = {
    Name = "TerraWeek-Server"
  }
}
```

2. Change the AMI ID to a different one and run `terraform plan` -- observe that Terraform plans to create the new instance before destroying the old one

3. Destroy everything:
```bash
terraform destroy
```
4. Watch the destroy order -- Terraform destroys in reverse dependency order. Verify in the AWS console that everything is cleaned up.

**Document:** What are the three lifecycle arguments (`create_before_destroy`, `prevent_destroy`, `ignore_changes`) and when would you use each?

The three HCL lifecycle meta-arguments are create_before_destroy, prevent_destroy, and ignore_changes. They are used inside a resource block to change how Terraform manages resource creation, updates, and deletion.

1. `**create_before_destroy**`

* What it does: Reverses the default replacement order. It builds the new replacement resource first, attaches it, and then deletes the old resource.
* When to use it: Use this for zero-downtime updates on critical assets like load balancers, auto-scaling groups, or DNS records where deleting the old item first causes an outage.

2. `**prevent_destroy**`

* What it does: Fails the plan and blocks any operation that tries to delete or replace the resource.
* When to use it: Use this to protect crucial, stateful data stores like production databases, S3 storage buckets, or core networking items from accidental deletion.

3. `**ignore_changes**`

* What it does: Tells the tool to skip tracking differences for specific attributes listed in an array.
* When to use it: Use this when an external system or manual process modifies attributes (like auto-scaling tags or specific cloud configurations) after creation, and you want to prevent Terraform from overwriting them

---

## Note
- `aws_vpc.main.id` syntax: `<resource_type>.<resource_name>.<attribute>`
- Use `terraform fmt` to keep your HCL clean
- CIDR `10.0.0.0/16` gives you 65,536 IPs, `10.0.1.0/24` gives you 256
- If you cannot SSH into the instance, check: security group rules, public IP, route table, internet gateway
- `terraform graph` outputs DOT format -- paste it into webgraphviz.com if you don't have Graphviz
- Always destroy resources when done to avoid AWS charges
