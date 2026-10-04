# Zero-to-Hero blog (Hugo + PaperMod)

**Standalone git repository.** This Hugo site is maintained separately from the application code—it is not a submodule of the Part 1/2 app repo.

**Parts 1–3** use the companion code in [mahisat/aws-basic-3-tier-architecture](https://github.com/mahisat/aws-basic-3-tier-architecture) (bootstrap scripts, Terraform, frontend, backend, and GitHub Actions workflows). Later series posts will reference a different repository when that is published.

This site uses [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme. Series posts are under `content/posts/` (`part-1-aws-console-three-tier`, `part-2-terraform-aws-three-tier`, `part-3-github-actions-oidc`).

## Prerequisites

- Hugo **Extended** (this project was built with v0.166+). On Windows with WinGet: `winget install Hugo.Hugo.Extended`
- Git (for the PaperMod theme submodule)

After installing Hugo, open a new terminal so `hugo` is on your `PATH`, or use the full path to `hugo.exe` from WinGet.

## First-time setup

```powershell
cd blog-series
git submodule update --init --recursive
```

## Local preview

```powershell
hugo server -D
```

Open http://localhost:1313/ to read and edit the series locally.
