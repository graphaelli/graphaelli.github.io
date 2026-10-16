# g.raphaelli.com

Personal blog powered by [Hugo](https://gohugo.io/) and the [hugo-kiera](https://github.com/funkydan2/hugo-kiera) theme, deployed automatically to GitHub Pages.

---

## Prerequisites

1. **Git Submodules**: Ensure theme submodules are checked out:
   ```bash
   git submodule update --init --recursive
   ```
2. **Hugo Extended**: If Hugo is not already on your path, install it using Go:
   ```bash
   go install -tags extended github.com/gohugoio/hugo@v0.141.0
   ```
   Ensure `~/go/bin` is in your `$PATH` (e.g. `export PATH="$HOME/go/bin:$PATH"` in `~/.zshrc`).

---

## Local Development & Preview

Start the local server with live-reload and draft previews enabled:

```bash
hugo server -D
```

Open [http://localhost:1313/](http://localhost:1313/) in your browser.

---

## Creating a New Post

Posts live in the `content/posts/` directory.

### 1. Generate the post skeleton
Use the `hugo new` command:
```bash
hugo new posts/$(date +%Y-%m-%d)-my-post-title.md
```

This generates a file with the standard TOML front matter:
```toml
+++
date = '2026-10-15T20:45:00-04:00'
draft = true
title = 'My Post Title'
+++
```

### 2. Write and preview
- While editing, the local server (`hugo server -D`) will reload automatically.
- Images and static assets can be placed in `static/` and referenced directly (e.g. `static/my-image.png` is served at `/my-image.png`).

### 3. Publish
When you're ready to publish:
1. Set `draft = false` (or delete the line) in the post front matter.
2. Commit and push to `main`:
   ```bash
   git add content/posts/$(date +%Y-%m-%d)-my-post-title.md
   git commit -m "Add post: My Post Title"
   git push origin main
   ```
3. GitHub Actions will automatically build and deploy to [g.raphaelli.com](https://g.raphaelli.com/).
