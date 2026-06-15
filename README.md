# gha-action-opentofu-plan

Runs an OpenTofu plan against an AWS account, optionally posting the result as a sticky pull-request comment.

## Permissions

The calling job must grant:

- `id-token: write` — required for OIDC federation to the target AWS role.
- `contents: read` — required to check out the calling repo (usually granted by default).
- `pull-requests: write` — required **only** if you want PR comments. Omit if you set `post_pr_comment: "false"` or never run the action from a `pull_request` event.

Example permissions block:

```yaml
permissions:
  id-token: write
  contents: read
  pull-requests: write
```

## Inputs

| Name                       | Required | Default       | Description |
|----------------------------|----------|---------------|-------------|
| `aws_account_id`           | yes      | —             | AWS account to plan against. |
| `aws_region`               | yes      | —             | AWS region passed to OpenTofu as `region`. |
| `env_name`                 | yes      | —             | Environment name passed to OpenTofu as `env` (e.g. `nonprod`, `prod`). |
| `infrastructure_directory` | no       | `infra`       | Directory containing the OpenTofu configuration. |
| `iam_role_name`            | no       | `tofu-rw`     | IAM role name to assume in the target account. |
| `use_tfvars`               | no       | `false`       | When `"true"`, pass a `-var-file` to OpenTofu. |
| `tfvars_file`              | no       | `""`          | Path to a `.tfvars` file relative to `infrastructure_directory`. Defaults to `<env_name>.tfvars` when `use_tfvars` is `"true"`. |
| `post_pr_comment`          | no       | `"true"`      | Post plan output as a PR comment on `pull_request` events. Set to `"false"` to disable. |
| `github_token`             | no       | `github.token`| Token used to post the PR comment. |

## PR comments

The action captures `tofu plan -no-color` and posts it as a comment on the related PR. It works with both event types:

- **`pull_request` (and `pull_request_target`)** — comments directly on the triggering PR.
- **`push` / `workflow_dispatch`** — looks up an open PR whose head branch matches the current ref via the GitHub API and comments there. If no open PR exists yet (e.g. you push a branch before opening one), the step skips silently.

The comment uses a sticky marker — `<!-- tofu-plan:{env_name}:{infrastructure_directory} -->` — so subsequent runs against the same PR + env + directory **update the existing comment in place** rather than stacking new ones. Different `env_name` or `infrastructure_directory` values get their own independent comments, so you can plan multiple envs from the same PR without them clobbering each other.

Plan output is truncated at ~60,000 characters with a note pointing reviewers to the workflow logs for the full diff. A failing plan still fails the workflow (the step's exit code is the plan's exit code) — the comment is posted either way so reviewers can see what went wrong.

When there's no associated PR (e.g. push to `main`, manual dispatch outside a branch context), the comment step is a silent no-op.

## Example usage

```yaml
name: Plan
on:
  pull_request:
    paths:
      - 'infra/**'

jobs:
  plan-nonprod:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: kraig-mcfadden/gha-action-opentofu-plan@v1
        with:
          aws_account_id: '123456789012'
          aws_region: us-east-1
          env_name: nonprod
```
