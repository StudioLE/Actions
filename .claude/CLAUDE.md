# Actions Repository

Reusable GitHub Actions workflows for CI/CD automation (Rust and .NET projects).

## Project Structure

All workflows live in `.github/workflows/`. Key categories:

- **Orchestrators**: `ci-cd.yml` (.NET), `ci-cd-rust.yml`, `ci-cd-crates.yml`, `ci-cd-bevy.yml`, `ci-cd-dioxus-web.yml`
- **Build**: `cargo-build.yml`, `docker-build.yml`
- **Release**: `github-release.yml`, `brew-release.yml`, `git-tag.yml`, `surveyor-release.yml`
- **Publish**: `push-to-nuget.yml`, `push-to-repo.yml`, `push-to-s3.yml`, `push-to-github-releases.yml`

## Critical Context

- **Main branch**: `v7` (many repos depend on these workflows - breaking changes need new version branch)
- **Reusable workflows**: Most use `workflow_call` trigger, called via `uses:` with `secrets: inherit`
- **Artifacts**: Named `{binary}-{version}-{platform}`, use `actions/upload-artifact@v4`/`download-artifact@v4`
- **Releases**: All use GitHub CLI (`gh`) for release operations

## Coding Standards

### Bash in Workflows
- `do` and `then` on new lines (not same line as `for`/`if`)
- Split long scripts into multiple workflow steps:
  - Step names serve as documentation (no verbose comments needed)
  - Each step should do one clear thing
  - The step name describes what the step does

### Behavior
- Never undo user changes without asking
- Be direct - disagree when you have better information
- No sycophancy or excessive validation

