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

4.  The AOD request PR cannot and should not be merged

    Please close the PR when you are done and a workflow will be triggered to do
    cleanup. Otherwise, the PR will automatically be closed after X hours
    depending on how you configure your
    [expire.yml](.github/workflows/expire.yml) job.

## How AOD works(TODO #7)

## Prerequisites

The admin of your GCP project/folder/org to complete the steps below.

1.  Create an AOD repository using this template, only copy main branch is
    required.

2.  Set up
    [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation),
    and a service account with adequate condition and permission, see guide
    [here](https://github.com/google-github-actions/auth#setting-up-workload-identity-federation).

    -   Make sure to map the attribute `"attribute.job_workflow_ref":
        "assertion.job_workflow_ref"` and add an attribute condition
        `matches(attribute.job_workflow_ref, \"abcxyz/access-on-demand/*\")`
        when creating the workload identity pool provider to only allow trusted
        GitHub workflows which are workflows from `abcxyz/access-on-demand`
        repo.

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

    -   Disable forking
    -   Set up
        [CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
        with the group to approve AOD requests
    -   Branch protection on the `main` (default) branch
        -   Require a pull request before merging
            -   Require approvals
            -   Dismiss stale pull request approvals when new commits are pushed
            -   Require review from Code Owners
            -   Require approval of the most recent reviewable push
        -   Require status checks to pass before merging
        -   Require signed commits
        -   [Lock branch](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#lock-branch)
        -   Disallow force pushes
        -   Disallow deletions

## Adding code other than AOD

You need to have:

-   A PR ready to merge: open a PR with signed commits -> pass status checks if
    any -> get required approvals.
-   Permissions to bypass the `lock branch` branch protection rule.
