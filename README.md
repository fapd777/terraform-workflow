# terraform-workflow

A reusable GitHub Actions workflow that runs `terraform plan` with AWS STS credentials. Call it from any repository to standardize how Terraform planning is executed across your infrastructure projects.

## Workflow

**File:** `.github/workflows/terraform-plan.yml`

**Trigger:** `workflow_call` — designed to be called from other workflows.

### What it does

1. Checks out the calling repository
2. Masks AWS credentials to prevent them from appearing in logs
3. Sets `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_SESSION_TOKEN` as environment variables
4. Installs the specified Terraform version
5. Runs `terraform init`
6. Runs `terraform validate`
7. Runs `terraform plan -var-file "./apply-tfvars/<tfvars_file>"`

### Inputs

| Name | Required | Type | Description |
|------|----------|------|-------------|
| `terraform_version` | Yes | string | Terraform version to install (e.g. `1.6.0`) |
| `tfvars_file` | Yes | string | Filename of the tfvars file inside the `./apply-tfvars/` directory |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `aws_sts_credentials_json` | Yes | JSON output from `aws sts get-session-token` containing `Credentials.AccessKeyId`, `Credentials.SecretAccessKey`, and `Credentials.SessionToken` |

## Usage

In your calling repository, create a workflow that references this one:

```yaml
name: "Run Terraform plan"

on:
  workflow_dispatch:
  # pull_request:

jobs:
  read-terraform-config:
    name: "Read Terraform configuration"
    runs-on: ubuntu-latest
    outputs:
      terraform_version: ${{ steps.tf_version.outputs.terraform_version }}
      tfvars_file: ${{ steps.tfvars_file.outputs.tfvars_file }}
    steps:
      - name: Checkout caller repository
        uses: actions/checkout@v4

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
    uses: fapd777/terraform-workflow/.github/workflows/terraform-plan.yml@main
    with:
      terraform_version: ${{ needs.read-terraform-config.outputs.terraform_version }}
      tfvars_file: ${{ needs.read-terraform-config.outputs.tfvars_file }}
    secrets:
      aws_sts_credentials_json: ${{ secrets.AWS_STS_CREDENTIALS_JSON }}
```

The `aws_sts_credentials_json` secret should be the raw JSON output from a command like:

```bash
aws sts get-session-token --no-cli-pager --duration-seconds 3600
```

In your calling repository:

Store that output as a repository or organization secret named `AWS_STS_CREDENTIALS_JSON` 

It must have a `./apply-tfvars/` directory containing the tfvars file referenced by the `tfvars_file` input.
