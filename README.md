# repo-template

**This is not an official Google product.**

This template is for creating an AOD (access on demand) repository to get access
to GCP resources on demand.

## Make an AOD Request

**Make sure you have completed
[prerequisite steps](https://github.com/abcxyz/aod-template/blob/main/README.md#prerequisites)
to set up this repo properly.**

1.  Open a PR

    -   A title and description to provide enough justification. E.g. "AOD
        request: debug customer issue X".
    -   To request IAM permissions on org/folder/project level, add an
        `iam.yaml` file in the repo root, see [example](example-iam.yaml).
        -   (Optional) Use predefined duration labels on the PR to specify the
            IAM permission expiration. Otherwise a 2h default duration will be
            used.
    -   To request on-demand `gcloud` commands, add a `tool.yaml` file in the
        repo root, see [example](example-tool.yaml).

2.  Checks

    -   The "Validate Request" check will report errors if there are problems in
        your `iam.yaml` or `tool.yaml`.

3.  Getting approval

    -   Ask one of the repo code owners to approve the PR.
    -   Upon approval, another GitHub workflow will run to grant you the IAM
        permissions or run the on-demand `gcloud` commands (`do` commands in
        your `tool.yaml` file).
    -   The result will be posted as PR comments.

4.  The AOD request PR **CANNOT** and **SHOULD NOT** be merged

    Please close the PR when you are done and a workflow will be triggered to do
    cleanup.

    -   IAM cleanup: removes the requested permissions.
    -   Tool cleanup: executes the requested cleanup commands.

    Otherwise, the PR will automatically be closed after X hours depending on
    how you configure your [expire.yml](.github/workflows/expire.yml) job.

    To retry if cleanup failed, please restore your branch, reopen the PR, and
    then close the PR.

## How AOD works

Please refer to the high level flow [here](https://github.com/abcxyz/access-on-demand#high-level-flow).

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

    -   When creating the workload identity pool provider, make sure to map the
        attributes such as `"attribute.job_workflow_ref":
        "assertion.job_workflow_ref"` and add attribute conditions:
        -   `attribute.event_name != \"pull_request_target\"` to prevent
            workflows triggered by a forked repository.
        -   `attribute.repository_owner_id == \"${var.github_owner_id}\" &&
            attribute.repository_id == \"${var.github_repository_id}\"` to only
            allow workflows from your AOD repository.
        -   `matches(attribute.job_workflow_ref, \"abcxyz/access-on-demand/*\")`
            to only allow trusted workflow jobs which are from
            `abcxyz/access-on-demand` source repo.

    -   The minimum permissions for the service account are `getIamPolicy` and
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

4.  (Optional) Create labels to be used as IAM permission expiration. IAM
    permissions granted via AOD will have a default 2 hour expiration when the
    request is approved. To use custom expirations, follow steps
    [here](https://docs.github.com/en/issues/using-labels-and-milestones-to-track-work/managing-labels#creating-a-label)
    to create duration labels. `duration-4h` is an example of valid duration
    label.

5.  It is critical to enable the following repo settings:

    #### Repository settings

    -   Disable forking
    -   Set up
        [CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
        with the group to approve AOD requests
    -   Add a new branch Ruleset on the target `main` (default) branch
        -   Restrict updates
        -   Restrict deletions
        -   Require signed commits
        -   Require a pull request before merging
            -   Require at least 1 approvals
            -   Dismiss stale pull request approvals when new commits are pushed
            -   Require approval of the most recent reviewable push
        -   Require status checks to pass before merging
        -   Block force pushes

## Adding code other than AOD

Rarely but the repo admins might need to check in changes to the repo that are
not AOD request. To do that, the admin would need to send PRs as usual with
"lock branch" temporarily disabled. This activity should be coordinated
carefully to make sure AOD request PRs won't be merged accidentally.
