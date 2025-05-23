# aod-template

**This is not an official Google product.**

This template is for creating an AOD (access on demand) repository to get access
to GCP resources on demand.

## Make an AOD Request

**Make sure you have completed
[prerequisite steps](https://github.com/abcxyz/aod-template/blob/main/README.md#prerequisites)
to set up this repo properly.**

1.  Open a PR

    - A title and description to provide enough justification. E.g. "AOD
      request: debug customer issue X".
    - (Optional) Add a line like AOD_DURATION={duration} at the end of your PR
      description in order to change the default AOD expiration from 2h.
      Examples: `AOD_DURATION=45m`, `AOD_DURATION=24h`, `AOD_DURATION=72h`. For
      more details see https://pkg.go.dev/time#ParseDuration.
    - To request IAM permissions on org/folder/project level, add an `iam.yaml`
      file in the repo root, see [example](example-iam.yaml).
    - To request on-demand `gcloud` commands, add a `tool.yaml` file in the repo
      root, see [example](example-tool.yaml).

2.  Checks

    - The "Validate Request" check will report errors if there are problems in
      your `iam.yaml` or `tool.yaml`.

3.  Getting approval

    - Ask one of the repo code owners to approve the PR.
    - Upon approval, another GitHub workflow will run to grant you the IAM
      permissions or run the on-demand `gcloud` commands (`do` commands in your
      `tool.yaml` file).
    - The result will be posted as PR comments.

4.  The AOD request PR **CANNOT** and **SHOULD NOT** be merged

    Please close the PR when you are done and a workflow will be triggered to do
    IAM cleanup.

    - Removes the requested permissions even if the AOD_DURATION has not
    elapsed.
    - Removes any expired premissions granted by AOD.

    Otherwise, the PR will automatically be closed after X hours of the last
    committed time depending on how you configure your
    [expire.yml](.github/workflows/expire.yml) job. The cleanup workflow will
    be triggered afterwards. With the expire workflow, it enforces a maximum
    IAM permissions expiration.

    To retry if cleanup failed, please restore your branch, reopen the PR, and
    then close the PR.

## How AOD works

Please refer to the high level flow
[here](https://github.com/abcxyz/access-on-demand#high-level-flow).

### Breakglass (optional)

Some teams may have the need to allow users to "breakglass" or escalate
permissions in an emergency or debugging situation. Often, this happens during
off hours when there aren't any other teammates around to approve these time
sensitive requests. To allow for this, the requester should add the
`aod-breakglass` label to their pull request. When this label is added, the
workflows will automatically trigger the request allowing for the access needed
in these scenarios. See the following section to learn how to setup this label
for your repository.

## Prerequisites

The admin of your GCP project/folder/org to complete the steps below. If you
have groups that cannot access each other's resources, please repeat the process
to create an AOD instance for each.

1.  Create an AOD repository using this template, only copy main branch is
    required.

2.  Set up
    [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation),
    and a service account with adequate condition and permission, see guide
    [here](https://github.com/google-github-actions/auth#setting-up-workload-identity-federation).
    Please restrict any human access to this service account, it should only be
    used by your AOD instance.

    - When creating the workload identity pool provider, make sure to map the
      attributes such as
      `"attribute.job_workflow_ref": "assertion.job_workflow_ref"` and add
      attribute conditions:

      - `attribute.event_name != \"pull_request_target\"` to prevent workflows
        triggered by a forked repository.
      - `attribute.repository_owner_id == \"${var.github_owner_id}\" && attribute.repository_id == \"${var.github_repository_id}\"`
        to only allow workflows from your AOD repository.
      - `matches(attribute.job_workflow_ref, \"abcxyz/access-on-demand/*\")` to
        only allow trusted workflow jobs which are from
        `abcxyz/access-on-demand` source repo.

    - The minimum permissions for the service account are `getIamPolicy` and
      `setIamPolicy`. These permissions can be on projects, folders, or
      organizations level. For example,
      ["roles/resourcemanager.projectIamAdmin"](https://cloud.google.com/resource-manager/docs/access-control-proj#resourcemanager.projectIamAdmin)
      is a predefined role for managing project level IAM permission, an
      alternative is to use custom roles for a more restricted set of
      permissions.

3.  Follow steps
    [here](https://docs.github.com/en/actions/learn-github-actions/variables#creating-configuration-variables-for-a-repository)
    to set the repository variables for WORKLOAD_IDENTITY_PROVIDER and
    SERVICE_ACCOUNT with outputs from step 2.

4.  (Optional) Create the `aod-breakglass` label to be used as the breakglass
    identifier for auto-approvals during an emergency situation. Follow the
    steps
    [here](https://docs.github.com/en/issues/using-labels-and-milestones-to-track-work/managing-labels#creating-a-label)
    to create the label `aod-breakglass`.

5.  It is critical to enable the following repo settings:

    #### Repository settings

    - Disable forking
    - Set up
      [CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
      with the group to approve AOD requests
    - Add a new branch Ruleset on the target `main` (default) branch
      - Restrict creations
      - Restrict updates
      - Restrict deletions
      - Require signed commits (Optional, but recommended)
      - Require a pull request before merging
        - Require at least 1 approvals
        - Dismiss stale pull request approvals when new commits are pushed
        - Require review from Code Owners
        - Require approval of the most recent reviewable push
      - Require status checks to pass before merging
      - Block force pushes

## Adding code other than AOD

Rarely but the repo admins might need to check in changes to the repo that are
not AOD request. To do that, the admin would need to send PRs as usual with
branch ruleset temporarily disabled, or grant bypass permissions when needed
(recommended), please check
[ruleset](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets#about-rulesets)
for more details. This activity should be coordinated carefully to make sure AOD
request PRs won't be merged accidentally.

A good example of these changes is to use the latest AOD workflows, repo admins
will need to check in changes to upgrade each of the workflows used in your AOD
repository, we recommend to use
[ratchet upgrade](https://github.com/sethvargo/ratchet?tab=readme-ov-file#upgrade)
for this type of change.
