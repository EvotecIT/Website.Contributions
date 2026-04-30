# Write for evotec.xyz

![Evotec website contributors quality card](assets/website-contributors-social.png)

Evotec is opening evotec.xyz to more practical voices from the community. If you have a useful PowerShell script, a migration lesson, a troubleshooting story, an odd edge case, or a clear technical note that could save someone else time, this repository is where that article can begin.

You do not need to arrive with a perfect post. Bring the useful idea, include the context, keep the images with the article, and open a pull request. Maintainers can help polish structure, wording, screenshots, metadata, and publishing details.

## Full Guide

The full contributor guide is currently kept inside this repository:

[How to contribute a post to evotec.xyz](posts/en/how-to-contribute-to-evotec-blog/index.md)

Once the article is published on evotec.xyz, this README can link to the live blog post instead.

![Contribution workflow from idea to pull request, review, and publishing](posts/en/how-to-contribute-to-evotec-blog/images/contribution-workflow.webp)

## How It Works

1. Fork this repository.
2. Create or update your author profile in `authors/<your-slug>.yml`.
3. Create your article folder in `posts/<language>/<article-slug>/`.
4. Put `index.md`, the cover image, and article images in that folder.
5. Write the article in Markdown.
6. Open a pull request.
7. Review feedback and update the article when needed.

Use `posts/en/...` for English articles and `posts/pl/...` for Polish articles.

## Article Folder

Keep every article self-contained:

```text
posts/en/my-article-slug/
  index.md
  cover.webp
  images/
    screenshot-01.webp
    diagram-01.webp
```

This makes review easier and keeps images from being separated from the article they belong to.

Accepted local image formats are PNG, JPG, JPEG, WEBP, and GIF. WEBP is preferred for covers and most screenshots when it keeps the detail readable.

## Repository Structure

```text
.github/
  workflows/
authors/
  images/
  your-name.yml
posts/
  en/
    how-to-contribute-to-evotec-blog/
      index.md
      cover.webp
      images/
templates/
  author.yml
  post/index.md
```

## Validation

Pull requests are checked automatically. Validation looks at the article structure, author profile, image paths, alt text, supported image formats, and file sizes.

Author photos are optional. If you add one, keep it under `authors/images/` and reference it from the author profile with a local path such as `avatar: "./images/your-name.webp"`.

If validation fails, read the message, update the article, and push again. The goal is to make publishing predictable, not to make contributing difficult.
