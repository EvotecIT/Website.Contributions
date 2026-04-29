---
title: "Write for evotec.xyz"
description: "A practical guide to contributing a guest post to evotec.xyz with Markdown, local images, author credit, and pull request review."
date: "2026-04-29"
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
  - evotec
image: "./cover.webp"
image_alt: "A friendly illustration showing the contribution workflow for writing a post for evotec.xyz"
draft: true
---

evotec.xyz has always been built around practical engineering notes: scripts that solved a real problem, migration lessons, odd edge cases, troubleshooting stories, and the kind of technical details that save someone else an afternoon.

If you have something useful to share, you do not need access to the private website repository. This public contribution repository is the place where articles can be prepared, reviewed, and improved before publication.

The process is simple:

1. Fork or clone the public `Website.Contributions` repository.
2. Add your author profile.
3. Create one folder for your article.
4. Write the article in Markdown.
5. Keep the cover image and screenshots next to the article.
6. Open a pull request.
7. After review, maintainers import the accepted article into evotec.xyz.

That gives contributors a public place to write, keeps the production website protected, and makes sure every accepted article has proper author credit.

![A friendly end-to-end contribution workflow showing fork, author profile, article folder, Markdown, pull request, validation, review, and publishing](./images/contribution-workflow.webp)

## What You Can Contribute

Good articles do not need to be huge. The best posts are often practical, focused, and based on something real.

Useful topics include:

- PowerShell scripts that solved a real problem
- Microsoft 365, Entra ID, Exchange, Active Directory, Intune, Azure, or security lessons
- migration notes
- troubleshooting stories
- unusual edge cases
- tooling improvements
- open-source project usage
- automation patterns
- reports, dashboards, or monitoring ideas
- lessons learned from production work

Blog contribution folders currently support English and Polish articles:

```text
posts/en/my-article-slug/
posts/pl/moj-artykul/
```

English is always a safe default when no Polish version is planned. Other website interface languages may exist, but blog contribution intake should stay in `en` or `pl` unless maintainers agree to extend the workflow.

## 1. Keep Each Article In One Folder

Each article should live in its own folder.

Use this structure:

```text
posts/en/my-article-slug/
  index.md
  cover.webp
  images/
    screenshot-01.webp
```

The first folder after `posts/` is the language.

The next folder is the article slug.

Everything used by that article stays inside that folder.

![A visual folder structure showing posts, language, article slug, index.md, cover image, and local screenshots folder](./images/article-folder-structure.webp)

For example, an English article called `how-to-check-dns-records` would use:

```text
posts/en/how-to-check-dns-records/
  index.md
  cover.webp
  images/
    dns-result.webp
    powershell-output.webp
```

A Polish article could use:

```text
posts/pl/jak-sprawdzic-rekordy-dns/
  index.md
  cover.webp
  images/
    wynik-dns.webp
    wynik-powershell.webp
```

This structure keeps review predictable. Reviewers can see the article, cover image, and screenshots together. The publishing importer can also move the accepted article into the production website without guessing where files belong.

## 2. Create Your Author Profile

Every contributor should have an author profile.

Create a file under `authors/`:

```text
authors/your-name.yml
```

Example:

```yaml
name: "Your Name"
slug: "your-name"
title: "Your role or short bio"
x: "https://x.com/your-handle"
linkedin: "https://www.linkedin.com/in/your-profile/"
website: "https://example.com/"
```

The `slug` is what your article uses in front matter.

For example:

```yaml
authors:
  - your-name
```

You do not need to include every link. If you do not use X, LinkedIn, or a personal website, leave out what does not apply.

The important part is that contributors get visible credit for their work.

## 3. Create The Article File

Your article should be written in Markdown and saved as:

```text
index.md
```

A minimal article looks like this:

