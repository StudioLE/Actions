# Actions Repository - Claude Context

## Project Overview
Reusable GitHub Actions workflows for automating CI/CD pipelines for Rust and .NET projects.

### Main Workflow Types
- **`.NET`**: Full CI/CD with tests, docs, NuGet publishing, Docker
- **`Rust`**: General-purpose Rust CI/CD for fullstack apps, CLI tools, and libraries
- **`Bevy`** (Rust): Game engine builds with WebAssembly support
- **`Dioxus Web`** (Rust): WebAssembly-only builds

## Key Workflows

### Core Orchestrators
- `ci-cd.yml` - .NET master workflow
- `ci-cd-rust.yml` - General-purpose Rust projects (CLI tools, fullstack apps, libraries)
- `ci-cd-bevy.yml` - Bevy Rust projects
- `ci-cd-dioxus-web.yml` - Dioxus WebAssembly
- `ci-cd-dioxus-fullstack.yml` - Wrapper for `ci-cd-rust.yml` (backward compatibility)

### Reusable Components
- `surveyor-release.yml` - Version detection and release notes
- `cargo-build.yml` - Rust build matrix
- `github-release.yml` - Create GitHub releases with assets
- `brew-release.yml` - Homebrew release automation (NEW)
- `git-tag.yml` - Git tag creation
- `docker-build.yml` / `docker-push.yml` - Container workflows

### Publishing
- `push-to-nuget.yml` - NuGet.org publishing
- `push-to-repo.yml` - Deploy artifacts to target repositories
  - Optional GPG commit signing (automatically enabled when credentials provided)
  - Requires `sign: true` input and `GPG_KEY_ID`, `GPG_PRIVATE_KEY` secrets to enable signing
  - Keys must be in armored format with no passphrase
- `push-to-s3.yml` - S3 deployments
- `push-to-github-releases.yml` - Upload release artifacts

### Testing
- `test-push-to-repo.yml` - Validates push-to-repo.yml functionality
  - Triggers on all branch commits and workflow_dispatch
  - Pushes test files to dedicated test repository (StudioLE/ActionsTests)
  - Verifies GPG signing and commit correctness
  - Requires test secrets: `TESTS_REPO_TOKEN`, `GPG_PRIVATE_KEY`, `GPG_KEY_ID`

## Common Tasks

### Testing Workflow Changes
```bash
# Check workflow syntax
gh workflow view <workflow-name>

# List all workflows
gh workflow list

# Manually trigger a workflow
gh workflow run <workflow-name>
```

### Working with Branches

Many other repositories depend on these workflows. Therefore breaking changes will require a new version branch.

Current main branch: `v7`

## Architecture Notes

### Matrix Strategy
Workflows use JSON matrices for multi-platform builds:
- OS: ubuntu-latest, windows-latest, macos-latest
- Rust targets: x86_64, aarch64, wasm32
- Build types: release, debug

### Artifact Flow
1. Build artifacts uploaded with naming: `{binary}-{version}-{platform}`
2. Artifacts downloaded in release workflows
3. Assets uploaded to GitHub releases via `gh release upload`

### Version Management
- Surveyor determines version from git tags/commits
- Versions follow semver: `v{major}.{minor}.{patch}`
- Supports both releases and prereleases

## Quick Reference

### Find specific workflow features
```bash
# Search for workflow inputs
grep -r "inputs:" .github/workflows/

# Find jobs that use specific actions
grep -r "uses: actions/" .github/workflows/

# Check matrix definitions
grep -A 10 "matrix:" .github/workflows/
```

### Workflow Dependencies
- Most workflows are `workflow_call` (reusable)
- Called from orchestrator workflows via `uses:`
- Inputs defined in `inputs:` section
- Secrets passed via `secrets: inherit`

## Coding Standards

### Bash Scripts
- **Control structures**: `do` and `then` keywords must be on new lines

  ```bash
  # Good
  for artifact in *.tar.xz
  do
    if [[ -f "$artifact" ]]
    then
      echo "$artifact"
    fi
  done

  # Bad
  for artifact in *.tar.xz; do
    if [[ -f "$artifact" ]]; then
      echo "$artifact"
    fi
  done
  ```

- **Script length**: Split long scripts into multiple workflow steps
  - Step names serve as documentation (no verbose comments needed)
  - Each step should do one clear thing
  - The step name describes what the step does

### Important: Respecting User Edits
- **NEVER undo user changes without asking first**
- If the user modifies something (removes a directory, changes structure, etc.), assume it's intentional
- Do not revert changes unless explicitly asked to do so
- When the system shows file modifications, take them into account and work with them

### Communication Style
- **Be direct and honest, not a sycophant**
- Don't say "You're absolutely right" or similar excessive validation
- If you have better information or know a better approach, speak up
- Mutual respect means being honest when you disagree
- Focus on technical accuracy, not validation

## Notes
- All workflows use GitHub CLI (`gh`) for releases
- Artifacts use `actions/upload-artifact@v4` and `actions/download-artifact@v4`
- Docker images support both prerelease and release tags
- WebAssembly builds compressed as `.tar.xz` before upload
- Homebrew formula generation only activates when `homebrew_tap` input is provided


## Homebrew Formula Generation

The `brew-release.yml` workflow includes automatic Homebrew formula generation:

### How It Works
1. Downloads artifacts from the build job
2. Loops through artifacts and calculates SHA256 checksums for each
3. Substitutes values into provided template string
4. Validates generated formula
5. Uploads formula as artifact
6. Uses `push-to-repo.yml` to commit and push formula to tap repository

### Required Inputs
- `version`: Release version
- `binary_name`: Name of the binary
- `brew_repo`: Tap repository (e.g., "username/homebrew-tap")
- `template`: Ruby formula template as multiline string

### Secret
- `brew_repo_token`: GitHub PAT with write access to tap repo (required if using Homebrew feature)

### Template Placeholders
**Basic placeholders:**
- `{{version}}` - Version number
- `{{binary_name}}` - Binary name

**SHA256 placeholders:**
- `{{sha256:target-triple}}` - SHA256 for specific target
- Example: `{{sha256:x86_64-apple-darwin}}` for macOS Intel
- Example: `{{sha256:aarch64-apple-darwin}}` for macOS ARM
- Example: `{{sha256:x86_64-unknown-linux-gnu}}` for Linux x64
- Target triple must match artifact filename pattern: `{binary}-{version}-{target}.tar.xz`

### Example Template

```ruby
class MountLuks < Formula
  desc "A simple CLI tool to unlock and mount a LUKS encrypted disk."
  homepage "https://github.com/StudioLE/mount-luks"
  license "GPL-3.0-only"
  version "{{version}}"

  on_linux do
    if Hardware::CPU.arm?
      url "https://github.com/StudioLE/mount-luks/releases/download/v{{version}}/{{binary_name}}-{{version}}-aarch64-unknown-linux-gnu.tar.xz"
      sha256 "{{sha256:aarch64-unknown-linux-gnu}}"
    else
      url "https://github.com/StudioLE/mount-luks/releases/download/v{{version}}/{{binary_name}}-{{version}}-x86_64-unknown-linux-gnu.tar.xz"
      sha256 "{{sha256:x86_64-unknown-linux-gnu}}"
    end
  end

  def install
    bin.install "{{binary_name}}"
  end

  test do
    system "#{bin}/{{binary_name}}", "--help"
  end
end
```
