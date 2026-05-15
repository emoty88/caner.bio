---
title: "How I Rebuilt My Personal Site with Hugo"
date: 2026-05-14
draft: false
tags: ["hugo", "web", "github-actions", "papermod"]
description: "How I moved from a static Bootstrap page to a Hugo site with PaperMod, a custom resume template, and auto-deploy via GitHub Actions — and how you can do the same."
---

My old site was a single `index.html` Bootstrap page. No blog, no easy way to add projects, and the resume lived in a completely separate repo. It worked but it was a pain to maintain. So I rebuilt it with Hugo.

Here's how I did it — and how you can do the same.

## What is Hugo

Hugo is a static site generator written in Go. You write content in markdown, define layouts in HTML templates, and Hugo compiles everything into a folder of static files ready to be served from anywhere — GitHub Pages, Netlify, S3, whatever you want.

No runtime, no database, no server to maintain. Just files.

It's also extremely fast. A full site build typically takes under a second. The dev server live-reloads on every save, so the feedback loop while writing is instant.

## Installation

On macOS with Homebrew:

```bash
brew install hugo
```

Verify it worked:

```bash
hugo version
# hugo v0.147.1+extended darwin/arm64
```

Make sure you get the **extended** version — some themes including PaperMod require it for SCSS processing.

## Creating a New Site

```bash
hugo new site my-site
cd my-site
git init
```

This scaffolds the basic folder structure:

```
archetypes/    # templates for new content
content/       # your markdown files live here
layouts/       # HTML templates (overrides theme)
static/        # files copied as-is to output (images, favicons)
themes/        # your theme goes here
hugo.toml      # site configuration
```

## Adding the PaperMod Theme

I went with [PaperMod](https://github.com/adityatelange/hugo-PaperMod) — minimal, fast, and well maintained. Add it as a git submodule so it stays in sync with upstream without bloating your repo:

```bash
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod themes/PaperMod
```

Then tell Hugo to use it in `hugo.toml`:

```toml
baseURL = "https://yourdomain.com/"
title = "Your Name"
theme = "PaperMod"
```

When someone clones your repo they'll need to run this once to pull the theme:

```bash
git submodule update --init --recursive
```

## Configuring the Site

PaperMod has a profile mode that works well for a personal site — a landing page with a photo, a subtitle, and buttons to your main sections. Here's the relevant config:

```toml
[params]
  defaultTheme = "auto"

  [params.profileMode]
    enabled = true
    title = "Your Name"
    subtitle = "What you do, in one line."
    imageUrl = "img/photo.jpg"

    [[params.profileMode.buttons]]
      name = "Blog"
      url = "/posts"

    [[params.profileMode.buttons]]
      name = "Resume"
      url = "/resume"

  [[params.socialIcons]]
    name = "github"
    url = "https://github.com/yourusername"

  [[params.socialIcons]]
    name = "linkedin"
    url = "https://linkedin.com/in/yourprofile"
```

Put your profile photo at `static/img/photo.jpg` and it'll show up on the landing page.

## Content Structure

Everything in `content/` becomes a page. I organised mine like this:

```
content/
  resume.md        ← my resume
  about.md         ← about page
  search.md        ← search page
  posts/           ← blog posts
  projects/        ← open source projects
```

A new blog post is just a file:

```bash
hugo new content posts/my-first-post.md
```

The front matter at the top controls metadata:

```markdown
---
title: "My First Post"
date: 2026-05-14
draft: false
tags: ["go", "hugo"]
description: "A short summary shown in post listings."
---

Your content here.
```

Set `draft: true` while writing, flip it to `false` when you're ready to publish.

## Custom Template for the Resume

This was the part I was most particular about. I wanted the resume to live as a plain markdown file but render with a proper print layout — clean, full-width, with a Print/Save as PDF button.

Hugo lets you override the theme template for any content type. Create a file at `layouts/resume/single.html` and Hugo will use it instead of the default template whenever it renders a page with `type: resume` in its front matter.

The template itself is straightforward — extend the base layout, render the content, add a print button and some CSS:

```html
{{- define "main" }}
<div class="resume-page">
  <div class="resume-actions no-print">
    <button onclick="window.print()">Print / Save as PDF</button>
  </div>
  <article>{{- .Content }}</article>
</div>

<style>
  @media print {
    .no-print { display: none; }
    header, footer, nav { display: none; }
  }
</style>
{{- end }}
```

And in `content/resume.md`:

```markdown
---
title: "Resume"
type: "resume"
layout: "single"
---

# Your Name
...
```

Now the resume is just a markdown file. Update it, push, deployed.

## Auto-Deploy with GitHub Actions

The deploy workflow lives at `.github/workflows/deploy.yml`. It triggers on every push to `main`, installs Hugo, builds the site, and publishes to GitHub Pages:

```yaml
name: Deploy Hugo site to GitHub Pages

on:
  push:
    branches:
      - main

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.147.1
    steps:
      - name: Install Hugo
        run: |
          wget -O ${{ runner.temp }}/hugo.tar.gz \
            https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz
          tar -xzf ${{ runner.temp }}/hugo.tar.gz -C ${{ runner.temp }}
          sudo mv ${{ runner.temp }}/hugo /usr/local/bin/

      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Build
        run: hugo --minify

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy
        id: deployment
        uses: actions/deploy-pages@v4
```

One thing to set up in your GitHub repo: go to **Settings → Pages → Source** and switch it from "Deploy from a branch" to **"GitHub Actions"**. Do this before your first push to main.

After that, every commit to main is a deployment. Takes about a minute.

## The Workflow Going Forward

Write a post in markdown, commit, push. That's the entire loop. No CMS to log into, no build command to remember, no separate deploy step. The friction is low enough that I might actually keep this thing updated.

We'll see.
