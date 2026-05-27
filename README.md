# Cubed Lithops Runtime Builder

A GitHub template repository for building and deploying [Lithops](https://lithops-cloud.github.io/) Lambda runtimes for [Cubed](https://github.com/cubed-dev/cubed) via CI — no local Docker required.

When you push changes to the `Dockerfile`, GitHub Actions builds a Docker image and deploys it as a Lambda container runtime named `cubed-runtime`.

## Prerequisites

- An AWS account
- A GitHub account
- The [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

## Setup

### 1. Create this repo from the template

Click **Use this template** → **Create a new repository**. You can name it anything you like — `cubed-lithops-runtime-builder` is a good default.

Then clone it to your local machine:

```bash
git clone https://github.com/YOUR_ORG/YOUR_REPO.git
cd YOUR_REPO
```

### 2. Bootstrap AWS

Run this once from your local machine. Replace `YOUR_ORG` with your GitHub username or organisation (e.g. `octocat`) and `YOUR_REPO` with the name you gave this repository.

```bash
aws cloudformation deploy \
  --template-file cloudformation/github-oidc-role.yml \
  --stack-name cubed-lithops-github-actions \
  --parameter-overrides GitHubOrg=YOUR_ORG GitHubRepo=YOUR_REPO \
  --capabilities CAPABILITY_NAMED_IAM
```

> **Already have a GitHub OIDC provider?** Use `--parameter-overrides GitHubOrg=YOUR_ORG GitHubRepo=YOUR_REPO CreateOIDCProvider=false` to skip creating one.

This creates a GitHub OIDC identity provider (if one doesn't already exist), an IAM role for GitHub Actions, and an IAM role for Lambda to assume when running functions. When it completes, retrieve both ARNs:

```bash
aws cloudformation describe-stacks \
  --stack-name cubed-lithops-github-actions \
  --query 'Stacks[0].Outputs' \
  --output table \
  --no-cli-pager
```

### 3. Add the secret

In your repo: **Settings → Secrets and variables → Actions → New repository secret**

| Name | Value |
|------|-------|
| `AWS_ROLE_ARN` | `GitHubActionsRoleArn` from the previous step |

### 4. Edit `.lithops/config`

Replace the placeholder values:

```yaml
aws:
    region: us-east-1                                         # your AWS region

aws_lambda:
    execution_role: arn:aws:iam::...   # LambdaExecutionRoleArn from the previous step
    user_id: AROAXXXXXXXXXXXXXXXXX     # GitHubActionsRoleId from the previous step
```

### 5. Add your dependencies

Edit the `Dockerfile` to add extra packages (there is a clearly marked section near the bottom), then commit and push with git — the CI pipeline builds and deploys the `cubed-runtime` Lambda runtime automatically.

## Security note

The `.lithops/config` you commit contains your AWS account ID (inside the `execution_role` ARN). This is also visible in CI build logs. AWS [does not consider the account ID a secret](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-identifiers.html), but as a precaution you may want to keep your repository private.

## Manual trigger

You can also trigger a build from **Actions → Build and Deploy Lithops Runtime → Run workflow**.

## Customisation

### Changing the runtime name

The runtime is named `cubed-runtime` by default. To use a different name, edit the `RUNTIME_NAME` env var at the top of `.github/workflows/build-runtime.yml`:

```yaml
env:
  RUNTIME_NAME: 'my-runtime-name'
```

Remember to also update the `runtime` key in your local Lithops config if you change this.

### Using the latest Cubed from GitHub

To use the latest development version of Cubed from the `main` branch instead of the PyPI release, replace the `cubed` line in the `Dockerfile`:

```dockerfile
RUN pip install \
        'git+https://github.com/cubed-dev/cubed.git#egg=cubed' \
        obstore
```
