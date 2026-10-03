# Zero-to-Hero blog (Hugo + PaperMod)

**Standalone git repository.** This Hugo site is maintained separately from the application code—it is not a submodule of the Part 1/2 app repo.

**Parts 1 and 2** use the companion code in [mahisat/aws-basic-3-tier-architecture](https://github.com/mahisat/aws-basic-3-tier-architecture) (Terraform, frontend, backend, and GitHub Actions workflows). Later series posts will reference a different repository when that is published.

This site uses [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme. Parts 1 and 2 of the series are under `content/posts/`.

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
