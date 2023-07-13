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
        `iam.yaml` file in the repo root.
    -   See an example of this file, see example in
        [example-iam.yaml](https://github.com/abcxyz/aod-template/blob/main/example-iam.yaml).
    -   (Optional) Use predefined duration labels on the PR to specify the IAM
        permission expiration. Otherwise a 2h default duration will be used.
    -   (Later) To request on-demand `gcloud` commands, add an `gcloud.yaml`
        file in the repo root.

2.  Checks

    -   All AOD PRs will fail the check "do_not_merge" to prevent accidental
        merge.
    -   The "Validate Request" check will report errors if there are problems in
        your `iam.yaml` or `gcloud.yaml`.

3.  Getting approval

    -   Ask one of the repo code owners to approve the PR
    -   Upon approval, another GitHub workflow will run to grant you the IAM
        permissions or run the on-demand `gcloud` commands

4.  Close the PR after you no longer need the access. The PR will automatically
    be closed after X hours (TODO #8).

## How AOD works(TODO #7)

## Prerequisites

The admin of your GCP project/folder/org to complete the steps below.

1.  Create an AOD repository using this template, only copy main branch is
    required.

2.  Set up
    [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation),
    and a service account, see guide
    [here](https://github.com/google-github-actions/auth#setting-up-workload-identity-federation).

    -   The minimum permissions are `getIamPolicy` and `setIamPolicy`. These
        permissions can be on projects, folders, or organizations level. For
        example,
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

    -   [Lock branch](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#lock-branch)
    -   Branch protection on main branch
    -   Disable forking
    -   Require approvals
    -   Set up [CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) with the group to approve AOD requests
    -   Require review from Code Owners
    -   Disallow specific actors to bypass required pull requests
    -   Require status checks to pass before merging
    -   Require signed commits
    -   Disallow force pushes
    -   Disallow deletions

## Adding code other than AOD (TODO #9)
