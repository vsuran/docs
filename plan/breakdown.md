# VLAD-1 · AWS Infrastructure End-to-End Automation — breakdown v2

Draft for review. Nothing created in Jira yet.
Supersedes v1: risk-first ordering (real layers before stack generation), Control Tower retained, scope cut to a 1-month solo demo.

## Context that shapes this plan

- **Solo, ~1 month.** Scope is cut to what one person can demo.
- **Terraform code for all resources already exists.** Layer work = adapt to publish/consume SSM, not write modules.
- **Foundations nearly done.** Spacelift AWS integration, S3 backend, IAM role almost complete.
- **Slow feedback loop.** 2 public + 1–2 private workers; a full environment run is long. Layers must be testable in isolation with stubbed SSM values.

## Decisions

| # | Decision | Status |
|---|---|---|
| D1 | Account vending stays in Control Tower + CfCT/StackSets; Terraform starts from an existing account | accepted (supersedes v1 D1) |
| D2 | Spacelift is the Terraform orchestrator (Starter+) | accepted |
| D3 | Terraform state in our own S3 backend, not Spacelift-managed | accepted |
| D4 | Cross-stack values pass through SSM Parameter Store; no `terraform_remote_state` | accepted |
| D5 | Request is a YAML file in Git; a root stack generates the child stacks | accepted |
| D6 | Secrets as SSM SecureString (cheaper than Secrets Manager); no rotation in v1 | proposed |
| D7 | Spacelift-managing Terraform lives in its own repo + workspace | proposed |
| D8 | Intake = local/internal web form that produces the request YAML and opens an MR; no auth, no approval logic (the MR is the approval) | proposed |
| D9 | One request = one complete new environment. No incremental "add a layer" requests | accepted |

## Scope

**In:** provisioning a full environment into an already-vended account, from a YAML request, orchestrated by Spacelift, wired through SSM; basic failure handling; a working demo.

**Out of v1:** account vending; authentication and approval in the form; Salesforce integration; automated status reporting; drift remediation; decommissioning automation; incremental environment changes. Manual paths documented instead.

## Open items (parked, not blocking)

- Exact repo/workspace layout for the Spacelift-managing Terraform (D7)
- Which repo holds request YAML files
- Confirm stack dependencies are available on Starter+ (affects S6 only)
- Which stacks need private-worker network reach beyond EKS

---

# Week 1 · Prove the chain with real modules

Risk first. No throwaway skeleton — the proof uses the real network and EKS modules.

### S1 · SSM parameter contract
The interface every stack publishes. Expensive to change once layers depend on it.
- **Done when:** `/abt/<env>/<layer>/<key>` convention documented; rules agreed (Terraform-managed only, reference by name not version, a stack writes only its own prefix); publisher helper module works; reader pattern documented with an example; decision recorded on SecureString for secrets.
- Tasks: design convention · decide account holding the tree + cross-account read policy · publisher helper module · reader pattern + example · document as the contract

### S2 · Finish foundations
- **Done when:** a stack runs end-to-end in Spacelift with state in our S3 bucket (versioning, encryption, locking, run-role-only access); short-lived credentials, no static keys; Spaces and baseline approval policy in place; **confirmed** whether stack dependencies are available on Starter+.
- Tasks: close out S3 backend + IAM · verify BYO backend (no backend block conflict) · Spaces/role model · confirm tier features · document

### S3 · Spacelift access to newly vended accounts
Touches CfCT, so start early even though it's small.
- **Done when:** a freshly vended account automatically has a Spacelift execution role with a trust policy scoped to our Spacelift account and least-privilege permissions; verified by vending a sandbox account and running a plan against it.
- Tasks: define role + trust policy · add to CfCT StackSet · vend test account · verify run · document

### S4 · Network layer publishes its contract
- **Done when:** existing network module publishes VPC ID, subnet IDs, SG IDs and route info to SSM under its prefix; applies clean in the sandbox; re-run produces no changes.
- Tasks: choose published values (curated, not everything) · add publishing · apply in sandbox · idempotency check · module docs

### S5 · EKS layer consumes from SSM
Hardest consumer first: needs private worker reach and the most inputs.
- **Done when:** EKS stack reads network values from SSM and applies successfully; runs on the private worker pool with API reachable; publishes its own contract (cluster name, endpoint, OIDC issuer); the network → EKS chain runs without manual tfvars edits; behaviour when a parameter is missing is documented (plan-time failure, not a wait).
- Tasks: SSM reader wiring · private worker pool config · apply in sandbox · publish EKS contract · document plan-time resolution gotcha

