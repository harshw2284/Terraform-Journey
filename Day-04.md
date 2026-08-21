# Terraform – Day 04 - Terraform State Management and Remote Backends

The state file is the single most important thing in Terraform. It is the source of truth -- the map between your `.tf` files and what actually exists in the cloud. Lose it and Terraform forgets everything. Corrupt it and your next apply could destroy production.

Today I learned to manage state like a professional -- remote backends, locking, importing existing resources, and handling drift.

---

### ✅ Task 1: Inspect Your Current State

Use your Day 3 config (or create a small config with a VPC and EC2 instance). Apply it and then explore the state:

```bash
terraform show                                    # Full state in human-readable format
terraform state list                              # All resources tracked by Terraform
terraform state show aws_instance.<name>          # Every attribute of the instance
terraform state show aws_vpc.<name>               # Every attribute of the VPC
```

Answer:
1. How many resources does Terraform track?

Terraform tracks every resource defined in your configuration files that has been successfully created or managed via `terraform apply` or added via `terraform import`.

2. What attributes does the state store for an EC2 instance? (hint: way more than what you defined)

In addition to your explicit arguments (like `ami`, `instance_type`, and `tags`), the state captures metadata provided by AWS upon resource creation:

* Identity & Amazon Resource Identifiers
* Network & Addressing Details: private_ip, public_ip, private_dns, public_dns, etc.
* Hardware & Location: availability_zone, tenancy, cpu_core_count, etc.
* State & Status: instance_state (e.g., running) ,password_data, and lifecycle flags, etc.
* Attached Infrastructure & Devices: root_block_device ,key_name, etc.
* Computed Metadata: tags_all (combining resource tags with provider-level default tags).

3. Open `terraform.tfstate` in an editor -- find the `serial` number. What does it represent?

In a Terraform state file (`terraform.tfstate`), the `serial` number is an incremental integer that acts as a version counter for the state file itself.

**What It Represents**

* State Lineage & Evolution Tracker: Every time Terraform modifies or updates the state file (e.g., after running `terraform apply`, `terraform refresh`, or state manipulation commands like `terraform state mv`), the `serial` number automatically increments by `1`.

* Conflict Prevention: When using remote state backends (like AWS S3 or Terraform Cloud) or state locking, Terraform uses the `serial` number to ensure it is working with the most up-to-date version of the state and to prevent race conditions or overwriting newer changes with older state data.

**Example**:

If `serial` value is `12` that means the state file has undergone 12 distinct modifications since it was first created (which starts at `serial: 1` or `0`).

---

### ✅ Task 2 : Set Up S3 Remote Backend

Storing state locally is dangerous -- one deleted file and you lose everything. Time to move it to S3.

1. First, create the backend infrastructure (do this manually or in a separate Terraform config):
```bash
# Create S3 bucket for state storage
aws s3api create-bucket \
  --bucket terraweek-state-<yourname> \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

# Enable versioning (so you can recover previous state)
aws s3api put-bucket-versioning \
  --bucket terraweek-state-<yourname> \
  --versioning-configuration Status=Enabled

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name terraweek-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-south-1
```

