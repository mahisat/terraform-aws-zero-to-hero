---
title: "Basic & Advanced Three Tier AWS"
description: "AWS three-tier architecture — console and Terraform basics, then GitHub Actions; advanced hardening and production patterns coming next."
---

**Outcome (Basic track):** After Parts 1–3, you should understand how AWS networking and tiers fit together (console and Terraform), how EC2 bootstrap works, how GitHub Actions deploys code, how OIDC replaces long-lived AWS keys, and how to debug common failures.

## Basic Three Tier (read in order)

1. [Part 1 — AWS Console](/posts/part-1-aws-console-three-tier/)
2. [Part 2 — Terraform](/posts/part-2-terraform-aws-three-tier/)
3. [Part 3 — GitHub Actions & OIDC](/posts/part-3-github-actions-oidc/)

Companion code: [mahisat/aws-basic-3-tier-architecture](https://github.com/mahisat/aws-basic-3-tier-architecture).

## Advanced Three Tier (coming soon)

**Advanced Three Tier Part 1** and later parts build on the same Todo stack with production-oriented changes (private repos on EC2, remote Terraform state, cost tuning, ALB, HTTPS, Multi-AZ, and more).

**Before Advanced Part 1:** finish all three [Basic Three Tier](/posts/part-1-aws-console-three-tier/) posts above so VPC, Terraform, and deployment automation are not new concepts.
