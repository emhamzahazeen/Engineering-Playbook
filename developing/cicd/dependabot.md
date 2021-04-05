# [CI/CD](README.md) / Dependabot

[Dependabot](https://dependabot.com/) is a tool to automatically find updates to dependencies in our repos.

## Configuration

The Dependabot website has configuration for when/how often to run checks and configuration variables but you'll configure each repo in a file `.dependabot/config.yml`.

Example from [Terraform AWS Alb Web Containers](https://github.com/trussworks/terraform-aws-alb-web-containers) module:

```yml
version: 1
update_configs:
  # Keep go modules up to date, batching pull requests weekly
  - package_manager: "go:modules"
    directory: "/"
    update_schedule: "weekly"
    # Apply default reviewer @trussworks/waddlers group to PRs
    default_reviewers:
      - "trussworks/waddlers"
    # Apply dependencies label to PRs
    default_labels:
      - "dependencies"
    automerged_updates:
      - match:
          dependency_type: "all"
          update_type: "all"

```

Dependabot can be configured to auto merge updates but respects branch protections. So, if you have automated tests, it should be stopped by a failure or if the branch requires a review from an owner.

### Auto Approve

If you decide that your repo can automatically merge any of these changes without a human approving or looking at them you can configure [Github Actions](https://github.com/features/actions) to evaluate and auto approve these changes from Dependabot.

Note: We tried [mergify](https://mergify.io/) and found that it could not work around the branch protections that we strongly suggest.

We suggest using [Auto-approve action](https://github.com/hmarr/auto-approve-action) in a file `.github/workflows/auto-approve.yml`.

You do not need to set the `GITHUB_TOKEN` as Github's bot should be able to run this for you.

Example configuration from [Terraform AWS Alb Web Containers](https://github.com/trussworks/terraform-aws-alb-web-containers):

```yml
name: Auto approve

on:
  pull_request

jobs:
  auto-approve:
    runs-on: ubuntu-latest
    steps:
    - uses: hmarr/auto-approve-action@v2.0.0
      if: github.actor == 'dependabot[bot]' || github.actor == 'dependabot-preview[bot]'
      with:
        github-token: "${{ secrets.GITHUB_TOKEN }}"
```
