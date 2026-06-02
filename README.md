# terraform-workflow

A reusable GitHub Actions workflow that runs `terraform plan` with AWS STS credentials and applies changes from the saved plan after at least one repository reviewer approves the pull request. Call it from any repository to standardize how Terraform planning and applying is executed across your infrastructure projects.

## Workflow

**File:** `.github/workflows/terraform-plan.yml`

**Trigger:** `workflow_call` — designed to be called from other workflows.

### What it does

The workflow runs three jobs on pull requests:

1. **Terraform plan job** — checks out the calling repository, configures AWS credentials, installs Terraform, runs `init`, `validate`, and `plan`, uploads the saved plan as an artifact, and posts the plan to the pull request (when triggered by a `pull_request` event).
2. **Check PR approval** — verifies that at least one repository reviewer has submitted an `APPROVED` review on the pull request.
3. **Terraform apply job** — downloads the plan artifact from the plan job and runs `terraform apply tfplan`. This job runs only when the plan succeeded and the PR has at least one approved review.

For non-PR triggers (such as `workflow_dispatch`), only the plan job runs.

### Inputs

| Name | Required | Type | Description |
|------|----------|------|-------------|
| `terraform_version` | Yes | string | Terraform version to install (e.g. `1.6.0`) |
| `tfvars_file` | Yes | string | Filename of the tfvars file inside the `./apply-tfvars/` directory |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `aws_sts_credentials_json` | Yes | JSON output from `aws sts get-session-token` containing `Credentials.AccessKeyId`, `Credentials.SecretAccessKey`, and `Credentials.SessionToken` |
| `gh_pr_token` | Yes | GitHub PAT with read and write access to pull requests, used to post plan comments and check review status |

## Usage

In your calling repository, create a workflow that references this one:

```yaml
name: "Run Terraform plan"

on:
  workflow_dispatch:
  pull_request:
  pull_request_review:
    types: [submitted]

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
    if: github.event_name != 'pull_request_review' || github.event.review.state == 'approved'
    uses: fapd777/terraform-workflow/.github/workflows/terraform-plan.yml@20260526-1030
    with:
      terraform_version: ${{ needs.read-terraform-config.outputs.terraform_version }}
      tfvars_file: ${{ needs.read-terraform-config.outputs.tfvars_file }}
    secrets:
      aws_sts_credentials_json: ${{ secrets.AWS_STS_CREDENTIALS_JSON }}
      gh_pr_token: ${{ secrets.GH_PR_TOKEN }}
```

The `pull_request_review` trigger allows apply to run when a reviewer approves the PR without requiring a new push. The `if` condition on `call-terraform-plan` avoids re-running the workflow on non-approval review submissions.

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
