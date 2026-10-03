---
title: "Zero-to-Hero Part 1: Build a Three-Tier AWS Architecture with Terraform"
date: 2025-09-01T09:00:00+05:30
draft: false
tags: ["AWS for beginners", "Terraform tutorial", "three-tier architecture", "VPC networking", "EC2 user data", "RDS MySQL", "NAT Gateway", "infrastructure as code"]
description: "VPC, public/private subnets, EC2, RDS, security groups, IAM, and bootstrap scripts — explained for complete beginners."
summary: "Build a three-tier Todo app on AWS with Terraform — VPC, EC2, RDS, security groups, IAM, and bootstrap scripts, explained for complete beginners."
ShowToc: true
weight: 1
---
**Previous:** [Series index](/)  
**Next:** [Part 2 — GitHub Actions & OIDC](/posts/part-2-github-actions-oidc/)

---

## Companion repository

**Read this post alongside the code:** [github.com/mahisat/aws-basic-3-tier-architecture](https://github.com/mahisat/aws-basic-3-tier-architecture).

That repo is the **code reference for Part 1 and Part 2** (later installments in this series will use a separate repository). It includes Terraform under `terraform/`, the React app under `frontend/`, and the API under `backend/`. Browse files on GitHub or clone the repo while you read; when this article mentions a path (for example `terraform/vpc.tf`), open the same file there for a concrete picture of what we are building.

---

## Who this is for

You might be thinking: *“I don’t know Terraform or AWS.”* This article is the **first step** in a series. We focus on **concepts**, not copy-paste without understanding. By the end you should know **what** each file does, **why** it exists, **where** it is configured, and **how** to debug when something breaks.

---

## What we are building

A **three-tier** Todo application:

| Tier | Role | In this project |
|------|------|-----------------|
| **Presentation** | User interface | React (Vite) on EC2 in a **public** subnet |
| **Application** | Business logic / API | Node.js + Express on EC2 in a **private** subnet |
| **Data** | Persistent storage | Amazon RDS for MySQL in **private** subnets |

Traffic flow:

```text
Browser → Frontend EC2 (nginx :80) → /api proxied to → Backend EC2 (:5000) → RDS (:3306)
```

Private instances reach the internet (git, npm) via a **NAT Gateway** in the public subnet.

---

## AWS concepts you need first

### Region and Availability Zone (AZ)

- **Region** (e.g. `us-east-1`): geographic area. You choose one in `terraform.tfvars`.
- **AZ** (e.g. `us-east-1a`): isolated data center within a region. RDS requires subnets in **at least two AZs**.

### VPC (Virtual Private Cloud)

**What:** Your private network in AWS.  
**Why:** Isolates your servers from other customers; you control IP ranges and routing.  
**Where:** `terraform/vpc.tf`.

### Subnets

**What:** IP ranges inside the VPC.  
**Why:** **Public** subnets can reach the internet via an Internet Gateway (IGW). **Private** subnets cannot — they use a **NAT Gateway** for outbound-only internet.  
**Where:** `terraform/vpc.tf` — e.g. `10.0.1.0/24` public, `10.0.2.0/24` and `10.0.3.0/24` private.

**Beginner mistake:** Putting the database in a public subnet. We keep RDS **private**.

### Internet Gateway vs NAT Gateway

| Resource | Purpose |
|----------|---------|
| **Internet Gateway (IGW)** | Bidirectional internet for **public** subnets |
| **NAT Gateway** | Outbound internet for **private** subnets (no inbound from internet) |

**Why NAT costs money:** It runs 24/7. Fine for learning; expensive for idle labs. See [Cost](#cost-considerations).

### Security groups

**What:** Virtual firewall attached to ENI/EC2/RDS.  
**Why:** Allow only required ports (e.g. HTTP to frontend, API from frontend SG to backend, MySQL from backend to RDS).  
**Where:** `terraform/security-group.tf`.

**Beginner mistake:** Opening SSH (`22`) or MySQL (`3306`) to `0.0.0.0/0`.

### EC2 and user_data

**What:** Virtual machine; **user_data** is a script that runs **once** at first boot (via cloud-init).  
**Why:** Automate install: clone repo, `npm build`, start systemd/nginx.  
**Where:** `terraform/ec2.tf` + `backend-setup.sh` / `frontend-setup.sh`.

**Beginner mistake:** Expecting `terraform apply` to re-run user_data on existing instances. It does **not**, unless you **replace** the instance or use `user_data_replace_on_change = true`.

### RDS

**What:** Managed MySQL.  
**Why:** You don’t patch the OS or manage MySQL process yourself.  
**Where:** `terraform/rds.tf`.

---

## Terraform concepts you need first

### What is Terraform?

**What:** Tool that reads `.tf` files and calls AWS APIs to create/update/delete resources.  
**Why:** Repeatable, reviewable infrastructure in Git.  
**Where:** All files under `terraform/`.

### Provider

**What:** Plugin that talks to AWS.  
**Why:** Terraform core doesn’t know AWS; the **hashicorp/aws** provider does.  
**Where:** `terraform/versions.tf`, `terraform/provider.tf`.

```hcl
provider "aws" {
  region = var.region
}
```

**Important:** `region` comes from a **variable**, not hardcoded in every resource.

### Resource

**What:** One AWS object, e.g. `aws_instance`, `aws_vpc`.  
**Why:** Each block declares desired state.

### Variable

**What:** Inputs (project name, passwords, repo URL).  
**Why:** Reuse code; keep secrets out of `.tf` files.  
**Where:** `terraform/variables.tf` + `terraform.tfvars` (local, gitignored).

### Output

**What:** Values printed after apply (frontend IP, backend private IP).  
**Why:** Configure GitHub Secrets, curl tests, debugging.  
**Where:** `terraform/outputs.tf`.

### State

**What:** File (`terraform.tfstate`) mapping Terraform addresses to real AWS IDs.  
**Why:** Terraform knows what it created on the next plan.  
**Where:** Local by default in `terraform/` (gitignored).

**Beginner mistake:** Committing `terraform.tfstate` to Git (can leak secrets).  
**Production:** Remote state on S3 + DynamoDB lock (future blog).

### Modules

**What:** Reusable packages of `.tf` files.  
**Why:** This learning repo uses a **flat** layout (no modules) to stay readable. In production you’d split `modules/vpc`, `modules/ec2`, etc.

### plan vs apply

| Command | What it does |
|---------|----------------|
| `terraform plan` | Preview changes (safe to run anytime) |
| `terraform apply` | Create/update/delete in AWS |
| `terraform destroy` | Tear down managed resources |

---

## Repository layout — every important file

### Root

| File | Purpose |
|------|---------|
| `README.md` | Quick start and links to this series |
| `.gitignore` | Ignore OS/IDE files, PEM keys, `.aws/` |

### `terraform/` — infrastructure

| File | What | Why |
|------|------|-----|
| `versions.tf` | Terraform & provider version constraints | Reproducible installs |
| `provider.tf` | AWS provider config | Region and API access |
| `variables.tf` | Input declarations | Parameterize project |
| `terraform.tfvars.example` | Sample values | Template for your `terraform.tfvars` |
| `terraform.tfvars` | **Your** values (gitignored) | Passwords, repo URL |
| `data.tf` | Data sources (AMI, AZs) | Look up Amazon Linux 2023 AMI |
| `vpc.tf` | VPC, subnets, routes, IGW | Networking foundation |
| `nat-gateway.tf` | EIP + NAT | Private subnet outbound internet |
| `security-group.tf` | Firewalls | Least-privilege traffic |
| `rds.tf` | RDS + subnet group | Database tier |
| `ec2.tf` | Key pair, EC2, user_data | App servers |
| `iam.tf` | EC2 instance profile for SSM | Manage private backend without SSH |
| `outputs.tf` | IPs, instance IDs | Ops and CI secrets |
| `backend-setup.sh` | Backend first-boot script | Clone, build, systemd |
| `frontend-setup.sh` | Frontend first-boot script | Clone, build, nginx |
| `nginx-frontend.conf.tpl` | nginx config template | Proxy `/api` to backend |
| `todo-backend.service` | systemd unit | Run Node API on boot |
| `install-backend-service.sh` | Manual repair script | Fix failed bootstrap |
| `iam-terraform-least-privilege.json` | IAM policy for Terraform user/role | Least privilege apply |
| `iam-github-deploy-least-privilege.json` | IAM policy for SSM deploy | Used in Part 2 |
| `IAM.md` | IAM documentation | Who needs which policy |
| `.gitignore` | Ignore state, `.terraform/`, tfvars, PEM | Safety |

### `backend/` — API (application code)

| Path | Purpose |
|------|---------|
| `package.json` | Scripts: `build` (tsc), `start` (node dist) |
| `src/server.ts` | Starts HTTP server on `0.0.0.0:5000` |
| `src/app.ts` | Express app, mounts `/api`, `/health` |
| `src/config/env.ts` | Validates `.env` |
| `src/config/database.ts` | MySQL pool + schema |
| `src/routes/`, `controllers/`, `services/`, `repositories/` | Layered API |

### `frontend/` — UI

| Path | Purpose |
|------|---------|
| `vite.config.ts` | Dev server + proxy `/api` → localhost:5000 |
| `src/api/todos.ts` | Calls `/api/todos` (production: same origin via nginx) |
| `src/config/api.ts` | `VITE_API_BASE_URL` handling |

### `.github/workflows/` — Part 2

Deploy workflows are explained in [Part 2](/posts/part-2-github-actions-oidc/).

---

## Deep dive: key Terraform files

### `vpc.tf` — routing explained

Public route table: `0.0.0.0/0` → **Internet Gateway**.  
Private route table: `0.0.0.0/0` → **NAT Gateway**.

**Common mistake:** Adding invalid `gateway_id = "local"` routes. AWS creates local VPC routes automatically.

### `ec2.tf` — wiring user_data

```hcl
user_data = base64encode(templatefile("${path.module}/backend-setup.sh", {
  github_repo  = var.github_repo_url
  db_endpoint  = aws_db_instance.database.address
  db_password  = var.db_password
  systemd_unit = file("${path.module}/todo-backend.service")
}))
user_data_replace_on_change = true
```

**What:** `templatefile` injects Terraform variables into the shell script, then base64 encodes for AWS.  
**Why:** Backend needs DB hostname and credentials at boot.  
**Beginner mistake:** Using `${BASH_VAR}` in the script that Terraform tries to interpret — use Terraform vars like `${db_endpoint}` or escape bash as `$${VAR}`.

### Public repository and `git clone`

Bootstrap runs:

```bash
git clone --depth 1 "${github_repo}" /home/ec2-user/app
```

**Why it works today:** The repo URL in `terraform.tfvars` points to a **public** GitHub repository. No token required.

**Upcoming blog:** Private repos need deploy keys, PAT in Secrets Manager, or artifact-based deploy — not plain `git clone`.

### `backend-setup.sh` — what happens on boot

1. Wait for **network** (NAT must work before GitHub is reachable).  
2. **Clone** repo with retries.  
3. Write **`.env`** (quoted heredoc so `$` in passwords is safe).  
4. **`npm ci` && `npm run build`** — creates `dist/server.js`.  
5. Wait for RDS **TCP on 3306** (no mysql CLI package needed).  
6. Install **systemd** unit and start **`todo-backend`**.

**Debug logs:**

```bash
sudo tail -100 /var/log/cloud-init-output.log
sudo tail -100 /var/log/backend-setup.log
sudo journalctl -u todo-backend -n 50
```

See [Linux commands reference](/reference/linux-commands-reference/).

---

## Step-by-step: deploy infrastructure

### 1. Prerequisites

- AWS account, IAM user or role with policies in `iam-terraform-least-privilege.json` (adjust account ID in ARNs).  
- Terraform installed locally.  
- AWS credentials configured (`aws configure` or environment variables).

### 2. Configure variables

```bash
cp terraform/terraform.tfvars.example terraform/terraform.tfvars
# Edit: github_repo_url, db_password, project name
```

### 3. Init, plan, apply

```bash
cd terraform
terraform init      # Downloads providers, creates .terraform.lock.hcl
terraform validate    # Syntax check
terraform plan        # Preview
terraform apply       # Create resources (~10–15 min first time)
```

### 4. Verify outputs

```bash
terraform output frontend_public_ip
terraform output -raw backend_private_ip
```

### 5. Test (after bootstrap completes)

Browser: `http://<frontend_public_ip>/`  
From frontend EC2: `curl http://<backend_private_ip>:5000/health`

---

## IAM for Terraform (least privilege)

**Why:** Your IAM user should not use `AdministratorAccess` in a real account.

**Where:** Attach `iam-terraform-least-privilege.json` (fix account ID `123456789012` → yours).

**When Terraform fails with IAM permission errors:**

Terraform (and the AWS provider during **plan** and **apply**) calls many API actions, including **Describe\*** reads to refresh state. If your policy is too narrow, AWS returns **`UnauthorizedOperation`** or **`AccessDenied`** and names the **exact action** missing (for example `ec2:DescribeKeyPairs`).

**What to do (repeat until `terraform plan` is clean):**

1. Copy the **action name** from the error message (e.g. `ec2:DescribeInstanceAttribute`).
2. In **IAM** → your user or role → **Permissions**, add that action to the policy (scope to `"Resource": "*"` for this learning stack, or tighten later).
3. Optionally merge additions into [`iam-terraform-least-privilege.json`](../../../../01-3-tier-basic/terraform/iam-terraform-least-privilege.json) so your repo documents the full set.
4. Run **`terraform plan`** again, then **`terraform apply`**.

**Beginner mistake:** Adding only “create” permissions. Terraform also needs **read/describe** permissions for resources it manages. One error often hides the next until you fix them one at a time.

**Not an IAM issue:** If the error is about RDS subnet groups needing **two Availability Zones**, add a second private subnet (`private_subnet_cidr_2`) in `vpc.tf` — that is configuration, not a missing IAM action.

**Examples we hit in this project** (add these if your policy matches our starter JSON and you see the same errors): `ec2:DescribeKeyPairs`, `ec2:DescribeAddressesAttribute`, `ec2:DescribeInstanceAttribute`.

---

## Troubleshooting playbook (Part 1)

| Symptom | Likely cause | What to do |
|---------|--------------|------------|
| No `/home/ec2-user/app/backend` | user_data failed early | cloud-init logs; replace EC2 |
| `Connection refused` :5000 | API not running / no build | `systemctl status todo-backend`; `npm run build` |
| `Cannot find module dist/server.js` | Skipped `npm run build` | Build before start |
| 502 on `/api` | nginx or backend | Part 2 + SELinux `httpd_can_network_connect` |
| `templatefile` missing key | Bash `${VAR}` vs Terraform | Use `${db_endpoint}` or `$${bash_var}` |
| Permission denied in SSM | Logged in as `ssm-user` | `sudo -u ec2-user` or `sudo ls` |

---

## Cost considerations

| Resource | Billing note |
|----------|----------------|
| NAT Gateway | Hourly + data; largest fixed cost in this lab |
| RDS | Instance hours + storage |
| EC2 t3.micro | Often free tier eligible |
| Elastic IP on NAT | Associated with NAT |

**Tip:** Run `terraform destroy` when not learning.

---

## What you learned in Part 1

- How **VPC, subnets, IGW, NAT** connect.  
- How **security groups** enforce tier isolation.  
- How **Terraform** declares AWS resources, **variables**, **outputs**, and **state**.  
- How **user_data** bootstraps apps from a **public** Git repo.  
- How to **read logs** and fix common bootstrap failures.

**Next:** [Part 2 — GitHub Actions, OIDC, and automated deployment](/posts/part-2-github-actions-oidc/) — then the [architecture evaluation](/posts/part-2-github-actions-oidc/#architecture-evaluation-learning-vs-production) at the end of Part 2.

**Further reading:** [Appendix — every Terraform & project file (deep dive)](/reference/appendix-terraform-and-project-files/) · [Linux commands reference](/reference/linux-commands-reference/)

---