```markdown
---
title: "Your Article Title"
description: "A short summary of what the article explains."
date: "2026-04-29"
language: "en"
authors:
  - your-name
categories:
  - PowerShell
tags:
  - powershell
  - automation
image: "./cover.webp"
image_alt: "Describe what the cover image shows"
draft: true
---

Start with the problem.

Explain who it affects, why it matters, and what the reader will learn.

## The Problem

Describe the situation clearly.

## The Solution

Show the practical steps.

## Example

Add code, screenshots, commands, or configuration examples.

## Wrap-Up

Summarize the result and mention anything the reader should watch out for.
```

Keep the title clear. Avoid titles that are too vague.

Good:

```text
How To Find Stale Active Directory Computers With PowerShell
```

Less useful:

```text
PowerShell Tips
```

The description should be one sentence that explains what the reader gets from the article.

## 4. Use Local Images

Please keep images local to the article folder.

Good:

```markdown
![DNS lookup result in PowerShell](./images/dns-lookup-result.webp)
```

Also fine:

```markdown
![DNS lookup result in PowerShell](./images/dns-lookup-result.png)
```

Avoid remote images:

```markdown
![Screenshot](https://some-external-site.example/image.png)
```

Remote images can disappear, change, load slowly, or track readers. Local images are safer and easier to review.

## 5. Image Formats

The contribution validator accepts these local image formats:

```text
.webp
.png
.jpg
.jpeg
.gif
```

Prefer `.webp` when possible because it usually gives good quality with smaller file sizes.

Use `.png` when:

- the image contains small text that must stay very sharp
- the screenshot becomes blurry as `.webp`
- the image is a diagram, UI capture, or command output where precision matters

Use `.jpg` or `.jpeg` for photos when transparency is not needed. Use `.gif` only when animation adds real value to the explanation.

For cover images, prefer:

```text
cover.webp
```

For screenshots, use descriptive names:

```text
images/install-module-command.webp
images/entra-admin-center-setting.webp
images/powershell-output.png
```

Avoid names like:

```text
image1.png
screenshot.png
final-final-new.png
```

Good names make review easier.

## 6. Write Useful Alt Text

Every image should have useful alt text.

Good:

```markdown
![GitHub pull request showing successful validation checks](./images/pull-request-checks.webp)
```

Less useful:

```markdown
![image](./images/pull-request-checks.webp)
```

Alt text should briefly explain what the image shows. It helps readers who use screen readers, and it also makes the article easier to understand when images do not load.

## 7. Remove Sensitive Information

Before opening a pull request, check every screenshot.

Remove or blur:

- tenant names
- customer names
- email addresses
- access tokens
- secrets
- license keys
- internal hostnames
- private IP addresses when they are not needed
- user data
- production incident details that should not be public

If the screenshot is about a security or identity topic, be extra careful. Use demo data wherever possible.

## 8. Keep The Article Practical

evotec.xyz articles are usually most useful when they show a real problem and a working solution.

A good structure is:

```text
Problem
Why it matters
Requirements
Steps
Code or configuration
Expected result
Common mistakes
Wrap-up
```

When adding code, prefer complete examples over tiny fragments.

For PowerShell articles, include:

- required modules
- tested PowerShell version if relevant
- permissions needed
- example command
- example output
- notes about limitations

For Microsoft 365, Entra ID, Azure, Exchange, or Active Directory articles, include the exact portal path or command where it helps.

Example:

```text
Microsoft Entra admin center
Identity > Applications > Enterprise applications > Consent and permissions
```

That kind of detail saves readers time.

## 9. Open A Pull Request

When your article is ready, open a pull request from your fork.

GitHub Actions will automatically check things like:

- article structure
- author profile
- image paths
- alt text
- file sizes

Sometimes the article can be accepted quickly. Sometimes maintainers may ask for a clearer screenshot, safer sample data, better formatting, or more context.

That is normal. The goal is not to make contribution difficult. The goal is to make the final article useful, safe, and easy to publish.

## 10. What Happens After Review

After the pull request is accepted, maintainers import the article into the production website.

You do not need access to the private website repository.

You write in public.

Review happens in public.

Publishing stays controlled.

Author credit stays attached to the article.

That is the whole idea: make writing approachable, keep review predictable, and help more people share useful engineering knowledge with the Evotec community.
