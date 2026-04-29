# Contributing Blog Posts

Thanks for writing for Evotec. Keep your submission focused, practical, and reproducible.

## New Post Checklist

- Fork this repository and open a pull request from your fork.
- Create one folder per post: `posts/<language>/<year>/<slug>/`.
- Put the article in `index.md`.
- Put the cover image beside `index.md` as `cover.jpg`, `cover.png`, or `cover.webp`.
- Put screenshots in `images/`.
- Use Markdown image syntax with useful alt text.
- Do not use SVG files, scripts, HTML snippets, tracking pixels, or remote images.
- Add or update your author profile in `authors/<slug>.yml`.
- Run validation before opening a pull request.

Start from:

- `templates/author.yml`
- `templates/post/index.md`

## Front Matter

```yaml
---
title: "Building DHCP Reports with PowerShell"
description: "A practical guide to generating DHCP reports with PowerShell."
date: "2026-05-01"
language: "en"
authors:
  - your-name
categories:
  - PowerShell
tags:
  - dhcp
  - reporting
image: "./cover.jpg"
image_alt: "PowerShell report preview showing DHCP scope usage"
draft: true
---
```

Use `draft: true` while writing. Maintainers publish accepted posts during import.

## Images

Use local relative paths:

```markdown
![DHCP scope report showing utilization percentages](./images/screenshot-01.png)
```

Allowed image formats are PNG, JPG, JPEG, WEBP, and GIF. Keep individual files under 5 MB and the whole post image set under 30 MB.
