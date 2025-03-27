# Heroku Flow Action
GitHub Action to upload the source code to Heroku from a private GitHub repository using the [Heroku Source Endpoint API](https://devcenter.heroku.com/articles/build-and-release-using-the-api#sources-endpoint). It mimics and extends the [Heroku GitHub integration](https://devcenter.heroku.com/articles/github-integration). The uploaded code is built to either deploy an app (on `push`, `workflow_dispatch` and `schedule` events) or create/update a review app (on `pull_request` events such as `opened`, `reopened`, `synchronize`). Whenever a PR branch is updated, the latest commit is deployed to the review app if existing, otherwise a new review app is created. 
The Review App is automatically removed when the pull request is closed (on `pull_request` events when the action is `closed`).
The action handles only the above mentioned events to prevent unexpected behavior, handling event-specific requirements and improving action reliability.

Despite it's possible to use this Action to create review apps without configuring the [Heroku GitHub integration](https://devcenter.heroku.com/articles/github-integration), to display them within your Pipeline it's necessary to enable it. It's enough to use an empty repository unrelated to your actual source repository and GitHub Org used by the Action, this will be used only as a workaround to make review apps visible. In this case the link shown in the Review App won't refer to your source code.

The [Heroku CI](https://devcenter.heroku.com/articles/heroku-ci) is not triggered with this Action as it's only activated via the Heroku GitHub integration.

# Supported Use Cases

- Review App creation on PR open
- Review App build and deploy on PR branch commit
- Review App creation on PR commit when no RA exists (e.g. deleted after the PR creation)
- Review App removal on PR close (without commit/merge to the main branch)
- Review App removal on PR close (with commit/merge to the main branch)
- App build and deploy on code push
- Manual App build and deploy via GitHub Actions “Run Workflow”
- Scheduled App build and deploy via GitHub Actions cron/scheduler


## Disclaimer
The author of this article makes any warranties about the completeness, reliability and accuracy of this information. **Any action you take upon the information of this website is strictly at your own risk**, and the author will not be liable for any losses and damages in connection with the use of the website and the information provided. **None of the items included in this repository form a part of the Heroku Services.**

## How to use it
Create a `.github/workflows` directory within your repository using the following YAML files (either an all-in-one file `push-and-pr.yml` or three separate files `push.yml`, `pr-opened.yml`, `pr-closed.yml` to deal with specific workflows) and configure them. <br/>

Additionally, if you want to enable the rebuild of an app manually or on schedule use `rebuild-app.yml`.

Configure the workflow files with the following input parameters:
  - `heroku-api-key` [required] Heroku API key used to create and deploy apps. Set it on GitHub as secret
  - `heroku-app-name` [required] Heroku app name. Set it on GitHub as variable at repository level
  - `heroku-pipeline-id` [required] used only for review apps creation. Set it on GitHub as variable at repository level
  - `remove-git-folder` [optional] overrides the default (true) - it's usually recommended to avoid exposing the .git folder
  - `review-app-creation-check-timeout` [optional] Review App creation timeout
  - `review-app-creation-check-sleep-time` [optional] sleep time while polling for Review Apps status

In a GitHub Workflow, this action requires to be preceeded by the [actions/checkout](https://github.com/actions/checkout) to work properly. If you need to filter files from your repository before deploy use the `sparse-checkout` option available with `actions/checkout`.


## Workflow examples

This will be executed on `push` and `pull_request` events
```
# push-and-pr.yml
name: Deploy push and PR, delete Review App on PR closed

on:
  push:
    paths-ignore:
      - '.github/workflows/**'
    branches:
      - main

  pull_request:
    paths-ignore:
      - '.github/workflows/**'
    types: [opened, reopened, synchronize, closed]

jobs:
  push-and-pr:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
        with:
          sparse-checkout: |
            /*
            !.gitignore
            !.github
      - uses: heroku-reference-apps/github-heroku-flow-action@v1
        with:
          heroku-api-key: ${{secrets.HEROKU_API_KEY}} # set it on GitHub as secret
          remove-git-folder: false # if you want to override the default (true) - it's usually recommended to avoid exposing the .git folder 
          heroku-app-name: ${{vars.HEROKU_APP_NAME}} # set it on GitHub as variable at repository level
          heroku-pipeline-id: ${{vars.HEROKU_REVIEWAPP_PIPELINE}} # set it on GitHub as variable at repository level
          review-app-creation-check-timeout: 600 # if you want to override the default (3600 seconds) 
          review-app-creation-check-sleep-time: 7 # if you want to override the default (10 seconds), min 5 seconds
```


This will be executed on `push` events
```
# push.yml
name: Deploy push

on:
  push:
    paths-ignore:
      - '.github/workflows/**'
    branches:
      - main

jobs:
  build-push:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
        with:
          sparse-checkout: |
            /*
            !.gitignore
            !.github
      - uses: heroku-reference-apps/github-heroku-flow-action@v1
        with:
          heroku-api-key: ${{secrets.HEROKU_API_KEY}} # set it on GitHub as secret
          heroku-app-name: ${{vars.HEROKU_APP_NAME}} # set it on GitHub as variable at repository level
          remove-git-folder: false # if you want to override the default (true) - it's usually recommended to avoid exposing the .git folder 
```

This will be executed whenever a PR is [`opened`, `reopened`, `synchronize`]
```
# pr-opened.yml
name: Deploy PR (create Review App)

on:
  pull_request:
    paths-ignore:
      - '.github/workflows/**'
    types: [opened, reopened, synchronize]

jobs:
  build-pr:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
        with:
          sparse-checkout: |
            /*
            !.gitignore
            !.github
      - uses: heroku-reference-apps/github-heroku-flow-action@v1
        with:
          heroku-api-key: ${{secrets.HEROKU_API_KEY}} # set it on GitHub as secret
          heroku-pipeline-id: ${{vars.HEROKU_REVIEWAPP_PIPELINE}} # set it on GitHub as variable at repository level
          remove-git-folder: false # if you want to override the default (true) - it's usually recommended to avoid exposing the .git folder
          review-app-creation-check-timeout: 600 # if you want to override the default (3600 seconds) 
          review-app-creation-check-sleep-time: 7 # if you want to override the default (10 seconds), min 5 seconds
```

This will be executed whenever a PR is [`closed`]
```
# pr-closed.yml
name: Close PR (delete Review App)

on:
  pull_request:
    paths-ignore:
      - '.github/workflows/**'
    types: [closed]

jobs:
  close-pr:
    runs-on: self-hosted
    steps:
      - uses: heroku-reference-apps/github-heroku-flow-action@v1
        with:
          heroku-api-key: ${{secrets.HEROKU_API_KEY}} # set it on GitHub as secret
          heroku-pipeline-id: ${{vars.HEROKU_REVIEWAPP_PIPELINE}} # set it on GitHub as variable at repository level
```

This will be executed on schedule and `workflow_dispatch` events
```
# rebuild-app.yml
name: Rebuild App manually or on schedule

on:
  workflow_dispatch:

  schedule:
    - cron: '0 1 * * 0'

jobs:
  rebuild-app:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - uses: heroku-reference-apps/github-heroku-flow-action@v1
        with:
          heroku-api-key: ${{secrets.HEROKU_API_KEY}} # set it on GitHub as secret
          heroku-app-name: ${{vars.HEROKU_APPNAME}} # set it on GitHub as variable
```

## Security Notes and Recommendations
Below are summarised some general recommendations to improve security for using GitHub Actions and self-hosted runners, for a complete guide and further details please refer to [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions):
- Using self-hosted runners with private repositories. This is because forks of your public repository can potentially run dangerous code on your self-hosted runner machine by creating a pull request that executes the code in a workflow
- Disable [Run workflows from fork pull requests](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository#enabling-workflows-for-forks-of-private-repositories) as it could be a potential source of leakage for secrets running in workflows. This option tells Actions to run workflows from pull requests originating from repository forks. Note that doing so will give maintainers of those forks the ability to use tokens with read permissions on the source repository
- Using only audited actions for specific GitHub Org(s), this can be enforced at GitHub level through [policies](https://docs.github.com/en/enterprise-cloud@latest/admin/enforcing-policies/enforcing-policies-for-your-enterprise/enforcing-policies-for-github-actions-in-your-enterprise#enforcing-policies)
- Using [Push rulesets](https://docs.github.com/en/enterprise-cloud@latest/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets#push-rulesets) to block pushes to private/internal repos based on files paths. This can prevent changing the original Workflow or action and inject malicious code
- Configuring a HEROKU_API_KEY at Repo level instead of Org level. This key can be tied to a Heroku user account that has limited permissions to a specific app (the one tied with the repository) limiting the attack area (to others app) in case the HEROKU_API_KEY is being leaked or stolen
- Using Runner Groups to [restrict runners to specific repos](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/managing-access-to-self-hosted-runners-using-groups#changing-which-repositories-can-access-a-runner-group)
- Using dependabot to keep your repositories and Actions up to date. Dependabot can run on GitHub runners [free of charge](https://github.blog/changelog/2024-05-13-dependabot-core-is-now-open-source-with-an-mit-license/)
- Being cautious when adding outside collaborators on GitHub — users with read permissions can view logs for workflow failures, view workflow history, as well as search and download logs
- Handling secrets, tokens and keys securely (e.g. using secrets in [GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions))
- Avoiding script injections, using an action instead of an inline script
- Isolating the runners from other environments if you want to limit potential access due to software bugs or security issues
- The amount of sensitive information in the runner environment should be kept to a minimum, always be mindful that any user capable of invoking workflows has access to this environment
- Accessing to logs and/or secrets through forked repositories should be examined
- Auditing and rotating registered secrets
- Consider requiring review for access to secrets
- Using [CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) to monitor repository changes
- Using [workflow templates](https://docs.github.com/en/actions/writing-workflows/using-workflow-templates) for code scanning
- Restricting permissions for tokens / permissions for the GITHUB_TOKEN
- Restricting [default permissions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication) granted to GITHUB_TOKEN when running workflows (e.g. Read repository contents and package permissions)
- Using third-party Actions: pin actions to a full length commit SHA, audit the source code, pin actions to a tag only if you trust the creator
- Prevent GitHub Actions from creating or approving pull requests
- Using [OpenSSF Scorecards](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions#using-openssf-scorecards-to-secure-workflow-dependencies) to secure workflows
- Evaluating potential impact of a compromised runner (e.g. accessing secrets, exfiltrating data from a runner, stealing the job's GITHUB_TOKEN, modifying the contents of a repository, ...)
- Considering [cross-repository access](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions#considering-cross-repository-access)
- Auditing GitHub Actions events
- Using the runner HIDDEN_ENV_VARS to avoid logging sensible runner environment variables to GitHub logs
- Conducting regular security audits of GitHub Actions workflows and Heroku configurations
- Implementing mandatory code reviews for all workflow changes
- Implementing branch protection rules to limit PRs to trusted contributors or specific branches
- Using Heroku's Review Apps feature to isolate test environments from production
- Using Heroku's Config Vars feature with the least privilege principle, only granting access to variables to allowed users

## Limits and Considerations
- Be mindufl of the number of API calls per Heroku user that you can utilise [4500/h](https://devcenter.heroku.com/articles/limits#heroku-api-limits) executing this Action