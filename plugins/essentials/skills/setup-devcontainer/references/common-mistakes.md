# Common Mistakes

Avoid these mistakes when adapting the setup-devcontainer skill for GitHub and GitHub Copilot.

## Don't do this

- Do **not** copy unrelated CI runner logic into GitHub Actions wording.
- Do **not** reference external CI runner tooling or unrelated assistant namespaces.
- Do **not** promise unsupported GitHub Copilot install or auth paths.
- Do **not** enable the firewall by default for normal development.
- Do **not** guess when repository evidence is missing.

## What to do instead

- Use GitHub Actions, repository manifests, and project config files as the evidence base.
- Only describe install or auth steps that are supported by the repo and its documented tooling.
- Treat the firewall as an opt-in hardening path, not a default requirement.
- If the repo does not clearly signal a tool or workflow, leave it out rather than inventing one.
