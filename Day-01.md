# Terraform – Day 01 - Introduction to Terraform and Your First AWS Infrastructure

I have been deploying containers, writing CI/CD pipelines, and orchestrating workloads on Kubernetes. But who creates the servers, networks, and clusters underneath ? Today I started my Infrastructure as Code journey with Terraform -- the tool that let us define, provision, and manage cloud infrastructure by writing code.

By the end of today, I will have created real AWS resources using nothing but a `.tf` file and a terminal.

---

### ✅ Task 1: Understand Infrastructure as Code

Before touching the terminal, research and write short notes on:

**1. What is Infrastructure as Code (IaC) ? Why does it matter in DevOps ?**

Infrastructure as Code (IaC) is a method where you set up and manage computers, servers, and networks using text files with code instead of manual clicking or physical setup.

Why IaC Matters in DevOps :

* **Stops mistakes**: Writing code removes human errors from manual server setups.
* **Keeps things the same**: Dev, test, and live systems match completely, fixing the "it works on my PC" issue.
* **Tracks changes**: Code goes into version control (like Git) so teams see who changed what and can roll back fast.
* **Speeds up work**: Teams launch entire data centers in minutes using tools like Terraform or AWS CloudFormation.
* **Helps scaling**: Systems grow or shrink automatically by updating the code parameters.

**2. What problems does IaC solve compared to manually creating resources in the AWS console ?**

Infrastructure as Code (IaC) solves human error, configuration drift, and scaling limits of the AWS console by using readable text files to create, track, and repeat cloud setups automatically.

Core Problems Solved by IaC : 

* **Human Error**: Manual clicks lead to typos or missed settings. Code ensures every setup is exact.
* **Configuration Drift**: Systems change over time by hand. Code keeps all environments identical.
* **No History**: Click changes lack records of who did what. Code uses Git to track every change.
* **Slow Scaling**: Setting up multi-region servers by hand takes days. Code deploys them in minutes.
* **No Reusability**: One-time console clicks cannot be easily copied. Code templates build new environments fast.

**3. How is Terraform different from AWS CloudFormation, Ansible, and Pulumi ?**

Terraform differs from AWS CloudFormation, Ansible, and Pulumi primarily in its combination of cloud-agnostic flexibility, use of a domain-specific declarative language (HCL), and its primary focus on infrastructure provisioning over software configuration. While all four are Infrastructure as Code (IaC) tools, they target different stages of deployment, languages, and cloud environments.

Core Structural Differences

| Feature | Terraform | AWS CloudFormation | Ansible | Pulumi |
| :--- | :--- | :--- | :--- | :--- |
| **Primary Purpose** | Infrastructure Provisioning | Infrastructure Provisioning | Configuration Management | Infrastructure Provisioning |
| **Cloud Support** | Cloud-agnostic (Multi-cloud) | AWS-exclusive (Native) | Cloud-agnostic / Server-focused | Cloud-agnostic (Multi-cloud) |
| **Language Type** | Declarative (HCL) | Declarative (YAML/JSON) | Procedural / Hybrid (YAML) | Imperative / Declarative Hybrid |
| **Language Used** | HashiCorp Configuration Language (HCL) | YAML or JSON | YAML (Playbooks) | General-purpose programming languages (TypeScript, Python, Go, etc.) |
| **State Tracking** | Managed via state files (`terraform.tfstate`) | Managed automatically by AWS backend | Stateless (Discovers live state dynamically) | Managed via Pulumi Service or backend state file |

**4. What does it mean that Terraform is "declarative" and "cloud-agnostic" ?** 

Terraform is declarative because you write code describing the final infrastructure you want, and it figures out the steps to build it. It is cloud-agnostic because it uses one workflow and language (HCL) to manage resources across different cloud companies like AWS, Azure, and Google Cloud.

**Declarative vs. Imperative**

* **Imperative approach**: You write step-by-step commands (e.g., "Step 1: create network, Step 2: launch server, Step 3: attach disk"). If a step fails, you must fix and run the rest manually.
* **Declarative approach** (Terraform): You write a goal state (e.g., "I want three servers and one database"). Terraform reads your current setup, compares it to your goal, and automatically creates, updates, or deletes only what is needed

---

### ✅ Task 2 : Install Terraform and Configure AWS

1. Install Terraform:
```bash
# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Linux (amd64)
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Windows
choco install terraform
```

2. Verify:
```bash
terraform -version
```

3. Install and configure the AWS CLI:
```bash
aws configure
# Enter your Access Key ID, Secret Access Key, default region (e.g., ap-south-1), output format (json)
```

4. Verify AWS access:
```bash
aws sts get-caller-identity
```

You should see your AWS account ID and ARN.


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
