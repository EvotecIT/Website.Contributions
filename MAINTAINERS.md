# Maintainer Notes

This repository is the public intake queue for blog posts. Keep contributors here and import accepted content into the private website repository.

## Recommended GitHub Settings

- Keep the repository public.
- Protect `main`.
- Require pull requests before merging.
- Require the `Validate Contributions` workflow.
- Use squash merge for contributor PRs.
- Keep direct pushes limited to maintainers.

## Review Flow

1. Review the PR text, images, and author profile.
2. Confirm validation passes.
3. Merge the PR into `main`.
4. Pull this repository locally.
5. Import accepted content from the private website checkout:

```powershell
.\build\Import-Contributions.ps1 -ContributionRoot C:\Support\GitHub\Website.Contributions -Publish
```

Use `-Force` only when intentionally replacing a previously imported post.
