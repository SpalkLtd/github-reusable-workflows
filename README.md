# github-reusable-workflows

Reusuable Github Workflows for Spalk Organisation

Example of how to use:

```
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
