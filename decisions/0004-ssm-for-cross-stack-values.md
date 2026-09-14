# 0004 · Cross-stack values pass through SSM Parameter Store

- Date: 2026-09-12
- Status: accepted
- Related: VLAD-2

## Context
Layers depend on each other's outputs (VPC ID into EKS, subnets into RDS, instance IPs into Ansible). The manual process copies these by hand. Something must carry them automatically.

## Options considered
1. **Spacelift dependency outputs** — least code, visible graph; ties the wiring to Spacelift, and outputs only propagate from a successful apply.
2. **`terraform_remote_state`** — pure Terraform, no extra store; downstream needs read access to the whole upstream state file, which exposes more than intended. Ruled out.
3. **SSM Parameter Store** — an explicit, curated contract readable by Terraform, scripts, applications and people; one more thing to write and keep consistent.

## Decision
Use SSM Parameter Store as the mechanism for passing values between stacks.

## Consequences
- Naming convention `/abt/<env>/<layer>/<key>` becomes an interface; settle it before any layer work (VLAD-2).
- Rules: Terraform-managed only, reference by name not version, a stack writes only its own prefix, publish a curated set rather than every output.
- Secrets never travel as values — publish the reference (ARN or parameter name); secrets stored as SecureString, cheaper than Secrets Manager, with no rotation in v1.
- SSM data sources resolve at **plan time**: a downstream stack that plans before its upstream applies fails with "parameter not found" rather than waiting. Ordering must be enforced by stack dependencies or the root stack.
- Requests with optional layers must not generate stacks that read parameters which will never exist.
