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

| Workflow              | Purpose                                                | AWS Credentials         | Code Checkout |
| --------------------- | ------------------------------------------------------ | ----------------------- | ------------- |
| `build-lambda.yml`    | Build Go binary, upload as artifact                    | None                    | Yes           |
| `deploy-lambda.yml`   | Deploy binary to Lambda, publish version, create alias (and optionally re-point a moving alias via `moving-alias-name`) | S3 + Lambda deploy      | No            |
| `activate-lambda.yml` | Update `current` alias (dev only)                      | Lambda UpdateAlias      | No            |
| `cleanup-lambda.yml`  | Delete PR aliases, their versions, and optional ZIPs   | Lambda delete + S3 (opt.)| No            |

The Lambda workflows enforce **artefact/activation separation**: `deploy-lambda.yml` can create new Lambda versions but cannot make them live. Only `activate-lambda.yml` can update the `current` alias, and its IAM trust policy is restricted to dev. Production activation is manual. See the [CI/CD Auth & Security docs](../../docs/infra/ci-cd-auth-security.md) for details.

`cleanup-lambda.yml` is the Lambda equivalent of `cleanup-ecr-images.yml`: it tears down per-PR Lambda aliases created by `deploy-lambda.yml`. It refuses to operate on the protected `current` alias (or prefixes like `v`/`release` that could match release aliases). Inputs:

- `aws-role` (required) — IAM role to assume via OIDC.
- `aws-region` (required) — AWS region for the Lambda functions.
- `function-names` (required) — JSON array of Lambda function names (e.g. `["fn-a", "fn-b"]`).
- `alias-prefix` (required) — prefix for aliases to delete (e.g. `pr-42-`). Must not match `current`.
- `lambda-binaries-bucket` (optional) — when set, also deletes `lambda_${alias-prefix}*.zip` objects from this bucket. When unset, no S3 cleanup is performed.

### Other workflows

| Workflow                          | Purpose                                          |
| --------------------------------- | ------------------------------------------------ |
| `tagandrelease.yml`               | Create GitHub Release from "cut version" commits |
| `changedonlychangelog.yml`        | Detect changelog-only PRs to skip CI             |
| `append-testing-sheet-entry.yml`  | Add PR to QA testing Google Sheet                |
| `set-requires-testing-status.yml` | Update testing sheet status on merge             |

## Composite actions

| Action               | Purpose                                                          |
| -------------------- | ---------------------------------------------------------------- |
| `use-base-ci-tools`  | Replace a PR's copies of CI verdict tools with the base branch's |

`use-base-ci-tools` exists so a pull request cannot rewrite the code that
judges it. The consuming repo's validate workflows are resolved from a base ref,
but the scripts they execute come from `actions/checkout` — the PR head. This
step overwrites those copies with the base branch's before anything runs them.

It must live **outside** the repo it protects: a PR in the consuming repo can
edit any file in that repo, including a vendored copy of this action, so the
trust root only holds while the action is somewhere the PR cannot reach. Pin it
by SHA and never call it by local `./` path.

**Pinning the action is not sufficient on its own.** On `pull_request`, GitHub
runs the workflow file from the PR's own merge commit — a commit controlled by
someone without write access to the base repository. So a PR can delete this
step, edit its `paths:`, or append a later step that overwrites the files
again. The action file is beyond the PR's reach; the *call site* is not.

What closes that is the shape of the calling workflow. It must itself be
resolved from the base ref — `pull_request_target`, `merge_group` or `push` —
and because such a workflow runs with base-branch privileges against
PR-controlled content, it must earn that by doing nothing else: no build, no
dependency install, no PR code executed before this step. Restrict it to the
restore and the verdict, give it `permissions: contents: read`, and pass it no
secrets.

```yaml
- name: Use base-branch CI tools
  uses: SpalkLtd/github-reusable-workflows/use-base-ci-tools@<sha>
  with:
    paths: |
      tooling/ci/coverage-shard.go
      scripts/coverage-gate.sh
```

### Inputs

`paths` is a newline-separated list of repo-relative files or directories.
Directories are restored recursively; files the PR added under one are left
alone, tracked or not.

`ref` defaults to `main` and is a fixed **trust root**, deliberately not
`github.base_ref`. `base_ref` is chosen by the PR author, so a stacked PR could
open a branch carrying a neutered gate and then target it, supplying its own
judge. The cost is that a PR targeting `release/1.2` is judged by `main`'s
tools unless the caller overrides `ref`.

### Failure modes

It fails closed on a path missing from the base ref; an empty `paths`; a mode
that is not a regular file (symlinks and submodules are refused rather than
written out); a path that escapes the workspace, including via a symlinked path
component; a truncated or empty tree listing; and a working directory in which
none of the requested paths exist — which means the step ran before
`actions/checkout`, or against the wrong directory.

`.github/workflows/use-base-ci-tools-test.yml` exercises all of this on every
PR: git and API modes across files, directories, modes in both directions,
shallow and full clones, the planted-symlink and planted-directory
substitutions, and each guard above alongside a control path that must still
succeed.

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
