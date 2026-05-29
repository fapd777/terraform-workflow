# terraform-workflow

A reusable GitHub Actions workflow that runs `terraform plan` with AWS STS credentials and optionally applies changes after environment approval. Call it from any repository to standardize how Terraform planning and applying is executed across your infrastructure projects.

## Workflow

**File:** `.github/workflows/terraform-plan.yml`

**Trigger:** `workflow_call` — designed to be called from other workflows.

### What it does

1. Checks out the calling repository
2. Masks AWS credentials to prevent them from appearing in logs
3. Sets `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_SESSION_TOKEN` as environment variables
4. Installs the specified Terraform version
5. Runs `terraform init -backend-config=./init-tfvars/<tfvars_file>`
6. Runs `terraform validate`
7. Runs `terraform plan -no-color -var-file "./apply-tfvars/<tfvars_file>"`
8. On pull requests, posts the plan output as a PR comment with a link to approve apply
9. On pull requests when plan succeeds, waits for environment approval, then runs `terraform apply -no-color -auto-approve`

Apply only runs on `pull_request` events when the plan step succeeds. Manual runs (`workflow_dispatch`) post the plan to the job summary only.

### Apply approval flow

After a successful plan on a pull request, the workflow posts a comment and pauses at the **Terraform apply** step until a required reviewer approves the configured GitHub Environment (default: `terraform-apply`). Reviewers approve from the PR **Checks** tab by clicking **Review deployments**.

Callers must create the environment in their repository before apply can run:

1. Go to **Settings → Environments → New environment**
2. Name it `terraform-apply` (or pass a custom name via the `apply_environment` input)
3. Enable **Required reviewers** and add users or teams allowed to approve applies
4. Optionally restrict deployment branches to protected branches only

### Inputs

| Name | Required | Type | Description |
|------|----------|------|-------------|
| `terraform_version` | Yes | string | Terraform version to install (e.g. `1.6.0`) |
| `tfvars_file` | Yes | string | Filename of the tfvars file inside the `./apply-tfvars/` directory |
| `apply_environment` | No | string | GitHub Environment name that gates `terraform apply` (default: `terraform-apply`) |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `aws_sts_credentials_json` | Yes | JSON output from `aws sts get-session-token` containing `Credentials.AccessKeyId`, `Credentials.SecretAccessKey`, and `Credentials.SessionToken` |
| `gh_pr_token` | Yes | GitHub PAT with read and write access to pull requests, used to post plan comments |

## Usage

In your calling repository, create a workflow that references this one:

```yaml
name: "Run Terraform plan"

on:
  workflow_dispatch:
  pull_request:

jobs:
  read-terraform-config:
    name: "Read Terraform configuration"
    runs-on: ubuntu-latest
    outputs:
      terraform_version: ${{ steps.tf_version.outputs.terraform_version }}
      tfvars_file: ${{ steps.tfvars_file.outputs.tfvars_file }}
    steps:
      - name: Checkout caller repository
        uses: actions/checkout@v6

      - name: Read .terraform-version
        id: tf_version
        run: |
          echo "terraform_version=$(tr -d '[:space:]' < .terraform-version)" >> "$GITHUB_OUTPUT"

      - name: Read tfvars file name
        id: tfvars_file
        run: |
          tfvars_file=$(find ./apply-tfvars -maxdepth 1 -type f -name '*.tfvars' -printf '%f\n')

          if [ -z "$tfvars_file" ]; then
            echo "No .tfvars file found in ./apply-tfvars" >&2
            exit 1
          fi

          if [ "$(printf '%s\n' "$tfvars_file" | wc -l)" -ne 1 ]; then
            echo "More than one .tfvars file found in ./apply-tfvars" >&2
            exit 1
          fi

          echo "tfvars_file=$tfvars_file" >> "$GITHUB_OUTPUT"

  call-terraform-plan:
    needs: read-terraform-config
    uses: fapd777/terraform-workflow/.github/workflows/terraform-plan.yml@20260526-1030
    with:
      terraform_version: ${{ needs.read-terraform-config.outputs.terraform_version }}
      tfvars_file: ${{ needs.read-terraform-config.outputs.tfvars_file }}
      apply_environment: terraform-apply  # optional; this is the default
    secrets:
      aws_sts_credentials_json: ${{ secrets.AWS_STS_CREDENTIALS_JSON }}
      gh_pr_token: ${{ secrets.GH_PR_TOKEN }}
```

The `AWS_STS_CREDENTIALS_JSON` secret should be the raw JSON output from a command like:

```bash
aws sts get-session-token --no-cli-pager --duration-seconds 3600
```

In your calling repository:

Store that output as a repository or organization secret named `AWS_STS_CREDENTIALS_JSON` 

The `GH_PR_TOKEN` secret should be a GitHub Actions Personal Access Token (PAT) with read and write permissions to pull requests.

In your calling repository:

Store the GHA PAT as a repository or organization secret named `GH_PR_TOKEN` 

It must have a `./apply-tfvars/` directory containing the tfvars file referenced by the `tfvars_file` input.
