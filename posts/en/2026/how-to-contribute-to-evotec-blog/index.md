---
title: "How to Contribute a Blog Post to evotec.xyz"
description: "A practical walkthrough for preparing a guest post with Markdown, local images, author credit, and pull request review."
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

Evotec articles are written in Markdown, reviewed through GitHub pull requests, and imported into the private website only after maintainers accept them. This keeps the publishing workflow open to contributors while protecting the production website.

This guide shows the shape of a strong contribution: one article folder, local images, clear author credit, and a small review surface.

![A clean folder structure with a Markdown article, image thumbnail, screenshots folder, and author profile card](./images/post-folder.webp)

## 1. Create One Folder For The Article

Keep the article and its images together:

```text
posts/en/2026/my-article-slug/
  index.md
  cover.webp
  images/
    screenshot-01.webp
```

This makes review easier and prevents images from being separated from the article they belong to. It also lets the importer copy everything into stable website asset paths.

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

Remote images, tracking pixels, raw HTML images, and SVG uploads are not accepted for normal posts. Keep screenshots focused on the thing you are explaining and crop out unrelated personal or customer data.

![A review workflow board with checkmarks, comment bubbles, and a publishing arrow](./images/review-workflow.webp)

## 4. Open A Pull Request

Before opening the pull request, run:

```powershell
powerforge-web contributions validate --root .
```

GitHub Actions will run the same validation when the pull request is opened. Maintainers review the content, ask for changes when needed, merge accepted posts, and import them into the private website repository.

That is the whole contract: contributors get a simple public writing workflow, and the production site stays maintainer-controlled.
