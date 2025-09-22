# Terraform GitHub Actions

A collection of GitHub Actions for Terraform and Terragrunt workflows with plan artifact management and PR commenting.

## Actions

### install-dependencies

Installs and caches Terraform, Terragrunt, and optional TF Plan Commenter binaries.

**Inputs:**
- `terraform-version` (optional): Terraform version to install (default: `1.9.8`)
- `terragrunt-version` (optional): Terragrunt version to install (default: `0.83.2`)
- `tf-commenter-version` (optional): TF Plan Commenter version to install (default: `1.1.0`). Set to `none` to skip installation.

**Outputs:**
- `cache-key`: Cache key used for the installed binaries
- `cache-hit`: Whether cache was hit

**Usage:**
```yaml
- uses: ./install-dependencies
  with:
    terraform-version: '1.9.8'
    terragrunt-version: '0.83.2'
    tf-commenter-version: '1.0.0'
```

### upload-tfplan

Converts binary Terraform plans to JSON and uploads both binary and JSON files as artifacts.

**Inputs:**
- `working-directory` (required): Working directory to execute the action from
- `retention-days` (optional): Number of days to retain the artifact (default: `2`)
- `binary-upload` (optional): Whether to upload the binary tfplan files along with JSON (default: `true`). Set to `false` when only using for PR comments to save storage space.

**Outputs:**
- `artifact-name`: Name of the uploaded artifact

**Usage:**
```yaml
- uses: ./upload-tfplan
  with:
    working-directory: 'infrastructure/prod'
    retention-days: 5
    binary-upload: 'false'  # Skip binary upload for PR comments only
```

### download-tfplan

Downloads all tfplan artifacts from the current workflow run. **Use only in merge workflows** - PR workflows should use `generate-pr-comment` which handles downloads internally.

**Inputs:**
- `github-token` (optional): GitHub token for API access (default: `${{ github.token }}`)

**Usage:**
```yaml
- uses: ./download-tfplan
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

### generate-pr-comment

Downloads tfplan artifacts and generates PR comments using go-tfplan-commenter. Handles artifact downloads internally.

**Inputs:**
- `artifacts-path` (optional): Path where tfplan artifacts are downloaded (default: `all-tfplans`)
- `output-file` (optional): Output file for the generated comment (default: `comment.md`)
- `post-comment` (optional): Whether to post the comment to the PR (default: `true`)
- `github-token` (optional): GitHub token for posting comments (default: `${{ github.token }}`)

**Outputs:**
- `comment-file`: Path to the generated comment file

**Usage:**
```yaml
- uses: ./generate-pr-comment
  with:
    output-file: 'terraform-comment.md'
    post-comment: 'true'
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

## Usage Patterns

### 1. Basic Installation Only
Use only `install-dependencies` to install and cache Terraform/Terragrunt binaries:

```yaml
- uses: ./install-dependencies
```

### 2. PR Comments Workflow
For PR plan review with comments (downloads handled internally):

```yaml
- uses: ./install-dependencies
  with:
    tf-commenter-version: '1.1.0'

- name: Run Terragrunt Plan
  run: terragrunt run-all plan -out=tfplan.tfplan

- uses: ./upload-tfplan
  with:
    working-directory: '.'
    binary-upload: 'false'  # JSON only for comments

- uses: ./generate-pr-comment  # Downloads artifacts internally
```

### 3. Full Workflow with Apply
For complete workflow where binary plans are used for apply on merge:

**PR Workflow:**
```yaml
- uses: ./install-dependencies
- name: Plan
  run: terragrunt run-all plan -out=tfplan.tfplan
- uses: ./upload-tfplan
  with:
    working-directory: '.'
    binary-upload: 'true'  # Keep binaries for apply
- uses: ./generate-pr-comment  # Downloads artifacts internally
```

**Merge Workflow:**
```yaml
- uses: ./install-dependencies
- uses: ./download-tfplan  # Only used in merge workflows
- name: Apply
  run: terragrunt run-all apply tfplan.tfplan
```

## Features

- **Caching**: Binaries are cached to speed up subsequent runs
- **Cross-platform**: Uses `$RUNNER_TEMP` for proper temporary file handling
- **Artifact Management**: Sanitized artifact names for better readability
- **PR Integration**: Automatic PR commenting with plan summaries
- **Flexible Storage**: Optional binary upload to optimize storage usage
- **Workflow Separation**: Clear distinction between PR and merge workflows
