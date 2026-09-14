# ADR-0003: Manage AWS infrastructure in Terraform from the first resource

- **Status:** Accepted
- **Date:** 13-Sep-2026

## Context

The program needs ECR repositories, an OIDC identity provider, and IAM
roles, growing later to Fargate, MSK and Secrets Manager. The first
resources were created with the AWS CLI while working interactively.

The question was how to make that reproducible. Recreating the same setup
for each additional service is otherwise a matter of re-running commands
recalled from a chat transcript.

## Options considered

### A — CLI commands recorded in a document
A `docs/aws-commands.md` alongside the service it configures. Cheapest.
Costs: a record of what was typed is not a description of what exists.
Drift is invisible, and nothing verifies the record still matches reality.

### B — CloudFormation
AWS-native, no third-party state. Rejected: single-cloud, and the program
anticipates GCP work. Job descriptions in the target role overwhelmingly
name Terraform.

### C — OpenTofu
MPL-licensed fork of Terraform, functionally equivalent. Rejected for the
same reason B was: the vocabulary that appears in job descriptions and
interviews is Terraform's.

### D — Terraform
Multi-cloud, declarative, the industry default.

## Decision

**Option D**, adopted immediately rather than deferred to the cloud
deployment thread.

Configuration lives in a private `cloud-infra` repository with `aws/` and
`gcp/` top-level directories. The AWS configuration is flat —
`main.tf`, `variables.tf`, `versions.tf` — with no environment directories
and no modules.

Environment separation and module extraction are deferred until a second
environment or a second consumer exists. A module with one caller is
indirection without reuse.

State is stored locally and gitignored. A remote backend is deferred until
more than one machine or person applies changes.

The existing ECR repository, created earlier by CLI, was adopted with an
`import` block rather than destroyed and recreated.

## Consequences

- Infrastructure changes are reviewable as a diff before they are applied.
- Adding an ECR repository is one list entry. The IAM push policy derives
  its resource ARNs from the ECR resource map, so the two cannot drift.
- The state file records resource metadata and, for some resource types,
  secrets in plaintext. It must never be committed. This is the most
  common way credentials leak through Terraform.
- Local state means no locking and no history. A second machine applying
  changes would create divergent state. Accepted while single-machine.
- The AWS provider resolves credentials through the default chain and
  reports no confirmation of which account it selected. A wrong
  environment variable would plan against the wrong account silently. A
  `data.aws_caller_identity` precondition guarding against this is
  deferred until a second account exists.
- Terraform was adopted a thread earlier than planned, so some of the
  cloud deployment thread's budgeted work is already done.

## Verification

`terraform plan` reports no changes against the applied configuration,
confirming that declared state and actual state agree.

Two findings during implementation are worth recording.

First, `thumbprint_list` on the OIDC provider resource is documented as
optional and unused for GitHub, because AWS validates GitHub's certificate
against its own trusted CA library rather than a configured thumbprint.
Most published examples still include a hardcoded thumbprint, and two
different obsolete values circulate.

Second, IAM rejects a web-identity trust policy that constrains neither
the `sub` nor the `job_workflow_ref` claim. An initial policy conditioned
only on the `repository` and `ref` claims — both clean and readable — was
refused with `MalformedPolicyDocument`. GitHub embeds numeric owner and
repository identifiers in the `sub` claim, in the form
`owner@<id>/repo@<id>`, which is absent from published documentation and
caused an exact-match condition to fail. The policy now matches `sub` with
wildcards covering only those numeric identifiers, alongside exact-match
conditions on `repository`, `ref` and `aud`.
