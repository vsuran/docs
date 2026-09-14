`# Overview — AWS Provisioning Platform (ABT Infrastructure Automation)

Upload this as Project knowledge. Jira (VLAD-1) holds tasks and status; this file holds the stable picture.

## Goal
Provision a complete AWS environment — network, EKS, RDS, Redis, Kafka, Cassandra, EC2 with Ansible — into an already-vended AWS account, from one standardized request, orchestrated by Spacelift. Replaces manual Terraform workspace runs and hand-edited tfvars.

## Definition of done
A live demo: one request provisions a complete environment through one automated workflow, with no manual tfvars edits and no manual workspace runs, and the result is usable for installing the ABT application.

## Scope
**In:** provisioning into an existing account; request → generated Spacelift stacks → layered apply; cross-stack values via SSM; basic failure handling.

**Out of v1:** the ABT application itself; AWS account vending (Control Tower + CfCT); auth and approval in the form; Salesforce integration; automated status reporting; drift remediation; decommissioning automation; incremental "add a layer" requests.

## Architecture summary
```
Request YAML (in Git, from a form or hand-written)
   │  validated against JSON Schema in CI
   ▼
Root stack (own repo + workspace, Spacelift Terraform provider)
   │  creates child stacks, variables, ordering, worker pool
   ▼
Spacelift stacks, layer by layer
   network → EKS / RDS / Redis / Kafka / Cassandra / EC2 → Ansible
   │  each publishes its curated contract to SSM, reads upstream from SSM
   ▼
Complete environment in the target account
```
Boundary: CloudFormation (CfCT) owns the account, SCPs, SSO and IAM Identity Center. Terraform owns everything from the VPC upward. No overlap.

## Milestones
| # | Week | Goal |
|---|---|---|
| M1 | 1 | Chain proven with real modules: network → EKS via SSM |
| M2 | 2 | Layer catalog widened |
| M3 | 3 | Request schema + root stack generating the graph |
| M4 | 4 | Intake, full end-to-end run, demo (and buffer) |

Critical path: VLAD-2 → VLAD-5 → VLAD-6 → VLAD-15 → VLAD-18.

## Constraints
- One person, about one month.
- Terraform code for all resources already exists — layer work is adapting it to publish/consume SSM, not writing modules.
- Spacelift Starter+: no Blueprints; 2 public and 1–2 private workers, so concurrency is capped by licence, not by the dependency graph.
- Only private workers reach private endpoints (EKS API, and anything else on private networking).
- Sandbox account available for testing.

## Risks
| Risk | Impact | Mitigation |
|---|---|---|
| Slow feedback: few workers, long runs | Few end-to-end attempts per day | Test layers in isolation with stubbed SSM values; full chain only at milestones |
| SSM contract churn | Breaks every consumer | Settle the contract (VLAD-2) before layer work |
| Stack dependencies unavailable on Starter+ | Root stack must sequence runs itself; VLAD-15 grows | Confirm in VLAD-3, week 1 |
| Variable topology per environment | Conditional stacks, dangling SSM reads | Encode layer dependency rules in the schema |
| One month, solo | No slack | Week 4 is buffer; cut the form (VLAD-17) first |
| CfCT change needs a vended account to verify | Slow verification loop | Start VLAD-4 in week 1 |

## Open questions
- [ ] Are stack dependencies available on Starter+? (VLAD-3)
- [ ] Which stacks need private-worker network reach beyond EKS?
- [ ] Repo/workspace layout for the Spacelift-managing Terraform
- [ ] Which repo holds request YAML files
- [ ] Which account holds the SSM parameter tree; cross-account read policy

## Links
- Epic: https://vladosur007.atlassian.net/browse/VLAD-1
- Stories: VLAD-2 … VLAD-19
