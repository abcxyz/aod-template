# repo-template

**This is not an official Google product.**

This template is for creating an AOD (access on demain) repository for GCP organizations, folders, and projects.

After completing the installation. Follow below steps to file an AOD request.
1. Open a PR
  - A title and description to provide enough justification. E.g. "AOD request: debug customer issue X".
  - Add following file at the repo root:
    - iam.yaml - Request for IAM permissions on organization / folder / project level. To see an example of this file, refer to the example-iam.yaml file.
  - Optionally use predefined labels on the PR to specify the AOD duration. The duration is default to 2h if no label is used.

2. PR approval
  - Github workflows will check if the PR a AOD request, marked by a "do_not_merge" failure workflow, and if it is a valid AOD request.
  - Upon PR approval, the request will be handled by AOD.

3. Close the PR after the request is handled successfully.

## Installation

1.  Create an AOD repository using this template, only copy main branch is requried.
2.  Setup [Workload Identity Federation](https://cloud.google.com/iam/docs workload-identity-federation).
  - Create workload identity pool and provider in a GCP project where appropriate.
  - Create a GCP service account that identities from this pool can impersonate.
  - Grant the service account the required IAM roles to perform AOD operations.
    -  The minimum permissions are the get and set IAM policy. These permissions can be on projects, folders, or organizations level. For example, ["roles/resourcemanager.projectIamAdmin"](https://cloud.google.com/resource-manager/docs/access-control-proj#resourcemanager.projectIamAdmin) is a predefined role for managing project level IAM permission, an alternative is to use custom roles for a more restricted set of permissions.

Below is an example of how you setup Workload Identity Federation via terraform.

```terraform
# Workload Identity Pool
resource "google_iam_workload_identity_pool" "github_pool" {
  project = var.project_id

  workload_identity_pool_id = "github-pool-${random_id.default.hex}"
  display_name              = var.wif_pool_name
  description               = "Identity pool for GitHub repo ${var.github_repository_id}"

  depends_on = [
    google_project_service.services["iam.googleapis.com"],
  ]
}

# Workload Identity Provider
resource "google_iam_workload_identity_pool_provider" "github_provider" {
  project = var.project_id

  workload_identity_pool_id          = google_iam_workload_identity_pool.github_pool.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-provider"
  display_name                       = var.wif_provider_name
  description                        = "GitHub OIDC identity provider for GitHub repo ${var.github_repository_id}"
  attribute_mapping = {
    "google.subject" : "assertion.sub"
    "attribute.actor" : "assertion.actor"
    "attribute.aud" : "assertion.aud"
    "attribute.event_name" : "assertion.event_name"
    "attribute.repository_owner_id" : "assertion.repository_owner_id"
    "attribute.repository" : "assertion.repository"
    "attribute.repository_id" : "assertion.repository_id"
    "attribute.workflow" : "assertion.workflow"
  }

  # We create conditions based on ID instead of name to prevent name hijacking
  # or squatting attacks.
  #
  # We also prevent pull_request_target, since that runs arbitrary code:
  #   https://securitylab.github.com/research/github-actions-preventing-pwn-requests/
  attribute_condition = "attribute.event_name != \"pull_request_target\" && attribute.repository_owner_id == \"${var.github_owner_id}\" && attribute.repository_id == \"${var.github_repository_id}\""

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }

  depends_on = [
    google_iam_workload_identity_pool.github_pool
  ]
}

# Service account
resource "google_service_account" "wif_service_account" {
  project = var.project_id

  account_id   = "${substr(var.name, 0, 19)}-${random_id.default.hex}-ci-sa" # 30 character limit
  display_name = "${var.name} CI Service Account"
}

resource "google_service_account_iam_member" "wif_github_iam" {
  service_account_id = google_service_account.wif_service_account.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github_pool.name}/*"
}

# Grant permissions required for operation AOD. This is just an example, your team is responsible for granting the right permissions.
resource "google_project_iam_member" "ci_service_account_iam" {
  for_each = toset([
    "roles/resourcemanager.projectIamAdmin"
  ])

  project = "suhongq-dev-041665"

  role   = each.key
  member = module.github_ci_infra.service_account_member
}

# Output below values for step 3.
output "wif_provider_name" {
  description = "The Workload Identity Federation provider name."
  value       = google_iam_workload_identity_pool_provider.github_provider.name
}

output "service_account_email" {
  description = "WIF service account identity email address."
  value       = google_service_account.wif_service_account.email
}
```

3.  Follow steps [here](https://docs.github.com/en/actions/learn-github-actions/variables#creating-configuration-variables-for-a-repository) to set the repository variables for WORKLOAD_IDENTITY_PROVIDER and SERVICE_ACCOUNT with outputs from step 2.

4. (Optional) Create labels to be used as IAM permission expiration. IAM permissions granted via AOD will have a default 2 hour expiration when the request is approved. To use custom expirations, follow steps [here](https://docs.github.com/en/issues/using-labels-and-milestones-to-track-work/managing-labels#creating-a-label) to create duration labels. "duration-4h" is an example of valid duration label.

5. It is critical to enable the following repo settings:
  - Disable forking
  - Branch protection on main branch
  - Require approvals
  - Require review from Code Owners
  - Disallow specific actors to bypass required pull requests
  - Require status checks to pass before merging
  - Require signed commits
  - Disallow force pushes
  - Disallow deletions
  - Set up CODEOWNERS with the group to approve AOD requests
