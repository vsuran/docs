# 0005 · Intake is a request YAML in Git; a root stack generates the child stacks

- Date: 2026-09-14
- Status: accepted
- Related: VLAD-7, VLAD-14, VLAD-15, VLAD-17

## Context
One standardized request must turn into a complete set of Spacelift stacks — the right layers, in the right order, with the right variables and worker pools — without anyone editing tfvars or creating stacks by hand.

Spacelift Blueprints are the native answer to this and are a Business-tier feature, so they are unavailable on Starter+ (0002). Something else has to carry the request and build the graph.

Environment topology varies: not every environment wants Kafka, Cassandra or EC2. Whatever generates the stacks has to handle optional layers without leaving stacks that read SSM parameters no upstream will ever publish (0004).

## Options considered
1. **Spacelift Blueprints** — native intake and stack templating, nothing to build; requires Business tier. Unavailable.
2. **Hand-maintained stack definitions per environment** — no generator to write; every new environment is manual editing again, which is the process being replaced. Fails the definition of done.
3. **Form posting directly to the Spacelift API** — fewest moving parts between request and stacks; no reviewable artifact, no history of what was requested, and the request exists only as an API call. Ruled out.
4. **Request YAML in Git, plus a root stack using the Spacelift Terraform provider** — the request is a reviewable file with history; generation is ordinary Terraform, testable like any other stack; we own and maintain the generator.

## Decision
A request is a YAML file committed to Git. A root stack — itself a Spacelift stack, running Terraform against the Spacelift provider — reads the request and creates the child stacks, their variables, ordering and worker pool assignment.

The merge request that adds the request file is the approval step. A form (VLAD-17) is a convenience that produces the same YAML and opens the same MR; it is not a second intake path.

## Consequences
- The request YAML is the single input artifact. It must be schema-validated in CI on every MR, before the root stack ever sees it (VLAD-14), because an invalid request that reaches generation fails late and expensively.
- Generation must be deterministic and idempotent: the same request produces the same stacks, and re-running changes nothing. Generated configuration is marked as generated so nobody edits it by hand.
- Optional layers must produce no stacks at all — not disabled stacks — so that no generated stack reads an SSM parameter that will never exist (0004).
- Ordering depends on whether stack dependencies are available on Starter+ (VLAD-3). If they are not, the root stack must sequence the runs itself, which materially increases the scope of VLAD-15.
- The root stack holds Spacelift administrative credentials and can create and destroy every child stack. Its blast radius is the whole environment; its own state and access need to be treated accordingly (0003).
- Teardown must be generated too — destructor resources, in the reverse order of creation — or removing a layer from a request silently orphans infrastructure.
- Where the request files live, and whether the root stack's Terraform lives in the same repo, are still open (D7).
