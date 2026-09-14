# 0001 · Account vending stays in Control Tower + CfCT

- Date: 2026-09-12
- Status: accepted (supersedes an earlier proposal to create accounts in Terraform)
- Related: VLAD-4

## Context
Accounts are already vended by AWS Control Tower with the baseline maintained in CloudFormation via CfCT and StackSets. The baseline covers the account in AWS Organizations, SCPs, SSO and IAM Identity Center, and creates no networking.

An earlier option was to create accounts with `aws_organizations_account` so the account and its infrastructure stayed in one Terraform lifecycle.

## Options considered
1. **Terraform-created accounts** — one lifecycle, maintainable in the same workspaces; but re-implements the Control Tower baseline and guardrails, and Terraform cannot close an AWS account anyway.
2. **Keep Control Tower + CfCT** — reuses a working, governed process; requires an explicit handover contract between CloudFormation and Terraform.

## Decision
Keep account vending in Control Tower + CfCT. Terraform starts from an already-vended account.

## Consequences
- Account creation leaves this project's scope.
- The CfCT baseline must deploy a Spacelift execution role to every new account (VLAD-4), with a trust policy scoped to our Spacelift account and least-privilege permissions.
- Ownership boundary must stay explicit: CloudFormation owns the account and guardrails; Terraform owns everything from the VPC upward. Overlap would cause permanent drift.
- Decommissioning still involves manual account closure; documented as a known gap.
