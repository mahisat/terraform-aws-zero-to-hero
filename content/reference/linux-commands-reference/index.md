---
title: "Linux Commands Reference"
date: 2025-09-01T08:00:00+05:30
draft: false
description: "Commands used during AWS deployment and troubleshooting in this project."
ShowToc: true
weight: 100
---

Short explanations for commands used during AWS deployment and troubleshooting. Run most diagnostic commands on EC2 via **SSM Session Manager** or SSH.

---

## Navigation and files

| Command | What it does | When to use |
|---------|--------------|-------------|
| `cd /path` | Change directory | Enter `terraform/`, `backend/`, or app folder on EC2 |
| `ls` | List files | See if `backend/`, `dist/` exist |
| `ls -la` | List including hidden + permissions | Debug “permission denied” |
| `pwd` | Print working directory | Confirm where you are |
| `cat file` | Print file contents | Read `.env`, nginx config (careful with secrets) |
| `tail -n 50 file` | Last 50 lines of a file | Log files |
| `tail -f file` | Follow log live | Watch app log (Ctrl+C to stop) |

**Beginner mistake:** `sudo cd ...` — `cd` is a shell builtin; use `cd` normally or `sudo -u ec2-user -i` for a login shell.

---

## Permissions and users

| Command | What it does | When to use |
|---------|--------------|-------------|
| `sudo command` | Run as root | Install packages, `systemctl`, read protected logs |
| `sudo -u ec2-user bash` | Run shell as app user | Build/run npm as owner of `/home/ec2-user/app` |
| `chmod 600 file` | Owner read/write only | SSH private key, `.env` |
| `chown user:group path` | Change owner | Fix root-owned files after mistaken `npm` as root |

**On EC2:** SSM may log you in as `ssm-user` — use `sudo` to inspect `ec2-user` files.

---

## Networking tests

| Command | What it does | When to use |
|---------|--------------|-------------|
| `curl -v URL` | HTTP request with details | Test `/health`, `/api/todos`, backend :5000 |
| `curl -sf URL` | Fail silently on HTTP errors | Scripts and CI checks |
| `ss -tlnp` | Listening TCP ports | Confirm Node listens on `:5000` |

**Interpretation:** `Connection refused` = nothing listening or wrong IP. `Timeout` = security group or routing.

---

## Services (systemd)

| Command | What it does | When to use |
|---------|--------------|-------------|
| `systemctl status todo-backend` | Service state | Backend up/down? |
| `systemctl restart todo-backend` | Restart API | After deploy or `.env` change |
| `systemctl enable todo-backend` | Start on boot | Persist across reboot |
| `journalctl -u todo-backend -n 50` | Service logs | Stack traces, DB errors |
| `systemctl status nginx` | Web server state | Frontend / proxy issues |
| `nginx -t` | Test nginx config syntax | Before reload |

---

## Node.js / app

| Command | What it does | When to use |
|---------|--------------|-------------|
| `npm ci` | Clean install from lockfile | CI and servers (reproducible) |
| `npm run build` | Compile TypeScript → `dist/` | **Required** before `npm start` |
| `npm start` | Run `node dist/server.js` | Manual test (prefer systemd in prod) |
| `npm run dev` | Local dev with hot reload | On laptop only |

---

## SSH / copy (frontend deploy)

| Command | What it does | When to use |
|---------|--------------|-------------|
| `ssh user@host` | Remote shell | Frontend EC2 (public IP) |
| `scp -r dir user@host:path` | Copy files over SSH | GitHub Actions frontend workflow |
| `chmod 600 ~/.ssh/id_rsa` | Restrict key permissions | SSH requires this |
| `ssh-keyscan -H host >> known_hosts` | Trust host key | Non-interactive CI SSH |

---

## Terraform (on your laptop)

| Command | What it does | When to use |
|---------|--------------|-------------|
| `terraform init` | Download providers | First time / after provider change |
| `terraform plan` | Preview changes | Before every apply |
| `terraform apply` | Apply changes | Create/update AWS |
| `terraform destroy` | Tear down | Stop billing |
| `terraform output name` | Print output values | IPs for secrets |

---

## AWS CLI (GitHub runner or laptop with creds)

| Command | What it does | When to use |
|---------|--------------|-------------|
| `aws ec2 describe-instances ...` | Find instances | Backend deploy workflow |
| `aws ssm send-command ...` | Run script on EC2 | Backend deploy |
| `aws sts get-caller-identity` | Who am I? | Verify OIDC role assumed |

---

## SELinux (Amazon Linux + nginx proxy)

| Command | What it does | When to use |
|---------|--------------|-------------|
| `setsebool -P httpd_can_network_connect 1` | Allow nginx outbound proxy | Fix **502** to backend |

---

## Logs specific to this project

| Path | Content |
|------|---------|
| `/var/log/cloud-init-output.log` | user_data / cloud-init |
| `/var/log/backend-setup.log` | Backend bootstrap script |
| `/var/log/frontend-setup.log` | Frontend bootstrap script |

Example:

```bash
sudo tail -100 /var/log/backend-setup.log
```

---

Return to [Part 1](/posts/part-1-aws-console-three-tier/) | [Part 2](/posts/part-2-terraform-aws-three-tier/) | [Part 3](/posts/part-3-github-actions-oidc/) | [Series home](/)
