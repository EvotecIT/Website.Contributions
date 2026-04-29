# Evotec Website Contributions

This repository is the intake area for guest and community blog posts for Evotec websites.

Writers work in self-contained post folders under `posts/<language>/<year>/<slug>/`. Keep the Markdown post, cover image, and supporting screenshots together so review stays simple and images do not get lost.

Recommended visibility is public when posts are meant to be submitted by outside writers. Keep drafts that must stay private out of this repository until you are ready to share them with maintainers.

## Structure

```text
.github/
  ISSUE_TEMPLATE/
  workflows/validate.yml
authors/
  README.md
  your-name.yml
posts/
  README.md
  en/
    2026/
      example-post/
        index.md
        cover.jpg
        images/
          screenshot-01.png
templates/
  author.yml
  post/index.md
```

## Writer Flow

1. Fork this repository.
2. Copy `templates/author.yml` into `authors/<your-slug>.yml`.
3. Copy `templates/post/index.md` into `posts/<language>/<year>/<post-slug>/index.md`.
4. Add cover images and screenshots inside the same post folder.
5. Open a pull request.

## Validate Locally

From this repository:

```powershell
powerforge-web contributions validate --root .
```

The GitHub workflow validates pull requests automatically against the current PowerForge engine.

When a post is accepted, maintainers import it into the private website repository:

```powershell
powerforge-web contributions import --root C:\Support\GitHub\Website.Contributions --site-root C:\Support\GitHub\Website --publish
```

The import rewrites local image paths such as `./cover.jpg` and `./images/screenshot-01.png` into website asset paths.