```hcl
resource "aws_s3_bucket" "my_state_bucket" {
  bucket = "terraweek-state-harsh" 
}

resource "aws_s3_bucket_versioning" "state_versioning" {
  bucket = aws_s3_bucket.my_state_bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_dynamodb_table" "my_state_dbtable" {
  name           = "terraweek-state-lock"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

2. Add the backend block to your Terraform config:
```hcl
terraform {
  backend "s3" {
    bucket         = "terraweek-state-<yourname>"
    key            = "dev/terraform.tfstate"
    region         = "ap-south-1"
    use_lockfile   = true                               # Used for state locking
    encrypt        = true
  }
}
```

3. Run:
```bash
terraform init
```
Terraform will ask: "Do you want to copy existing state to the new backend?" -- say yes.

4. Verify:
   - Check the S3 bucket -- you should see `dev/terraform.tfstate`
   - Your local `terraform.tfstate` should now be empty or gone
   - Run `terraform plan` -- it should show no changes (state migrated correctly)

---

### ✅ Task 3 : Test State Locking

State locking prevents two people from running `terraform apply` at the same time and corrupting the state.

1. Open **two terminals** in the same project directory
2. In Terminal 1, run:
```bash
terraform apply
```
3. While Terminal 1 is waiting for confirmation, in Terminal 2 run:
```bash
terraform plan
```
4. Terminal 2 should show a **lock error** with a Lock ID

**Document:** What is the error message? Why is locking critical for team environments?

5. After the test, if you get stuck with a stale lock:
```bash
terraform force-unlock <LOCK_ID>
```

---

### ✅ Task 4 : Import an Existing Resource

Not everything starts with Terraform. Sometimes resources already exist in AWS and you need to bring them under Terraform management.

1. Manually create an S3 bucket in the AWS console -- name it `terraweek-import-test-<yourname>`
2. Write a `resource "aws_s3_bucket"` block in your config for this bucket (just the bucket name, nothing else)

```hcl
resource "aws_s3_bucket" "imported" {
  bucket = "terraweek-import-test-harsh"
}
```

3. Import it:
```bash
terraform import aws_s3_bucket.imported terraweek-import-test-<yourname>
terraform import aws_s3_bucket.imported terraweek-import-test-harsh
```

<img width="1007" height="344" alt="Screenshot 2026-08-19 224358" src="https://github.com/user-attachments/assets/006b3a57-ef84-4de9-931f-3b483afd988c" />


4. Run `terraform plan`:
   - If you see "No changes" -- the import was perfect
   - If you see changes -- your config does not match reality. Update your config to match, then plan again until you get "No changes"

5. Run `terraform state list` -- the imported bucket should now appear alongside your other resources

<img width="723" height="102" alt="image" src="https://github.com/user-attachments/assets/7228c3a9-af99-47d8-9fd0-0060a34366c7" />

**Document:** What is the difference between `terraform import` and creating a resource from scratch?

The primary difference lies in where the infrastructure originates and how Terraform associates it with state:

Creating a Resource from Scratch (`terraform apply`):
You write the HCL configuration block, and Terraform creates a brand-new cloud resource in AWS (or another provider) upon running `terraform apply`. Terraform automatically creates the physical infrastructure and maps it to your state file simultaneously.

Importing an Existing Resource (`terraform import`):
The cloud resource already exists outside of Terraform management (created manually via AWS Console, CLI, or another tool). Running `terraform import` updates your Terraform state file so Terraform becomes aware of the existing resource and its current attributes. However, `terraform import` does not generate the code for you; you must write the corresponding resource block in your `.tf` file manually so state and configuration match.

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

### ✅ Task 6 : Simulate and Fix State Drift

State drift happens when someone changes infrastructure outside of Terraform -- through the AWS console, CLI, or another tool.

1. Apply your full config so everything is in sync
2. Go to the **AWS console** and manually:
   - Change the Name tag of your EC2 instance to `"ManuallyChanged"`
   - Change the instance type if it's stopped (or add a new tag)
3. Run:
```bash
terraform plan
```
You should see a **diff** -- Terraform detects that reality no longer matches the desired state.

4. You have two choices:
   - **Option A:** Run `terraform apply` to force reality back to match your config (reconcile)
   - **Option B:** Update your `.tf` files to match the manual change (accept the drift)

5. Choose Option A -- apply and verify the tags are restored.

6. Run `terraform plan` again -- it should show "No changes." Drift resolved.

**Document:** How do teams prevent state drift in production? (hint: restrict console access, use CI/CD for all changes)

Teams prevent and manage Terraform state drift in production by strictly locking down cloud console access, automating continuous scheduled drift detection, and enforcing a single GitOps deployment pipeline. Treating version-controlled code as the absolute source of truth stops unauthorized out-of-band modifications.

---

## Note
- S3 bucket names must be globally unique
- DynamoDB table must have a `LockID` string key -- this is what Terraform uses for locking
- `terraform init -migrate-state` explicitly triggers state migration
- `terraform refresh` (or `terraform apply -refresh-only`) updates state to match real infrastructure without making changes
- State locking only works with backends that support it (S3+DynamoDB, Consul, Terraform Cloud)
- `terraform force-unlock` should only be used when you are sure no other operation is running
- Always version your S3 bucket so you can recover a previous state file if something goes wrong
