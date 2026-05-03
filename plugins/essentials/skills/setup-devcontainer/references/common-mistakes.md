# Common Mistakes

Avoid these mistakes when adapting the setup-devcontainer skill for GitHub and GitHub Copilot.

## Don't do this

- Do **not** transplant the source skill's forge-specific runner or executor discovery heuristics into GitHub Actions wording. In this adaptation, rely on repository evidence such as workflow jobs, `runs-on`, `container`, `services`, and explicit tool invocations.
- Do **not** reuse the source skill's legacy assistant-specific CLI names, hidden home-directory mount patterns, or config namespace wording that were replaced during the GitHub + GitHub Copilot adaptation.
- Do **not** describe GitHub Copilot as its own in-container workflow surface, installer, or secret-sharing channel. Treat it as IDE/auth compatibility layered on top of repository evidence and supported editor attach flows.
- Do **not** promise unsupported GitHub Copilot install or auth paths.
- Do **not** reach for container-level network restrictions when the real requirement is controlling what Copilot CLI can access.
- Do **not** guess when repository evidence is missing.

## What to do instead

- Use GitHub Actions, repository manifests, and project config files as the evidence base.
- Map CI behavior from what the repository actually declares in workflows and scripts, not from source-skill carryovers.
- Keep assistant/editor guidance limited to supported IDE attach, authentication, and mount behavior already evidenced by the repo or chosen host.
- Only describe install or auth steps that are supported by the repo and its documented tooling.
- Use Copilot CLI tool/path/URL permissions as the default control surface for agent behavior.
- If the repo does not clearly signal a tool or workflow, leave it out rather than inventing one.
