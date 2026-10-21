# ASL Pro
### Autonomous Security Layer Pro

**Self-healing security for software systems.**

ASL Pro is being designed as an autonomous application-security platform that continuously analyzes code repositories, dependencies, build artifacts, and infrastructure, then proposes, validates, and safely delivers security fixes.

Traditional security tooling often stops at detection. ASL Pro is built around the complete remediation loop:

**Discover → Contextualize → Prioritize → Remediate → Verify → Deliver → Learn**

The objective is not to generate more alerts. It is to reduce exploitable risk and security debt while preserving application correctness, developer control, and a complete audit trail.

> **Project status:** Architecture and build specification. Product capabilities described here are the intended design, not a claim that they are already implemented. The initial release should be deliberately narrower than the long-term platform.

---

## The problem

Modern applications inherit risk from source code, open-source dependencies, containers, infrastructure-as-code, cloud permissions, secrets, and CI/CD configuration. Existing scanners can identify many of these problems, but teams still have to determine which findings matter, implement fixes, test compatibility, and coordinate deployment.

ASL Pro aims to close that operational gap by connecting findings to application context and executing a controlled, evidence-backed remediation workflow.

## Product objectives

- **Continuous security:** scan repositories and relevant artifacts on commits, pull requests, schedules, and vulnerability-intelligence updates.
- **Contextual prioritization:** correlate severity with reachability, exposure, affected assets, ownership, and potential blast radius.
- **Autonomous remediation:** generate minimal, reviewable patches for supported vulnerability classes.
- **Evidence-based verification:** build, test, rescan, and evaluate each candidate patch before it is considered successful.
- **Policy-controlled action:** start with pull requests and explicit approvals; permit more automation only when customer policy and verification evidence allow it.
- **Auditable operation:** preserve the finding, evidence, proposed change, test results, policy decision, actor, and outcome.
- **Deployment flexibility:** design for multi-tenant SaaS first, with a path to single-tenant and private deployments.
- **Developer-first integration:** meet engineers in Git hosting, CI/CD, IDEs, and developer portals instead of forcing every task through a dashboard.

## Initial scope

The first useful release should prove that ASL Pro can securely remediate a narrow set of high-confidence findings in supported repositories.

### Initial capabilities

1. Connect a GitHub repository with least-privilege permissions.
2. Scan source code, dependencies, secrets, and common infrastructure-as-code configurations.
3. Normalize findings into a consistent schema and suppress duplicates.
4. Prioritize findings using severity, confidence, reachability where available, and asset context.
5. Generate a patch for a deliberately limited set of supported issues.
6. Run builds, tests, and targeted security rescans in an isolated, ephemeral environment.
7. Open a pull request containing the patch, rationale, evidence, and verification results.
8. Record the full remediation lifecycle in an audit log.

### Explicitly out of scope for the first release

- Unrestricted autonomous production changes.
- A general-purpose autonomous penetration tester.
- Guaranteed remediation of every vulnerability class.
- Full runtime threat prevention across every cloud.
- Training a proprietary foundation model before product evidence justifies it.
- Active-active, multi-cloud infrastructure on day one.

## System model

`Repository / Cloud → Collect → Analyze → Correlate → Decide → Patch → Verify → Deliver`

The platform is separated into trust boundaries:

1. **Control plane:** tenant configuration, integrations, policies, identity, job orchestration, and audit metadata.
2. **Analysis plane:** scanners, finding normalization, risk correlation, and prioritization.
3. **Execution plane:** isolated workers that inspect untrusted code, create candidate patches, and run builds/tests.
4. **Delivery plane:** pull requests, CI results, notifications, and policy-gated actions.

Model output is untrusted input. A model must never receive unrestricted production credentials or independently authorize its own patch for merge or deployment.

## Proposed technology stack

The stack is a target architecture. Adopt components incrementally and validate operational complexity before introducing distributed infrastructure.

