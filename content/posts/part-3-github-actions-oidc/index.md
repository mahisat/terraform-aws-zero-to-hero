---
title: "Zero-to-Hero Part 3: Automated Deployment for Your Part 2 Infrastructure (GitHub Actions & OIDC)"
date: 2026-09-27T09:00:00+05:30
draft: false
tags: ["GitHub Actions AWS", "OIDC federation", "AWS IAM role trust policy", "SSM Run Command", "CI/CD EC2", "GitHub Actions secrets"]
description: "Automate deploy of the three-tier stack from Part 2 — GitHub Actions workflows, OIDC to AWS, SSM to private EC2, SSH frontend deploy, and debugging without long-lived access keys."
summary: "Automate deployment for the infrastructure you built in Part 2 — GitHub Actions, AWS OIDC, SSM to private EC2, and SSH frontend deploy without long-lived access keys."
ShowToc: true
weight: 3
aliases:
  - /posts/part-2-github-actions-oidc/
---
**Previous:** [Part 2 — Terraform + AWS](/posts/part-2-terraform-aws-three-tier/)  
**Index:** [Series home](/)

---

## Companion repository

**Read this post alongside the code:** [github.com/mahisat/aws-basic-3-tier-architecture](https://github.com/mahisat/aws-basic-3-tier-architecture).

This repo covers **Parts 2 and 3** — upcoming series posts will point to a new repository when it is published. Part 2 infrastructure and the Todo app live here; Part 3 adds automation in [`.github/workflows/`](https://github.com/mahisat/aws-basic-3-tier-architecture/tree/main/.github/workflows) and the IAM/OIDC pieces under `terraform/`. Keep the repo open while you read so workflow YAML, trust policies, and deploy steps map directly to files you can inspect and diff against your own fork.

---

## Who this is for

You completed **Part 2** (or built the same stack in [Part 1](/posts/part-1-aws-console-three-tier/)) — infrastructure exists. Now you want **push-to-deploy**: change backend code, push to `main`, and see it on AWS without manual SSH for every file.

This article explains **GitHub Actions** terms, **OIDC** to AWS, each **workflow** file, and how to fix failures — including GitHub’s **new OIDC `sub` claim** format for new repositories.

---

## GitHub Actions — vocabulary for beginners

| Term | What it is | Why it matters |
|------|------------|----------------|
| **Workflow** | YAML file in `.github/workflows/` | Defines automation on GitHub events |
| **Event (`on`)** | Trigger (e.g. `push` to `main`) | Controls when jobs run |
| **Job** | Group of steps on one runner | Backend deploy = one job |
| **Step** | Single task (checkout, run script) | Runs sequentially |
| **Runner** | VM that executes steps (`ubuntu-latest`) | GitHub-hosted = public internet |
| **Action** | Reusable step (`actions/checkout@v4`) | Avoid rewriting common tasks |
| **Secret** | Encrypted repo setting | PEM keys, role ARNs, passwords |
| **Variable** | Non-secret repo config | Region, repo URL for Terraform CI |
| **Environment** | Optional gate (approvals) | Protect `production` apply |
| **Permissions** | Token scopes for the workflow | OIDC needs `id-token: write` |
| **Concurrency** | One deploy at a time | Prevents race on same EC2 |

**Where configured:** Repository → **Settings** → Secrets and variables → Actions; workflow YAML in your repo.

---

## Why not store AWS access keys in GitHub?

**Long-lived `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`:**

- Leak from logs, forks, or compromised repos.  
- Hard to rotate; broad permissions if attached to power user.

**OIDC (OpenID Connect):**

- GitHub mints a **short-lived token** per workflow run.  
- AWS **trusts** GitHub’s identity provider and **assumes an IAM role** with scoped permissions.  
- No static AWS keys in GitHub Secrets for those jobs.

**Where in AWS:** IAM → **Identity providers** → `token.actions.githubusercontent.com`.  
**Where in GitHub:** Workflow step `aws-actions/configure-aws-credentials@v4` with `role-to-assume`.

---

## GitHub OIDC `sub` claim — classic vs new (important)

AWS trust policies often restrict **who** can assume a role using the JWT **`sub`** (subject) claim.

**Classic format (older repos):**

```text
repo:my-org/my-repo:ref:refs/heads/main
```

**New format (many new repositories):** includes **immutable numeric IDs**:

```text
repo:my-org@12345678/my-repo@9876543210:ref:refs/heads/main
```

**Why:** Renaming org/repo does not break federation identity.

**Beginner mistake:** Copying a tutorial trust policy with `repo:org/name:ref:...` when GitHub sends `repo:org@ID/name@ID:ref:...` → **`Not authorized to perform sts:AssumeRoleWithWebIdentity`**.

**How to debug:**

1. Add a temporary workflow step to print claims (or use GitHub’s OIDC documentation).  
2. In AWS IAM role **Trust relationships**, match **`StringLike`** on `token.actions.githubusercontent.com:sub` to the **actual** claim (often include `*` carefully or use both patterns).  
3. Also set **`aud`** = `sts.amazonaws.com`.

**Example trust condition (illustrative — adjust IDs):**

```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
  },
  "StringLike": {
    "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG@*/YOUR_REPO@*:ref:refs/heads/main"
  }
}
```

Verify exact syntax against [GitHub’s OIDC docs](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) for your repo type.

---

## AWS setup: OIDC provider + IAM role (backend deploy)

### Step 1 — OIDC identity provider (once per account)

**What:** Registers GitHub as a federated issuer.  
**Where:** IAM → Identity providers → Add provider → OpenID Connect.  
**URL:** `https://token.actions.githubusercontent.com`  
**Audience:** `sts.amazonaws.com`

### Step 2 — IAM role trust policy

**What:** Says “GitHub Actions from this repo may assume this role.”  
**Where:** IAM → Roles → Create role → Web identity → GitHub.

Bind **`sub`** and **`aud`** as above.

### Step 3 — Permissions policy

**What:** What AWS APIs the role may call.  
**Where:** Attach `terraform/iam-github-deploy-least-privilege.json` (SSM send command, describe instances).

**Why not AdministratorAccess:** Role can only deploy, not delete the whole VPC.

### Step 4 — GitHub secret

**Name:** `AWS_DEPLOY_ROLE_ARN`  
**Value:** `arn:aws:iam::ACCOUNT_ID:role/your-deploy-role`

---

## GitHub setup checklist

| Item | Where | Purpose |
|------|--------|---------|
| `AWS_DEPLOY_ROLE_ARN` | Secrets | Backend workflow OIDC |
| `EC2_PRIVATE_KEY` | Secrets | SSH to **frontend** (public) |
| `FRONTEND_EC2_PUBLIC_IP` | Secrets | From `terraform output` |
| `BACKEND_EC2_PRIVATE_IP` | Secrets | nginx `proxy_pass` target |
| Workflow `permissions` | YAML | `id-token: write` for OIDC jobs |

**Optional (Terraform CI):** `AWS_TERRAFORM_ROLE_ARN`, `TF_VAR_db_password`, repo variable `GITHUB_REPO_URL`.

---

## Workflow 1: `backend-deployment.yml` — line by line

**Purpose:** Deploy API to **private** EC2 without SSH from GitHub.

**Trigger:**

```yaml
on:
  push:
    branches: [main]
    paths: ['backend/**', '.github/workflows/backend-deployment.yml']
  workflow_dispatch:
```

**Why paths:** Avoid redeploying backend when only frontend changes.

**Permissions:**

```yaml
permissions:
  contents: read
  id-token: write
```

**Why `id-token`:** Required for OIDC token minting.

**Configure AWS credentials:**

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
    aws-region: us-east-1
```

**What happens:** GitHub requests OIDC token → AWS STS → temporary credentials on runner.

**Find instance:**

```bash
aws ec2 describe-instances --filters "Name=tag:Name,Values=my-project-backend-server" ...
```

**Why tag:** Stable target after you know naming convention from Terraform `var.project`.

**SSM Run Command:**

```bash
aws ssm send-command --document-name AWS-RunShellScript --instance-ids ...
```

**Why SSM:** Runner is on the public internet; **private IP is not SSH-reachable**. SSM uses AWS API + agent on instance.

**Commands on instance:** `git pull`, `npm ci`, `npm run build`, `systemctl restart todo-backend`, `curl` health.

**Common failures:**

| Error | Cause | Fix |
|-------|--------|-----|
| AssumeRoleWithWebIdentity denied | Wrong `sub` in trust | Update trust for new repo ID format |
| No running instance | Wrong tag / stopped EC2 | Check Name tag and state |
| SSM access denied | Role missing `ssm:SendCommand` | Fix IAM policy |
| Command Failed | No git repo on disk | Fix Part 2 user_data first |
| npm build fails | No devDependencies | Run `npm ci` not `npm install --omit=dev` before build |

---

## Workflow 2: `frontend-deployment.yml` — line by line

**Purpose:** Copy frontend to **public** EC2, build, update nginx.

**Auth:** **SSH** with `EC2_PRIVATE_KEY` — no OIDC required for this workflow (could be improved later).

**Flow:**

1. `scp` frontend folder to EC2.  
2. Write nginx config with **`BACKEND_EC2_PRIVATE_IP`**.  
3. `npm ci` && `npm run build` → copy `dist` to `/var/www/frontend`.  
4. **SELinux:** `setsebool httpd_can_network_connect 1` (fixes many **502** errors).  
5. `nginx -t` && `systemctl reload nginx`.  
6. `curl` verify from runner.

**Why `/api` in React:** Browser calls same origin; nginx proxies to backend private IP — **do not** put private IP in `VITE_*` for public users.

**Common mistakes:**

| Mistake | Result |
|---------|--------|
| Wrong workflow name in `paths:` filter | Push does not trigger workflow |
| Stale `BACKEND_EC2_PRIVATE_IP` after EC2 replace | 502 Bad Gateway |
| Skip `npm run build` | Old or missing static files |

---

## How GitHub authenticates to AWS (OIDC flow)

```text
1. Workflow starts → GitHub issues OIDC JWT (sub, aud, ref, repository)
2. configure-aws-credentials → STS AssumeRoleWithWebIdentity
3. AWS validates JWT against GitHub IdP + trust policy
4. Runner gets temporary AccessKey/SessionToken (minutes)
5. aws ssm send-command / aws ec2 describe-* using temp creds
6. Credentials expire automatically
```

**Why it’s secure:** No permanent keys; trust tied to repo/branch/ref.

---

## Debugging GitHub Actions

For EC2 and log commands (`ssh`, `journalctl`, `systemctl`, `curl`), see the [Linux commands reference](/reference/linux-commands-reference/).

1. **Actions tab** → failed run → expand failed step logs.  
2. **OIDC:** CloudTrail `AssumeRoleWithWebIdentity` events.  
3. **SSM:** Read stdout/stderr from `get-command-invocation` (workflow prints them).  
4. **SSH frontend:** Test locally: `ssh -i key.pem ec2-user@PUBLIC_IP`.  
5. **Secrets:** No quotes in secret values; PEM must include full `BEGIN/END` lines.

---

## Architecture evaluation (learning vs production)

This section closes the **Zero-to-Hero** series for this stack. Use it to decide what to keep for learning and what to change for real workloads.

### Advantages (why this is good for learning)

- Covers **full stack**: network, compute, database, firewall rules, bootstrap, CI/CD.  
- **Three-tier separation** mirrors real enterprise patterns at small scale.  
- **Terraform** + **GitHub Actions** are industry-standard skills.  
- **OIDC** teaches modern credential hygiene.  
- **SSM** introduces managing private servers without bastion SSH.

### Disadvantages / limits

- **Single EC2 per tier** — no Auto Scaling.  
- **Single-AZ RDS** (in typical lab config) — no Multi-AZ failover.  
- **NAT Gateway** cost and complexity for a tiny app.  
- **SSH + PEM** for frontend deploy — operational burden.  
- **HTTP only** — no TLS on ALB/CloudFront.  
- **user_data + git clone** — brittle; drift between instances.  
- **Local Terraform state** — team collaboration risk.

### Security considerations

| Area | Learning stack | Production direction |
|------|----------------|---------------------|
| SSH | Often open to world on frontend | SSM only, bastion, or VPN |
| Secrets | `.env` on disk, tfvars local | Secrets Manager, SSM Parameter Store |
| DB creds | In user_data/terraform vars | Rotate, never in user_data |
| Network | SG tiering (good baseline) | NACLs, WAF, private API Gateway |
| IAM | Least-privilege JSON in repo | IAM Access Analyzer, SCPs |

### Scalability

- **Vertical only** (change instance size).  
- Bottleneck: single Node process, single RDS instance.  
- **Improve:** ALB + Auto Scaling Group, read replicas, cache (ElastiCache).

### Resilience & availability

- EC2 or AZ failure → **downtime** until manual recovery.  
- RDS without Multi-AZ → maintenance/failover risk.  
- **Improve:** Multi-AZ RDS, ASG across AZs, health checks with ALB.

### Disaster recovery & backup

- **RDS:** enable automated backups + snapshot retention; document restore runbook.  
- **App code:** Git is source of truth.  
- **Terraform state:** backup or remote state with versioning.  
- **RTO/RPO:** not defined in lab — define for production.

### Failure scenarios

| Scenario | Likely impact | Detection |
|----------|---------------|-----------|
| Frontend EC2 stopped | Site down | Uptime check |
| Backend systemd crash | 502 on `/api` | ALB health (future), journalctl |
| RDS storage full | API errors | CloudWatch RDS alarms |
| NAT failure | No git/npm on private nodes | SSM logs, bootstrap fail |
| Wrong OIDC trust | Deploy pipeline fails | GitHub Actions logs |
| Expired GitHub secret IP | 502 after replace | Update secrets after terraform |

### Single points of failure

- One frontend EC2, one backend EC2, one NAT GW, one RDS instance (lab).  
- **Mitigation in prod:** redundancy per tier, Multi-AZ, multiple NAT (cost), global DNS.

### Operational complexity

- Many moving parts for a Todo app — **intentional for education**.  
- Prod simplifies ops with managed services (Elastic Beanstalk, ECS, Lambda) at cost of abstraction.

### Cost (recap)

NAT + RDS + 24/7 EC2 adds up. **Destroy** lab resources when idle.

### Verdict

| Use case | Fit |
|----------|-----|
| **Learning Terraform/AWS/Actions** | Excellent |
| **Portfolio demo** | Good with HTTPS + cleanup story |
| **Production customer data** | **Not sufficient** without hardening table above |

---

## Series outcome — what you should know now

From **“I don’t know Terraform or AWS”** to:

- Understanding **each major file** in the repo and how tiers connect.  
- Running **Terraform plan/apply** and reading **state/outputs**.  
- Debugging **user_data**, **systemd**, **nginx**, and **RDS** connectivity.  
- Configuring **GitHub OIDC** (including **new `sub` claim** awareness).  
- Deploying via **SSM** and **SSH** workflows and fixing common CI failures.  
- Judging **what to improve** for production.

**Planned next articles:** private GitHub repos on EC2, S3 remote state + Terraform in Actions, cost optimization, ALB + HTTPS production path.

**Further reading:** [Linux commands reference](/reference/linux-commands-reference/) · [Appendix — Terraform & project files](/reference/appendix-terraform-and-project-files/) (companion to Part 2)

---
