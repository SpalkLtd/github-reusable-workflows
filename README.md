# github-reusable-workflows

Reusable GitHub Workflows for the Spalk Organisation.

## Workflow families

### ECR workflows (Docker/ECS services)

| Workflow                         | Purpose                                         | AWS Credentials          | Code Checkout |
| -------------------------------- | ----------------------------------------------- | ------------------------ | ------------- |
| `build-ecr-image.yml`            | Build Docker image, upload as artifact          | ECR pull only (optional) | Yes           |
| `push-ecr-image.yml`             | Push artifact to ECR                            | ECR push                 | No            |
| `create-multi-arch-manifest.yml` | Create multi-arch manifest from per-arch images | ECR push                 | No            |
| `promote-to-dev.yml`             | Retag image as `:latest` in dev                 | ECR push (dev only)      | No            |
| `cleanup-ecr-images.yml`         | Delete PR images from ECR                       | ECR delete               | No            |

### Lambda workflows (Go Lambda functions)

| Workflow              | Purpose                                                | AWS Credentials    | Code Checkout |
| --------------------- | ------------------------------------------------------ | ------------------ | ------------- |
| `build-lambda.yml`    | Build Go binary, upload as artifact                    | None               | Yes           |
| `deploy-lambda.yml`   | Deploy binary to Lambda, publish version, create alias | S3 + Lambda deploy | No            |
| `activate-lambda.yml` | Update `current` alias (dev only)                      | Lambda UpdateAlias | No            |

The Lambda workflows enforce **artefact/activation separation**: `deploy-lambda.yml` can create new Lambda versions but cannot make them live. Only `activate-lambda.yml` can update the `current` alias, and its IAM trust policy is restricted to dev. Production activation is manual. See the [CI/CD Auth & Security docs](../../docs/infra/ci-cd-auth-security.md) for details.

### Other workflows

| Workflow                          | Purpose                                          |
| --------------------------------- | ------------------------------------------------ |
| `tagandrelease.yml`               | Create GitHub Release from "cut version" commits |
| `changedonlychangelog.yml`        | Detect changelog-only PRs to skip CI             |
| `append-testing-sheet-entry.yml`  | Add PR to QA testing Google Sheet                |
| `set-requires-testing-status.yml` | Update testing sheet status on merge             |

## Example usage

```yaml
jobs:
  tag_and_release:
    uses: SpalkLtd/github-reusable-workflows/.github/workflows/tagandrelease.yml@main
```

## Authorising reusable workflows in AWS IAM.

When setting up a new repository, you want to lock down the IAM role used.
This will require customising the Github OIDC token to include details about
what it is going and updating the IAM policy to allow this.

### Github OIDC Token

You can get details about the OIDC token running this command:

```bash
gh api \
    --method GET \
    -H "Accept: application/vnd.github+json" \
    -H "X-GitHub-Api-Version: 2022-11-28" \
    /repos/SpalkLtd/{repo_name}/actions/oidc/customization/sub
```

When `aws-actions/configure-aws-credentials` is run, it will send an OIDC token to AWS,
we will want to include the `job_workflow_ref` `sub` claim of the token.

This can be done with the following command:

```bash
gh api \
    --method PUT \
    -H "Accept: application/vnd.github+json" \
    -H "X-GitHub-Api-Version: 2022-11-28" \
    /repos/SpalkLtd/{repo_name}/actions/oidc/customization/sub \
    -F "use_default=false" \
    -f "include_claim_keys[]=repo" \
    -f "include_claim_keys[]=job_workflow_ref"
```

This will result in a token that looks like this:

```
repo:SpalkLtd/synchroniser:job_workflow_ref:SpalkLtd/github-reusable-workflows/.github/workflows/build-and-deploy-ecr-image.ml@f4cfdf77ca0470aabea01d44d58e11bd954155ce
```

### AWS IAM

Your IAM policy should then be updated to require this sub claim.

For example:

```hcl
data "aws_iam_policy_document" "spalk_ffmpeg_github_assume_role_policy_cd" {
statement {
    principals {
    type        = "Federated"
    identifiers = [var.github_iam_oidc_provider_arn]
    }
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]

    condition {
    test     = "StringEquals"
    variable = "token.actions.githubusercontent.com:aud"
    values   = ["sts.amazonaws.com"]
    }
    condition {
    test     = "StringLike"
    variable = "token.actions.githubusercontent.com:sub"
    values   = [
        "repo:SpalkLtd/spalk-ffmpeg:job_workflow_ref:SpalkLtd/github-reusable-workflows/.github/workflows/uild-and-deploy-ecr-image.yml@f4cfdf77ca0470aabea01d44d58e11bd954155ce"
    ]
    }
}
}
```
