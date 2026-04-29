---
title: "How to Contribute a Blog Post to Evotec"
description: "A visual walkthrough for preparing a guest post with Markdown, local images, author credit, and pull request review."
date: "2026-05-01"
language: "en"
authors:
  - przemyslaw-klys
categories:
  - Community
  - Documentation
tags:
  - contributions
  - markdown
  - github
image: "./cover.webp"
image_alt: "Illustration of a pull request card, Markdown document, and image thumbnails on a clean desktop workspace"
draft: true
---

This example shows the shape of a strong contribution. It is intentionally stored under `examples/`, so it will not be imported into the private website. To submit a real post, copy this structure into `posts/<language>/<year>/<slug>/`.

![A clean folder structure with a Markdown article, image thumbnail, screenshots folder, and author profile card](./images/post-folder.webp)

## 1. Create One Folder For The Post

Keep the article and its images together:

```text
posts/en/2026/my-post-slug/
  index.md
  cover.webp
  images/
    screenshot-01.webp
```

This makes review easier and prevents images from being separated from the article they belong to.

## 2. Add Author Credit

Each writer should have one profile in `authors/<author-slug>.yml`. Posts reference authors by slug:

```yaml
authors:
  - your-name
```

Author profiles can include an X profile, LinkedIn profile, and personal website.

## 3. Use Local Images With Alt Text

Use local relative image links:

```markdown
![Useful description of the screenshot](./images/screenshot-01.webp)
```

Remote images, tracking pixels, raw HTML images, and SVG uploads are not accepted for normal posts.

![A review workflow board with checkmarks, comment bubbles, and a publishing arrow](./images/review-workflow.webp)

## 4. Open A Pull Request

Before opening the pull request, run:

```powershell
powerforge-web contributions validate --root .
```

GitHub Actions will run the same validation when the pull request is opened. Maintainers review the content, merge accepted posts, and import them into the private website repository.
