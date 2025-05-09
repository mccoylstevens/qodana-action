# Qodana Scan [<img src="https://api.producthunt.com/widgets/embed-image/v1/top-post-badge.svg?post_id=304841&theme=dark&period=daily" alt="" align="right" width="190" height="41">](https://www.producthunt.com/posts/jetbrains-qodana)

[![official JetBrains project](https://jb.gg/badges/official.svg)][jb:confluence-on-gh]
[![GitHub Discussions](https://img.shields.io/github/discussions/jetbrains/qodana)][jb:discussions]
[![Twitter Follow](https://img.shields.io/badge/follow-%40Qodana-1DA1F2?logo=twitter&style=social)][jb:twitter]

**Qodana** is a code quality monitoring tool that identifies and suggests fixes for bugs, security vulnerabilities,
duplications, and imperfections.

**Table of Contents**

<!-- toc -->

- Qodana Scan
    - [Usage](#usage)
    - [Configuration](#configuration)
- [Issue Tracker](#issue-tracker)

<!-- tocstop -->
[//]: # (title: GitHub Actions)

## Usage

The [Qodana Scan GitHub action](https://github.com/marketplace/actions/qodana-scan)
allows you to run Qodana on a GitHub repository.

<anchor name="basic-configuration"></anchor>

### Basic configuration

To configure Qodana Scan, save the `.github/workflows/code_quality.yml` file containing the workflow configuration:

```yaml
name: Qodana
on:
  workflow_dispatch:
  pull_request:
  push:
    branches:
      - main
      - 'releases/*'

jobs:
  qodana:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      checks: write
    steps:
      - uses: actions/checkout@v3
        with:
          ref: ${{ github.event.pull_request.head.sha }}  # to check out the actual pull request commit, not the merge commit
          fetch-depth: 0  # a full history is required for pull request analysis
      - name: 'Qodana Scan'
        uses: JetBrains/qodana-action@v2025.1
        env:
          QODANA_TOKEN: ${{ secrets.QODANA_TOKEN }} # read the steps about it below
```

To set `QODANA_TOKEN` environment variable in the build configuration:

1. In the GitHub UI,
   create the `QODANA_TOKEN` [encrypted secret](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository)
   and
   save the [project token](https://www.jetbrains.com/help/qodana/cloud-projects.html#cloud-manage-projects) as its value.
2. In the GitHub workflow file,
   add `QODANA_TOKEN` variable to the `env` section of the `Qodana Scan` step:

Using this workflow, Qodana will run on the main branch, release branches, and on the pull requests coming to your
repository.

Note: `fetch-depth: 0` is required for checkout in case Qodana works in pull request mode
(reports issues that appeared only in that pull request).

We recommend that you have a separate workflow file for Qodana
because [different jobs run in parallel](https://help.github.com/en/actions/getting-started-with-github-actions/core-concepts-for-github-actions#job)

![Qodana Cloud](https://user-images.githubusercontent.com/13538286/214899046-572649db-fe62-49b2-a368-b5d07737c1c1.gif)

### Apply quick-fixes

To make Qodana automatically fix found issues and push the changes to your repository,
you need
to
1. Choose what kind of fixes to apply
    - [Specify `fixesStrategy` in the `qodana.yaml` file in your repository root](https://www.jetbrains.com/help/qodana/qodana-yaml.html)
    - Or set the action `args` property with the quick-fix strategy to use: `--apply-fixes` or `--cleanup`
2. Set `push-fixes` property to
    - `pull-request`: create a new branch with fixes and create a pull request to the original branch
    - or `branch`: push fixes to the original branch. Also, set `pr-mode` to `false`: currently, this mode is not supported for applying fixes.
3. Set the correct permissions for the job (`contents: write`, `pull-requests: write`, `checks: write`)
    - If you use `pull-request` value for `push-fixes` property: [**allow GitHub Actions to create and approve pull requests**](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository#preventing-github-actions-from-creating-or-approving-pull-requests)

Example configuration:

```yaml
- name: Qodana Scan
  uses: JetBrains/qodana-action@v2025.1
  with:
    pr-mode: false
    args: --apply-fixes
    push-fixes: pull-request
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

> **Note**
> Qodana could automatically modify not only the code, but also the configuration in `.idea`: if you do not wish to push these changes, add `.idea` to your `.gitignore` file.

If you want to do different `git` operations in the same job, you can disable `push-fixes` and do the wanted operations manually

<details>
<summary> 💡Full script example </summary>

```yaml
name: Qodana
on:
  workflow_dispatch:
  pull_request:
  push:
    branches:
      - master
      - 'releases/*'

jobs:
  qodana:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      checks: write
    steps:
      - uses: actions/checkout@v3
        with:
          ref: ${{ github.event.pull_request.head.sha }}
          fetch-depth: 0
      - name: 'Qodana Scan'
        uses: JetBrains/qodana-action@v2025.1
        with:
          args: --cleanup
      - run: |
          git config user.name github-actions
          git config user.email github-actions@github.com
          git checkout -b quick-fixes-$GITHUB_RUN_ID
          git add -- . ':!.idea'
          git commit -m "I fixed some issues"
          git push origin quick-fixes-$GITHUB_RUN_ID
          gh pr create --repo $GITHUB_REPOSITORY --base $GITHUB_REF_NAME --head quick-fixes-$GITHUB_RUN_ID --title "Pull requests" --body "I fixed some issues"
        env:
          GH_TOKEN: ${{ github.token }}
```
</details>

### GitHub code scanning

You can set
up [GitHub code scanning](https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/about-code-scanning)
for your project using Qodana. To do it, add these lines to the `code_quality.yml` workflow file right below
[the basic configuration](#basic-configuration) of Qodana Scan:

```yaml
      - uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: ${{ runner.temp }}/qodana/results/qodana.sarif.json
```

This sample invokes `codeql-action` for uploading a SARIF-formatted Qodana report to GitHub, and specifies the report
file using the `sarif_file` key.

> GitHub code scanning does not export inspection results to third-party tools, which means that you cannot use this data for further processing by Qodana. In this case, you have to set up baseline and quality gate processing on the Qodana side prior to submitting inspection results to GitHub code scanning, see the
[Quality gate and baseline](#quality-gate-and-baseline) section for details.

### Pull request quality gate

You can enforce GitHub to block the merge of pull requests if the Qodana quality gate has failed. To do it, create a
[branch protection rule](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/managing-a-branch-protection-rule)
as described below:

1. Create a new or open an existing GitHub workflow that invokes the Qodana Scan action.
2. Set the workflow to run on `pull_request` events that target the `main` branch.

```yaml
on:
  pull_request:
    branches:
      - main
```

Instead of `main`, you can specify your branch here.

3. Set the number of problems (integer) for the Qodana action `fail-threshold` option.
4. Under your repository name, click **Settings**.
5. On the left menu, click **Branches**.
6. In the branch protection rules section, click **Add rule**.
7. Add `main` to **Branch name pattern**.
8. Select **Require status checks to pass before merging**.
9. Search for the `Qodana` status check, then check it.
10. Click **Create**.

<anchor name="quality-gate-and-baseline"></anchor>

### Quality gate and baseline

You can combine the [quality gate](https://www.jetbrains.com/help/qodana/quality-gate.html) and [baseline](https://www.jetbrains.com/help/qodana/qodana-baseline.html) features to manage your
technical debt, report only new problems, and block pull requests that contain too many problems.

Follow these steps to establish a baseline for your project:

1. Run Qodana [locally](https://www.jetbrains.com/help/qodana/getting-started.html#Analyze+a+project+locally) over your project:

```shell
cd project
qodana scan --show-report
```

2. Open your report at `http://localhost:8080/`, [add detected problems](https://www.jetbrains.com/help/qodana/ui-overview.html#Technical+debt) to the baseline,
   and download the `qodana.sarif.json` file.

3. Upload the `qodana.sarif.json` file to your project root folder on GitHub.

4. Append `--baseline,qodana.sarif.json` argument to the Qodana Scan action configuration `args` parameter in the `code_quality.yml` file:

```yaml
- name: Qodana Scan
  uses: JetBrains/qodana-action@main
  with:
    args: --baseline,qodana.sarif.json
```

If you want to update the baseline, you need to repeat these steps once again.

Starting from this, GitHub will generate alters only for the problems that were not added to the baseline as new.

To establish a quality gate additionally to the baseline, add this line to `code_quality.yml` right after the
`baseline-path` line:

```yaml
fail-threshold: <number-of-accepted-problems>
```

Based on this, you will be able to detect only new problems in pull requests that fall beyond the baseline. At the same
time, pull requests with **new** problems exceeding the `fail-threshold` limit will be blocked, and the workflow will fail.

### Get a Qodana badge

You can set up a Qodana workflow badge in your repository, to do it, follow these steps:

1. Navigate to the workflow run that you previously configured.
2. On the workflow page, select **Create status badge**.
3. Copy the Markdown text to your repository README file.

<img src="https://user-images.githubusercontent.com/13538286/148529278-5d585f1d-adc4-4b22-9a20-769901566924.png" alt="Creating status badge" width="706"/>

## Configuration

Most likely, you won't need other options than `args`: all other options can be helpful if you are configuring multiple Qodana Scan jobs in one workflow.

Use [`with`](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepswith) to define any action parameters:

```yaml
with:
  args: --baseline,qodana.sarif.json
  cache-default-branch-only: true
```

| Name                        | Description                                                                                                                                                                                  | Default Value                                       |
|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| `args`                      | Additional [Qodana CLI `scan` command](https://github.com/jetbrains/qodana-cli#scan) arguments, split the arguments with commas (`,`), for example `-i,frontend,--print-problems`. Optional. | -                                                   |
| `results-dir`               | Directory to store the analysis results. Optional.                                                                                                                                           | `${{ runner.temp }}/qodana/results`                 |
| `upload-result`             | Upload Qodana results (SARIF, other artifacts, logs) as an artifact to the job. Optional.                                                                                                    | `false`                                             |
| `artifact-name`             | Specify Qodana results artifact name, used for results uploading. Optional.                                                                                                                  | `qodana-report`                                     |
| `cache-dir`                 | Directory to store Qodana cache. Optional.                                                                                                                                                   | `${{ runner.temp }}/qodana/caches`                  |
| `use-caches`                | Utilize [GitHub caches](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#usage-limits-and-eviction-policy) for Qodana runs. Optional.           | `true`                                              |
| `primary-cache-key`         | Set [the primary cache key](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#matching-a-cache-key). Optional.                                   | `qodana-2025.1-${{ github.ref }}-${{ github.sha }}` | 
| `additional-cache-key`      | Set [the additional cache key](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#matching-a-cache-key). Optional.                                | `qodana-2025.1-${{ github.ref }}`                   |
| `cache-default-branch-only` | Upload cache for the default branch only. Optional.                                                                                                                                          | `false`                                             |
| `use-annotations`           | Use annotation to mark the results in the GitHub user interface. Optional.                                                                                                                   | `true`                                              |
| `pr-mode`                   | Analyze ONLY changed files in a pull request. Optional.                                                                                                                                      | `true`                                              |
| `post-pr-comment`           | Post a comment with the Qodana results summary to the pull request. Optional.                                                                                                                | `true`                                              |
| `github-token`              | GitHub token to access the repository: post annotations, comments. Optional.                                                                                                                 | `${{ github.token }}`                               |
| `push-fixes`                | Push Qodana fixes to the repository, can be `none`, `branch` to the current branch, or `pull-request`. Optional.                                                                             | `none`                                              |


## Issue Tracker

All the issues, feature requests, and support related to Qodana are handled on [YouTrack][youtrack].

If you'd like to file a new issue, please use the link [YouTrack | New Issue][youtrack-new-issue].

[gh:qodana]: https://github.com/JetBrains/qodana-action/actions/workflows/code_scanning.yml
[youtrack]: https://youtrack.jetbrains.com/issues/QD
[youtrack-new-issue]: https://youtrack.jetbrains.com/newIssue?project=QD&c=Platform%20GitHub%20action
[jb:confluence-on-gh]: https://confluence.jetbrains.com/display/ALL/JetBrains+on+GitHub
[jb:discussions]: https://jb.gg/qodana-discussions
[jb:twitter]: https://twitter.com/Qodana
[jb:docker]: https://hub.docker.com/r/jetbrains/qodana
