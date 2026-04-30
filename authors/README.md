# Authors

Each contributor should have one YAML profile:

```text
authors/<author-slug>.yml
```

Use `templates/author.yml` as the starting point. The slug is what posts reference in front matter:

```yaml
authors:
  - your-name
```

Optional author photos are supported. Put the image under `authors/images/` and reference it from your profile:

```yaml
avatar: "./images/your-name.webp"
```

Use PNG, JPG, JPEG, WEBP, or GIF. A square image works best.
