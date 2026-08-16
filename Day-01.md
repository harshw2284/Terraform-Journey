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

### ✅ Task 2 : Create a Headless Service

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

### ✅ Task 3 : Create a StatefulSet

1. Write a StatefulSet manifest with `serviceName` pointing to your Headless Service

```yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: web
  serviceName: web-service
```

2. Set replicas to 3, use the nginx image

```yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: web
  serviceName: web-service
  replicas: 3

  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
          name: web
```

3. Add a `volumeClaimTemplates` section requesting 100Mi of ReadWriteOnce storage

```yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: web
  serviceName: web-service
  replicas: 3

  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: web-data
          mountPath: /usr/share/nginx/html

  volumeClaimTemplates:
  - metadata:
      name: web-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 100Mi
```

4. Apply and watch: `kubectl get pods -l <your-label> -w`

Observe ordered creation — `web-0` first, then `web-1` after `web-0` is Ready, then `web-2`.

Check the PVCs: `kubectl get pvc` — you should see `web-data-web-0`, `web-data-web-1`, `web-data-web-2` (names follow the pattern `<template-name>-<pod-name>`).


Verify: What are the exact pod names and PVC names?

Pod names are :

* web-0
* web-1
* web-2

PVC names are :

* web-data-web-0
* web-data-web-1
* web-data-web-2

---

### ✅ Task 4 : Stable Network Identity

Each StatefulSet pod gets a DNS name: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`

1. Run a temporary busybox pod and use `nslookup` to resolve `web-0.<your-headless-service>.default.svc.cluster.local`

```bash
kubectl run busybox --image=busybox:1.36 --restart=Never -it --rm -- sh

nslookup web-0.web-service.default.svc.cluster.local
```

2. Do the same for `web-1` and `web-2`

```bash
nslookup web-1.web-service.default.svc.cluster.local
nslookup web-2.web-service.default.svc.cluster.local
```

3. Confirm the IPs match `kubectl get pods -o wide`

Verify: Does the nslookup IP match the pod IP ?

Yes ! Both nslookup IP and pod IP matched.

---

### ✅ Task 5 : Stable Storage — Data Survives Pod Deletion

1. Write unique data to each pod: `kubectl exec web-0 -- sh -c "echo 'Data from web-0' > /usr/share/nginx/html/index.html"`

2. Delete `web-0`: `kubectl delete pod web-0`

3. Wait for it to come back, then check the data — it should still be "Data from web-0"

The new pod reconnected to the same PVC.

Verify: Is the data identical after pod recreation ?

Yes ! Data is identical.

---

### ✅ Task 6 : Ordered Scaling

1. Scale up to 5: `kubectl scale statefulset web --replicas=5` — pods create in order (web-3, then web-4)

2. Scale down to 3 — pods terminate in reverse order (web-4, then web-3)

```bash
kubectl scale statefulset web --replicas=3
```

3. Check `kubectl get pvc` — all five PVCs still exist. Kubernetes keeps them on scale-down so data is preserved if you scale back up.

Verify: After scaling down, how many PVCs exist ?

After scaling down, 5 PVCs exist.

---

### ✅ Task 7 : Clean Up

1. Delete the StatefulSet and the Headless Service

2. Check `kubectl get pvc` — PVCs are still there (safety feature)

3. Delete PVCs manually

Verify: Were PVCs auto-deleted with the StatefulSet ?

No, PVCs are not auto-deleted when you delete a StatefulSet.

This is an intentional Kubernetes safety feature designed to prevent accidental data loss. If you want to remove the storage, you must delete the PVCs manually after deleting the StatefulSet.

**Note**:

* `kubectl get sts` is the short name for StatefulSets
* `serviceName` must match an existing Headless Service
* Pod DNS: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`
* PVC naming: `<template-name>-<statefulset-name>-<ordinal>`
* Pods create in order (0, 1, 2) and terminate in reverse (2, 1, 0)
* Scaling down does not delete PVCs — data is preserved
* Deleting a StatefulSet does not delete PVCs — clean up separately

---