| Area | Proposed technology | Responsibility |
|---|---|---|
| Web application | Next.js, React, TypeScript | Dashboard and administration |
| External API and control plane | Go, REST/OpenAPI; gRPC internally where useful | Tenant APIs, connectors, scheduling |
| Security-critical workers | Rust | Patch handling, parsers, artifact validation |
| AI and evaluation services | Python | Triage, patch synthesis, model evaluation |
| Workflow orchestration | Temporal | Durable scan and remediation workflows |
| Event streaming | Kafka when sustained event volume requires it | Durable event distribution |
| System of record | PostgreSQL | Tenants, repositories, findings, policies, jobs |
| Analytical security events | ClickHouse when volume justifies it | Historical findings and operational analytics |
| Cache and ephemeral coordination | Valkey | Rate limits, cache, short-lived state |
| Artifact storage | S3-compatible object storage | SBOMs, reports, logs, evidence, artifacts |
| Code and dependency analysis | Semgrep, Trivy, ecosystem-native advisory feeds; CodeQL where licensing/access fits | Detection and inventory |
| SBOM | CycloneDX | Component inventory and supply-chain evidence |
| Policy | Open Policy Agent (OPA) | Authorization of workflow decisions |
| Isolation | Ephemeral containers with gVisor; Firecracker microVMs for stronger isolation where needed | Untrusted builds and patch verification |
| Identity and access | OIDC, enterprise SSO/SCIM integration, fine-grained authorization | Authentication and tenant isolation |
| Observability | OpenTelemetry, Prometheus, Grafana; centralized logs and traces | Reliability and diagnosis |
| Deployment | Containers, Kubernetes when needed, OpenTofu, Argo CD | Repeatable infrastructure and delivery |
| Supply-chain integrity | Cosign, in-toto attestations, SLSA-aligned build provenance | Signed releases and verifiable builds |

See [docs/BUILD.md](docs/BUILD.md) for architecture, security controls, implementation phases, and acceptance criteria.

## Security principles

- **Least privilege:** repository and cloud access must be narrowly scoped, revocable, and separately configurable.
- **Tenant isolation:** tenant identity must be enforced in authorization, storage access, jobs, caches, and artifacts.
- **Untrusted repository boundary:** source code, build scripts, dependency hooks, model output, and scanner output must be treated as hostile.
- **No ambient production credentials:** patch workers must not inherit access to customer production environments.
- **Human control by default:** pull-request delivery is the default remediation action; merge and deployment require explicit policy.
- **Verification before delivery:** a generated patch is not a fix until the relevant checks pass and the original finding is re-evaluated.
- **Reproducible evidence:** preserve tool versions, inputs, patch hashes, test results, and policy decisions.
- **Fail closed for high-impact actions:** missing evidence, policy errors, or verification failures must not silently authorize a change.
- **Privacy and residency:** minimize source-code retention, document model data flows, and provide region-aware storage and processing controls as the product matures.

## What success means

ASL Pro should be measured by security outcomes and developer effort, not the raw number of alerts.

- Verified remediation rate.
- Time from detection to verified fix.
- Patch acceptance and merge rate.
- Regression and rollback rate.
- False-positive rate and duplicate-finding rate.
- Percentage of patches with complete evidence.
- Cost per successfully verified remediation.
- Reduction in exploitable risk and overdue security debt.
- Isolation failures and cross-tenant security incidents, with a target of zero.

## Build roadmap

- **Phase 0 — Foundation:** threat model, architecture decisions, repository conventions, CI, secure development baseline.
- **Phase 1 — Repository intake:** GitHub integration, permissions, scan jobs, normalized finding schema.
- **Phase 2 — Detection and prioritization:** source, dependency, secret, and IaC findings; deduplication and context.
- **Phase 3 — Verified remediation:** isolated patch generation, tests, rescanning, evidence bundles, pull requests.
- **Phase 4 — Governance:** policy-as-code, approval tiers, audit export, tenant administration.
- **Phase 5 — Scale and enterprise:** multi-region operations, SSO/SCIM, private deployment, advanced cloud/runtime context.

Each phase must pass its acceptance criteria before expanding automation or blast radius. Detailed milestones are maintained in [docs/BUILD.md](docs/BUILD.md).

## Documentation

- [Build plan and technical architecture](docs/BUILD.md)

## Design principle

**Detect what matters. Fix what can be proven safe. Explain every change.**

ASL Pro is being built around that standard.
