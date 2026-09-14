# 0002 · Spacelift is the Terraform orchestrator

- Date: 2026-09-12
- Status: accepted
- Related: VLAD-3, VLAD-15

## Context
The manual process runs multiple Terraform workspaces by hand in the right order and copies outputs between them. Something has to own ordering, credentials, and run history.

## Options considered
1. **Custom orchestration** (Lambda/Step Functions driving Terraform) — full control; everything is ours to build and operate.
2. **Spacelift** — stacks, ordering, policies, drift detection, short-lived cloud credentials out of the box.

## Decision
Use Spacelift (Starter+ tier).

## Consequences
- Blueprints are a Business-tier feature and are unavailable, so intake is a request YAML in Git plus a root stack using the Spacelift Terraform provider to generate stacks (see 0005 in Jira scope / VLAD-15).
- Starter+ gives 2 public and 1–2 private workers. Parallelism is capped by licensed concurrency, not by the dependency graph. A full environment run is long, so end-to-end attempts per day are few.
- Only private workers can reach private endpoints (EKS API and similar); stacks must be assigned to pools deliberately.
- Ansible is supported as a stack type but has no outputs, so an Ansible layer can only consume and is always a leaf.
- Whether stack dependencies are available on Starter+ must be confirmed (VLAD-3). If not, the root stack must sequence runs itself.
