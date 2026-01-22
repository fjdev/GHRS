# GitHub Repository Sync (GHRS)

[![Python 3.7+](https://img.shields.io/badge/python-3.7+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A tool for syncing GitHub repositories (particularly Terraform modules) with version tracking and automated cleanup.

## ✨ Features

- **Version Tracking**: Automatic version detection with `@version` suffix (e.g., `module@v1.0.0`)
- **Release-Only Sync**: Always pulls published releases (latest by default) for deterministic builds
- **Automated Cleanup**: Keeps only essential Terraform files (configurable)
- **Parallel Sync**: Sync multiple modules concurrently (bounded worker pool)
- **Resilient Fetch**: Built-in retries and GitHub rate-limit backoff
- **Safe Extraction**: Guards against path traversal in archives
- **Flexible Authentication**: Token from `GITHUB_TOKEN`, `GH_TOKEN`, or `AZURE_DEVOPS_PAT`
- **Organization Support**: Global organization setting with per-module override
- **Smart Versioning**: Preserves all versions side-by-side for easy rollback
- **Production Ready**: Proper logging, error handling, and OOP design
- **Minimal Dependencies**: Core uses Python standard library only

## 🚀 Quick Start

### Installation

```bash
# Download or clone the repository
cd ghrs

# Make executable
chmod +x ghrs

# Ready to use!
```

### Basic Usage

```bash
# Sync modules from latest release (default)
./ghrs

# That's it! The tool reads modules.json and syncs everything
```

### Configuration

Create a `modules.json` file:

```json
{
  "organization": "myorg",
  "modules": [
    "terraform-module-name",
    "another-module"
  ]
}
```

Or use full repository paths (no organization field needed):

```json
{
  "modules": [
    "owner/repo-name",
    "anotherorg/another-repo"
  ]
}
```

Or specify versions directly with `@version` syntax:

```json
{
  "organization": "myorg",
  "modules": [
    "terraform-module-name@v1.0.0",
    "owner/another-module@v2.1.3"
  ]
}
```

## 📋 Requirements

- Python 3.7+
- Git CLI
- Token (optional, for private repos or to avoid rate limits): `GITHUB_TOKEN`, `GH_TOKEN`, or `AZURE_DEVOPS_PAT`
- No external Python dependencies

## 🔐 Authentication
GHRS reads a single token from environment variables (first match wins):

1. `GITHUB_TOKEN`
2. `GH_TOKEN`
3. `AZURE_DEVOPS_PAT` (fallback for environments that already expose this secret)

Example:

```bash
export GITHUB_TOKEN="ghp_yourtoken"
```

Without a token, public repositories still work but may hit GitHub API rate limits.

## 📊 Configuration File

### modules.json Structure

```json
{
  "organization": "myorg",
  "destination_root": "modules",
  "modules": [
    "simple-module-name",
    "another-module@v1.0.0",
    {
      "name": "custom-module",
      "repo": "owner/repo",
      "mode": "release",
      "tag": "v1.2.0",
      "module_dir": "."
    }
  ]
}
```

### Configuration Options

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `organization` | string | (required*) | Default GitHub organization for simple module names |
| `destination_root` | string | `"modules"` | Directory where modules are synced |
| `modules` | array | (required) | List of modules to sync |

*Required only if using simple module names (without `/`)

### Module Configuration

#### Simple Format

```json
"module-name"
```

Resolves to: `{organization}/module-name` (latest release)

**With version:**

```json
"module-name@v1.2.0"
```

Resolves to: `{organization}/module-name` at tag `v1.2.0`

**Full path with version:**

```json
"owner/repo-name@v1.2.0"
```

Resolves to: `owner/repo-name` at tag `v1.2.0`

#### Full Format

```json
{
  "name": "module-name",
  "repo": "owner/repository",
  "mode": "release",
  "module_dir": "."
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | string | (required) | Local name for the module |
| `repo` | string | (required) | GitHub repository (`owner/repo`) |
| `mode` | string | `"release"` | Sync mode (release-only) |
| `tag` | string | `null` | Release tag (release mode, `null` = latest) |
| `asset_name` | string | `null` | Specific release asset name |
| `module_dir` | string | `"."` | Subdirectory within repo to extract |

## 🔧 Sync Mode

### Release Mode (Only)

Downloads from GitHub releases (latest if version omitted) and unpacks the zipball into the local destination:

**Simple syntax (recommended):**
```json
"myorg/terraform-module@v1.0.0"
```

**Full format (for advanced options):**
```json
{
  "name": "my-module",
  "repo": "myorg/terraform-module",
  "mode": "release",
  "tag": "v1.0.0"
}
```

**Version detection**: Uses release tag name (or `latest` when unspecified)

**Use cases**:
- Production deployments
- Stable, tagged versions
- Reproducible builds

**Note**: Omit version/`"tag"` to automatically sync the latest release.

### Runtime Tuning (env vars)

| Variable | Default | Description |
|----------|---------|-------------|
| `GHRS_MAX_WORKERS` | `4` | Max parallel module sync workers |
| `GHRS_HTTP_TIMEOUT` | `20` | Seconds for network timeouts |
| `GHRS_RETRY_ATTEMPTS` | `3` | Total retry attempts for API/downloads |
| `GHRS_RETRY_BACKOFF_SECONDS` | `1` | Base backoff between retries |

## 📦 Output Structure

GHRS creates versioned directories:

```
modules/
├── terraform-module-example@v1.0.0/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── terraform.tf
│   └── README.md
├── terraform-module-example@v1.1.0/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── terraform.tf
│   └── README.md
└── another-module@abc1234/
    └── ...
```

### File Cleanup

GHRS automatically keeps only essential files:
- `main.tf`
- `variables.tf`
- `outputs.tf`
- `terraform.tf`
- `README.md`

All other files and directories are removed to keep modules clean.

## 🎯 Usage in Terraform

Reference modules using the versioned directory:

```hcl
module "resource_group" {
  source = "../modules/terraform-module-example@v1.0.0"
  
  # Module inputs
  location = "eastus"
  name     = "my-rg"
}
```

### Version Pinning Benefits

- **Explicit versions**: Clear which version you're using
- **Multiple versions**: Keep multiple versions for gradual migration
- **Easy rollback**: Switch back to older versions by changing the path
- **No symlinks needed**: Direct, explicit references

## 🔧 Advanced Usage

### Quick Examples

**Multiple versions, mixed formats:**
```json
{
  "organization": "myorg",
  "modules": [
    "simple-module@v1.0.0",
    "another-module",
    "external-org/special-module@v2.1.0"
  ]
}
```

### Mixed Organization Sources

```json
{
  "organization": "myorg",
  "modules": [
    "my-module",
    "anotherorg/external-module",
    {
      "name": "custom",
      "repo": "thirdorg/special-module"
    }
  ]
}
```

### Monorepo Modules

Extract a subdirectory from a repository:

```json
{
  "name": "networking",
  "repo": "myorg/terraform-modules",
  "module_dir": "modules/networking"
}
```

Or with a specific version:
```json
{
  "name": "networking",
  "repo": "myorg/terraform-modules",
  "tag": "v2.1.0",
  "module_dir": "modules/networking"
}
```

### Specific Release Assets

Download a specific release artifact:

```json
{
  "name": "my-module",
  "repo": "myorg/terraform-module",
  "mode": "release",
  "asset_name": "module-bundle.tar.gz"
}
```

## 🚀 CI/CD Integration

You can run GHRS in both Azure Pipelines and GitHub Actions. These examples assume:
- `modules.json` is in repo root
- `GITHUB_TOKEN` (or a PAT) is available as a secret with repo read access (write if you commit back)
- Optional: set `GHRS_VERSION` to pin the GHRS script version (defaults to `main` if omitted)

### CI Setup Checklist

- Provide `GITHUB_TOKEN`/PAT with `contents:read` (and `contents:write` if committing)
- Ensure `modules.json` exists in repo root
- Pin `GHRS_VERSION` to a released tag for reproducibility
- Adjust triggers (push paths, schedule) to match your repo
- If committing back, set git user.name/email in the pipeline

### Azure Pipelines

Add a pipeline like the sample in [sync.yml](sync.yml). Key pieces:

```yaml
variables:
  GHRS_VERSION: 'v1.2.0'
  GHRS_MAX_WORKERS: 4

steps:
  - checkout: self
    persistCredentials: true

  - task: UsePythonVersion@0
    inputs:
      versionSpec: '3.x'

  - script: |
      python -m pip install --upgrade pip
      curl -sSfL "https://raw.githubusercontent.com/fjdev/GHRS/$(GHRS_VERSION)/ghrs" -o ghrs
      chmod +x ghrs
    displayName: Prepare GHRS

  - script: ./ghrs
    displayName: Run GHRS
    env:
      GITHUB_TOKEN: $(GITHUB_TOKEN)
      GHRS_MAX_WORKERS: $(GHRS_MAX_WORKERS)

  - script: |
      git config --global user.email "azure-pipelines@users.noreply.github.com"
      git config --global user.name "Azure Pipelines"
      git add modules
      git commit -m "chore: sync terraform modules [skip ci]" || echo "No changes"
      git push origin HEAD:$(Build.SourceBranchName)
    displayName: Commit and push changes
```

Notes:
- If you want a scheduled sync, add a `schedules` block (see [sync.yml](sync.yml)).
- Ensure the service connection/token has `contents: write` if committing.
- `MODULES_DEST` can be adjusted; default output is `modules/`.

### GitHub Actions

Create `.github/workflows/ghrs.yml`:

```yaml
name: Sync Terraform Modules

on:
  workflow_dispatch:
  push:
    paths: ["modules.json"]
    branches: ["main"]
  schedule:
    - cron: "0 2 * * *"  # nightly

permissions:
  contents: write  # needed if you commit/push changes

env:
  GHRS_VERSION: v1.2.0
  GHRS_MAX_WORKERS: 4
  GHRS_HTTP_TIMEOUT: 20
  GHRS_RETRY_ATTEMPTS: 3
  GHRS_RETRY_BACKOFF_SECONDS: 1

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-python@v5
        with:
          python-version: '3.x'

      - name: Prepare GHRS
        run: |
          python -m pip install --upgrade pip
          curl -sSfL "https://raw.githubusercontent.com/fjdev/GHRS/${GHRS_VERSION}/ghrs" -o ghrs
          chmod +x ghrs

      - name: Run GHRS
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GHRS_MAX_WORKERS: ${{ env.GHRS_MAX_WORKERS }}
          GHRS_HTTP_TIMEOUT: ${{ env.GHRS_HTTP_TIMEOUT }}
          GHRS_RETRY_ATTEMPTS: ${{ env.GHRS_RETRY_ATTEMPTS }}
          GHRS_RETRY_BACKOFF_SECONDS: ${{ env.GHRS_RETRY_BACKOFF_SECONDS }}
        run: ./ghrs

      - name: Commit and push changes
        run: |
          git config --global user.email "github-actions@users.noreply.github.com"
          git config --global user.name "github-actions"
          git add modules
          git commit -m "chore: sync terraform modules [skip ci]" || echo "No changes"
          git push
```

Notes:
- The default `GITHUB_TOKEN` works for public repos; for private org-wide access, use a PAT in `secrets.GITHUB_TOKEN`.
- If you need a different destination, set `destination_root` in `modules.json` and adjust the `git add` path.
- The workflow watches `modules.json`; adjust triggers as needed.

## 🛠️ Troubleshooting

### Common Issues

**Authentication Failed**

```bash
# Verify GitHub CLI is logged in
gh auth status

# Or check your token
echo $GITHUB_TOKEN

# Run with debug logging (edit ghrs to enable DEBUG level)
```

**Module Not Found**

- Verify the repository exists and is accessible
- Check organization spelling in config
- For private repos, ensure authentication is configured
- Use full `owner/repo` format to bypass organization setting

**Rate Limiting (HTTP 403)**

- GitHub has rate limits for unauthenticated requests (60/hour)
- Set a token via `GITHUB_TOKEN`, `GH_TOKEN`, or `AZURE_DEVOPS_PAT`
- Authenticated requests have a 5,000/hour limit

**Download/Release Fetch Failed**

- Check network connectivity to GitHub
- Ensure authentication is set to avoid rate limits
- Verify the release tag/asset exists

**Wrong Files in Module**

- GHRS keeps only: `main.tf`, `variables.tf`, `outputs.tf`, `terraform.tf`, `README.md`
- All other files are automatically removed
- Check the `cleanup_unwanted_files()` function if you need to customize

### Error Messages

**"Module 'name' missing organization"**

You're using a simple module name without an organization field. Either:
1. Add `"organization": "yourorg"` to `modules.json`, OR
2. Use full format: `"owner/repo"`

**"Config not found: modules.json"**

- Ensure `modules.json` exists in the same directory as `ghrs`
- Check JSON syntax is valid

**"Command failed: git clone"**

- Git is not installed or not in PATH
- Repository doesn't exist or you don't have access
- Network issues with github.com

## ⚡ Performance

GHRS is designed for efficiency:

- **Parallel Downloads**: Bounded worker pool for multiple modules
- **Minimal Downloads**: Pulls release archives only
- **Retry & Backoff**: Built-in retries with rate-limit handling
- **Incremental**: Writes versioned module directories side-by-side

### Typical Sync Times

| Modules | Total Size | Time |
|---------|-----------|------|
| 1-5 | <10 MB | <30s |
| 5-20 | <50 MB | 1-2 min |
| 20-50 | <200 MB | 3-5 min |

**Note**: Times vary based on network speed and GitHub API response times.

## 📝 Examples

### Example 1: Simple Terraform Modules

```json
{
  "organization": "mycompany",
  "modules": [
    "terraform-aws-vpc",
    "terraform-aws-eks",
    "terraform-azure-aks"
  ]
}
```

Syncs to:
```
modules/
├── terraform-aws-vpc@v2.1.0/
├── terraform-aws-eks@v18.0.0/
└── terraform-azure-aks@v1.5.2/
```

### Example 2: Multi-Organization

```json
{
  "modules": [
    "hashicorp/terraform-aws-consul",
    "cloudposse/terraform-aws-components",
    "myorg/custom-module"
  ]
}
```

## 🏗️ Architecture

GHRS follows professional software engineering patterns:

- **Strategy Pattern**: Pluggable sync strategies (Release-only)
- **Provider Pattern**: Multiple authentication providers
- **Dataclasses**: Type-safe configuration and results
- **ABC Classes**: Clean interfaces and extensibility
- **Error Hierarchy**: Specific exception types for clear error handling
- **Separation of Concerns**: Downloaders, extractors, parsers separated
- **Configuration Driven**: All behavior controlled via JSON config

Based on patterns from the [ARA](https://github.com/fjdev/ara) tool.

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.

## 👤 Author

fjdev - [GitHub](https://github.com/fjdev)

## 🔗 Related Tools

- **ARA** (Azure Role Assignment Exporter) - Export Azure role assignments
- **TMVS** (Terraform Module Version Scanner) - Scan Terraform Cloud private registry modules
- **UTMS** (Universal Terraform Module Scanner) - Scan repositories for Terraform module usage

Part of the VCC (Version Control & Compliance) Toolkit.

---

GHRS v1.0.0 - Making Terraform module management simple and professional. 🚀
