# Zero-to-Hero blog (Hugo + PaperMod)

**Standalone git repository.** It lives next to the `01-3-tier-basic` project under `architecture-aws/`, but it is not part of that repo’s history. Clone or push this folder on its own remote.

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

Open http://localhost:1313/

## Build static site

```powershell
hugo --minify
```

For GitHub Pages or a custom domain, pass your real site URL so feeds and canonical URLs are correct:

```powershell
hugo --minify --baseURL "https://your-username.github.io/your-repo/"
```

Output is in `public/`.