### S6 · Request schema on paper
Not implemented yet, but layers need to know their input shape.
- **Done when:** draft schema covers all layers with optional layers and inter-layer dependency rules (EKS requires network, Ansible requires EC2); two real past requests expressible in it.
- Tasks: inventory fields from existing Confluence requests · draft schema · encode layer dependency rules · validate against two real examples

---

# Week 2 · Widen the layer catalog

Same shape for every story: *the layer applies from its stack, reads what it needs from SSM, publishes its curated contract, passes policy checks, re-runs clean, and has module docs.*
Each is testable in isolation by stubbing upstream SSM values.

- **S7 · RDS** — sizing and networking from the request; credential handling decided (SecureString vs RDS-managed password); publishes endpoint, port, secret reference — never the secret itself.
- **S8 · Redis (ElastiCache)**
- **S9 · Kafka (MSK)**
- **S10 · Cassandra (Keyspaces)**
- **S11 · EC2 compute** — instance types and counts from the request; publishes instance IDs and private IPs for Ansible.
- **S12 · Ansible configuration** — Ansible stack runs after EC2, builds inventory from SSM. Note: Ansible stacks have no outputs, so this layer consumes only and must be a leaf in the graph.

---

# Week 3 · Generate the environment from a request

### S13 · Request schema implemented and validated
- **Done when:** JSON Schema with types, allowed values, defaults; business rules (allowed regions, sizes, naming, CIDR uniqueness) enforced; invalid requests rejected with readable errors; CI job validates every request file on MR.
- Tasks: implement schema · business rules · CIDR uniqueness check · CI validation job · error message pass

### S14 · Root stack generates the stack graph
The core of the epic.
- **Done when:** a request YAML produces exactly the stacks for the layers requested, with correct variables, ordering and worker pool assignment; absent layers produce no stacks and no dangling SSM reads; re-running changes nothing; generated config is deterministic and marked generated; destructors give ordered teardown.
- Tasks: design request → stacks mapping · implement with the Spacelift Terraform provider · conditional layers · encode ordering (dependencies, or explicit sequencing if unavailable) · destructor resources · test against two sample requests (different topologies)

### S15 · Basic failure handling
Minimum needed for the demo to survive a bad run.
- **Done when:** a mid-chain failure reports clearly which layers applied and which didn't; a resumed run completes without manual state surgery; the same request applied twice creates nothing twice.
- Tasks: failure taxonomy (validation / provisioning / missing dependency) · resume approach · idempotency test · short runbook

---

# Week 4 · Intake, full run, demo

### S16 · Request form
- **Done when:** an internal-only form produces a schema-valid request YAML and opens an MR; it never provisions directly; a rejected input shows a readable error; output is identical in shape to a hand-written request.
- Tasks: generate form from the schema · YAML output · MR creation · local/internal-only hosting · note the Salesforce adapter path for later

### S17 · End-to-end run and demo
- **Done when:** a single request provisions a complete environment through one automated workflow — no manual tfvars edits, no manual workspace runs — and the result is usable for installing the ABT application; a second request with a different topology also succeeds.
- Tasks: full dry run · fix findings · second topology run · demo script · record results

### S18 · Documentation and manual paths
- **Done when:** the published SSM contract, request format, and operating instructions are written up; manual decommissioning (including account closure) is documented as a known gap.
- Tasks: environment documentation · contract reference · decommission runbook · list of deferred items

---

# Critical path and risks

**Critical path:** S1 → S4 → S5 → S14 → S17.

| Risk | Why it hurts | Mitigation |
|---|---|---|
| Slow feedback (few workers, long runs) | Few end-to-end attempts per day | Test layers in isolation with stubbed SSM; full chain only at milestones |
| SSM contract churn | Breaks every consumer | Settle in S1, before layer work |
| Stack dependencies unavailable on Starter+ | S14 grows: root stack must sequence runs | Confirm in S2, week 1 |
| Variable topology | Conditional stacks, dangling reads | Encode layer dependency rules in the schema (S6/S13) |
| One month, solo | No slack | Week 4 is buffer; cut S16 to hand-written YAML if needed |

**First thing to cut if time runs short:** S16 (the form). A hand-written request YAML demonstrates the same automation.
