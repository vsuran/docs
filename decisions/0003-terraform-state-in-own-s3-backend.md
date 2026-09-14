# 0003 · Terraform state in our own S3 backend

- Date: 2026-09-12
- Status: accepted
- Related: VLAD-3

## Context
Spacelift can manage Terraform state itself, but we want state to remain ours and usable independently of the orchestrator, since these workspaces are maintained long after provisioning.

## Options considered
1. **Spacelift-managed state** — no setup, state history and rollback in the UI; state lives with the orchestrator.
2. **Own S3 backend** — portable, independent of Spacelift; we own the bucket, locking, encryption and access.

## Decision
Bring our own S3 backend.

## Consequences
- The two are mutually exclusive: with a backend block in the code, Spacelift-managed state fails to initialise. The BYO path must be verified explicitly.
- We lose Spacelift's state history and rollback UI.
- The bucket needs versioning, encryption, blocked public access, working locking, and IAM that only the run role can use.
- Key convention: `<env>/<layer>/terraform.tfstate`.
