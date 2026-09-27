---
title: "Appendix: Terraform & Project Files"
date: 2025-09-01T08:30:00+05:30
draft: false
description: "Deep dive on every Terraform and bootstrap file in the three-tier project."
ShowToc: true
weight: 101
---

Companion to [Part 1](/posts/part-1-terraform-aws-three-tier/). For each file: **what**, **why**, **where it runs**, **common mistakes**.

---

## `versions.tf`

**What:** Locks Terraform core and provider versions (`aws`, `tls`, `local`).  
**Why:** Same provider behavior on your laptop and in CI.  
**Mistake:** Deleting `.terraform.lock.hcl` without re-testing apply.

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws  = { source = "hashicorp/aws", version = "~> 6.0" }
    tls  = { source = "hashicorp/tls", version = "~> 4.0" }
    local = { source = "hashicorp/local", version = "~> 2.5" }
  }
}
```

- **`tls`:** Generates SSH key in Terraform.  
- **`local`:** Writes `.pem` file to disk for GitHub Secret `EC2_PRIVATE_KEY`.

---

## `provider.tf`

**What:** Configures the AWS provider region from `var.region`.  
**Why:** All resources default to one region unless overridden.

---

## `variables.tf` / `terraform.tfvars`

**What:** Inputs; values live in **gitignored** `terraform.tfvars`.  
**Why:** Passwords and repo URL must not be committed.

| Variable | Purpose |
|----------|---------|
| `project` | Name prefix for resources (`my-project-web-server`) |
| `github_repo_url` | **Public** clone URL for user_data |
| `db_password` | RDS master password (sensitive) |
| `frontend_vite_api_base_url` | Usually `/api` for nginx same-origin |
| `frontend_use_backend_private_api_url` | `false` for public browsers |

**Mistake:** `$` in password with unquoted bash heredocs in old scripts — use quoted `'ENVFILE'` in `backend-setup.sh`.

---

## `data.tf`

**What:** Read-only lookups.

```hcl
data "aws_ami" "amazon_linux_2023" { ... }
data "aws_availability_zones" "available" { state = "available" }
```

**Why:** Hard-coded AMI IDs expire; data source always finds a current AL2023 image.

---

## `vpc.tf` + `nat-gateway.tf`

**Resources:** `aws_vpc`, subnets (1 public + 2 private), IGW, route tables, associations, EIP, NAT.

**Why two private subnets:** RDS subnet group requires **two AZs**.

**How traffic works:**

- Public EC2 → IGW → internet.  
- Private EC2 → NAT (in public subnet) → IGW → internet.  
- Private EC2 → backend SG → RDS (never via NAT).

---

## `security-group.tf`

Layered rules:

1. **Web SG:** `:80` from world.  
2. **SSH SG:** `:22` (learning only — tighten for prod).  
3. **Backend SG:** `:5000` from **web SG only**.  
4. **DB SG:** `:3306` from **backend SG only**.

**Mistake:** Attaching DB SG in Terraform but forgetting `vpc_security_group_ids` on `aws_db_instance` (we attach in `rds.tf`).

---

## `rds.tf`

**What:** `aws_db_subnet_group` + `aws_db_instance` (MySQL 8, `db.t3.micro`).  
**Why:** Managed DB; credentials from variables.  
**Mistake:** Hardcoding username/password in resource block while passing different values to user_data.

---

## `ec2.tf`

**Resources:** TLS key, `aws_key_pair`, `local_file` PEM, `aws_instance` backend + frontend.

**Important attributes:**

- `user_data` + `user_data_replace_on_change = true`  
- `iam_instance_profile` → SSM  
- Frontend: `associate_public_ip_address = true`  
- Backend: private subnet, no public IP  

**Depends_on:** Web server after backend (nginx needs backend private IP in template).

---

## `iam.tf`

**What:** Role + `AmazonSSMManagedInstanceCore` + instance profile.  
**Why:** Backend deploy via SSM without SSH from internet.  
**Where used:** Both EC2 instances (can use for frontend too).

---

## `outputs.tf`

Export IPs and instance IDs for GitHub Secrets and debugging.  
**Mistake:** Forgetting to refresh secrets after `terraform apply -replace` EC2.

---

## Bootstrap scripts

| Script | Runs on | Purpose |
|--------|---------|---------|
| `backend-setup.sh` | Backend first boot | clone, build, systemd |
| `frontend-setup.sh` | Frontend first boot | clone, build, nginx |
| `install-backend-service.sh` | Manual | Repair failed bootstrap |
| `nginx-frontend.conf.tpl` | Embedded in user_data | `/api` → backend:5000 |
| `todo-backend.service` | systemd | `node dist/server.js` |

---

## IAM JSON policies

| File | Consumer |
|------|----------|
| `iam-terraform-least-privilege.json` | Human/CI running `terraform apply` |
| `iam-github-deploy-least-privilege.json` | GitHub OIDC role for SSM |

Replace placeholder account ID in role ARNs.

---

## Application entry points (why the stack behaves as it does)

**Backend `server.ts`:** Binds `0.0.0.0:5000` so nginx on another host can connect.

**Backend `app.ts`:** Mounts routes at `/api` and `/health`.

**Frontend `todos.ts`:** Calls `/api/todos` — in production, nginx on same host forwards to private backend.

---

[Back to Part 1](/posts/part-1-terraform-aws-three-tier/) · [Part 2](/posts/part-2-github-actions-oidc/)
