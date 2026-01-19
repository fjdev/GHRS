# GitHub Repository Sync (GHRS)

[![Version](https://img.shields.io/badge/version-1.0.0-green.svg)](https://github.com/fjdev/ghrs)
[![Python 3.7+](https://img.shields.io/badge/python-3.7+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A professional tool for syncing GitHub repositories (particularly Terraform modules) with version tracking and automated cleanup.

## ✨ Features

- **Version Tracking**: Automatic version detection with `@version` suffix (e.g., `module@v1.0.0`)
- **Multiple Sync Modes**: Release-based or Git mirror syncing
- **Automated Cleanup**: Keeps only essential Terraform files (configurable)
- **Flexible Authentication**: GitHub App or Personal Access Token
- **Organization Support**: Global organization setting with per-module override
- **Smart Versioning**: Preserves all versions side-by-side for easy rollback
- **Production Ready**: Proper logging, error handling, and OOP design
- **Minimal Dependencies**: Core uses Python standard library (PyJWT optional for GitHub App)

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
# Sync modules from configuration
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

## 📋 Requirements

- Python 3.7+
- Git CLI
- GitHub credentials (optional, for private repos or to avoid rate limits):
  - GitHub App credentials, OR
  - Personal Access Token
- Optional: `PyJWT` for GitHub App authentication (`pip install PyJWT`)

## 🔐 Authentication

GHRS supports two authentication methods:

### 1. GitHub App (Recommended for Organizations)

Set environment variables:

```bash
export GITHUB_APP_ID="123456"
export GITHUB_APP_INSTALLATION_ID="12345678"
export GITHUB_APP_PRIVATE_KEY_PATH="/path/to/private-key.pem"
# Or use inline key:
export GITHUB_APP_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\n..."
```

### 2. Personal Access Token

```bash
export GITHUB_TOKEN="ghp_yourtoken"
# Or:
export GH_TOKEN="ghp_yourtoken"
```

### No Authentication

GHRS works without authentication for public repositories, but may hit GitHub API rate limits.

## 📊 Configuration File

### modules.json Structure

```json
{
  "organization": "myorg",
  "destination_root": "modules",
  "modules": [
    "simple-module-name",
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

Resolves to: `{organization}/module-name`

#### Full Format

```json
{
  "name": "module-name",
  "repo": "owner/repository",
  "mode": "mirror",
  "ref": "main",
  "module_dir": "."
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | string | (required) | Local name for the module |
| `repo` | string | (required) | GitHub repository (`owner/repo`) |
| `mode` | string | `"mirror"` | Sync mode: `"mirror"` or `"release"` |
| `ref` | string | `null` | Git branch/tag (mirror mode only) |
| `tag` | string | `null` | Release tag (release mode, `null` = latest) |
| `asset_name` | string | `null` | Specific release asset name |
| `module_dir` | string | `"."` | Subdirectory within repo to extract |

## 🔧 Sync Modes

### Mirror Mode (Default)

Clones the repository and extracts the latest commit:

```json
{
  "name": "my-module",
  "repo": "myorg/terraform-module",
  "mode": "mirror",
  "ref": "main"
}
```

**Version detection**: Uses `git describe --tags --always`

**Use cases**:
- Development/testing with latest code
- Modules without formal releases
- Custom branches or commits

### Release Mode

Downloads from GitHub releases:

```json
{
  "name": "my-module",
  "repo": "myorg/terraform-module",
  "mode": "release",
  "tag": "v1.0.0"
}
```

**Version detection**: Uses release tag name

**Use cases**:
- Production deployments
- Stable, tagged versions
- Reproducible builds

**Note**: Omit `"tag"` to automatically sync the latest release.

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

### Mixed Organization Sources

```json
{
  "organization": "myorg",
  "modules": [
    "my-module",
    "anotherorg/external-module",
    {
      "name": "custom",
      "repo": "thirdorg/special-module",
      "ref": "develop"
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
- Configure authentication (GitHub App or PAT)
- Authenticated requests have 5,000/hour limit

**Git Clone Failed**

- Ensure Git is installed: `git --version`
- Check network connectivity to GitHub
- Verify repository URL and access permissions

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

- **Fast Cloning**: Uses `--depth 1` for shallow clones
- **Minimal Downloads**: Only fetches what's needed
- **Parallel Safe**: Can sync multiple modules (run multiple instances)
- **Incremental**: Only downloads new versions on subsequent runs

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

### Example 3: Mixed Modes

```json
{
  "organization": "myorg",
  "modules": [
    {
      "name": "prod-module",
      "repo": "myorg/terraform-module",
      "mode": "release",
      "tag": "v1.0.0"
    },
    {
      "name": "dev-module",
      "repo": "myorg/terraform-module",
      "mode": "mirror",
      "ref": "develop"
    }
  ]
}
```

## 🏗️ Architecture

GHRS follows professional software engineering patterns:

- **Strategy Pattern**: Pluggable sync strategies (Release, Mirror)
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
