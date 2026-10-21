# ASL Pro Build Specification

This document defines the intended architecture and implementation sequence for ASL Pro (Autonomous Security Layer Pro). It is a build specification, not evidence that every listed component or capability is already implemented.

## 1. Product contract

ASL Pro continuously examines software assets, identifies meaningful security risk, and automates verified remediation within customer-defined boundaries.

The product's primary unit of value is a **verified remediation**: a finding with traceable evidence, a justified fix, passing required checks, and a recorded delivery outcome.

### Operating loop

1. **Discover:** ingest repository, dependency, artifact, and later cloud/runtime signals.
2. **Normalize:** convert tool-specific output into a stable internal finding model.
3. **Correlate:** link findings to repositories, packages, services, owners, environments, and exposure context.
4. **Prioritize:** calculate risk using severity, confidence, reachability, exposure, exploit intelligence, and business context.
5. **Plan:** select an approved remediation strategy and define required checks.
6. **Patch:** produce the smallest justified change in an isolated worker.
7. **Verify:** build, test, rescan, inspect the diff, and check policy.
8. **Deliver:** open a pull request or request approval for a higher-impact action.
9. **Record:** store evidence, decisions, versions, and outcomes.
10. **Learn:** use accepted/rejected patches and verification outcomes to improve ranking and evaluation, without allowing feedback to bypass controls.

## 2. Architecture and trust boundaries

### A. Control plane

Responsibilities:
- tenant, user, organization, repository, and integration management
- authentication and authorization
- policy configuration and approval thresholds
- workflow initiation and scheduling
- job status and customer-facing APIs
- audit metadata and usage accounting

Suggested implementation: Go services, PostgreSQL, OIDC, REST/OpenAPI, and Temporal.

The control plane should not execute customer build scripts or run arbitrary repository code.

### B. Analysis plane

Responsibilities:
- scanner orchestration and tool-version management
- source, dependency, secret, container, and IaC analysis
- result normalization, deduplication, and suppression
- SBOM processing and vulnerability intelligence ingestion
- ownership and asset correlation
- risk ranking and remediation eligibility

Use existing scanners as replaceable analysis engines. ASL Pro's differentiation is correlation, prioritization, safe execution, and outcome measurement, not rebuilding every scanner.

### C. Remediation execution plane

Responsibilities:
- checkout a pinned repository revision
- reproduce the relevant finding where practical
- prepare a candidate patch
- inspect and constrain the diff
- run builds, tests, and security rescans
- emit a signed or integrity-protected evidence bundle

Treat every repository and build as hostile. Use disposable workers, resource quotas, restricted egress, non-root execution, read-only base images where practical, and no customer production credentials. Start with hardened ephemeral containers; use gVisor or Firecracker microVMs where the threat model requires stronger isolation. Do not run arbitrary pull-request code on privileged shared runners.

### D. Delivery plane

Responsibilities:
- create pull requests with clear explanations and evidence
- attach check results and links to findings
- request human approval where policy requires it
- report merge, closure, failure, and rollback outcomes
- integrate with CI providers and notifications

The default action is to open a pull request, not to merge or deploy automatically.

### Logical flow

`Git provider / artifact / cloud signal`
→ `connector and intake`
→ `durable workflow`
→ `scanner adapters`
→ `normalized findings + asset context`
→ `risk and policy decision`
→ `isolated patch worker`
→ `build + tests + rescans`
→ `policy gate`
→ `pull request / approval`
→ `audit and outcome analytics`

## 3. Technology choices

The following are proposed defaults, not mandatory dependencies to deploy all at once.

| Layer | Default | Notes |
|---|---|---|
| Web console | Next.js, React, TypeScript | Dashboard, repository setup, findings, policies |
| Control-plane services | Go | API, connectors, schedulers, tenancy-aware services |
| Security-critical components | Rust | Diff validation, artifact processing, selected high-assurance workers |
| AI and evaluation | Python | Triage, patch generation, evaluation harnesses |
| API contracts | REST/OpenAPI externally; gRPC selectively internally | Prefer operational simplicity until service boundaries justify RPC complexity |
| Durable workflows | Temporal | Retries, timeouts, compensation, resumability, human approval waits |
| Event streaming | Kafka at demonstrated scale | Introduce only when independent consumers and sustained throughput justify operating it |
| Primary database | PostgreSQL | Source of truth for tenants, integrations, findings, policies, jobs, audit metadata |
| Cache | Valkey | Short-lived cache, rate limits, deduplication coordination; not authoritative storage |
| Analytics | ClickHouse when event volume warrants it | High-volume findings, scan history, and outcome analytics |
| Object storage | S3-compatible object store | Reports, SBOMs, logs, evidence bundles, build artifacts with retention controls |
| Code scanning | Semgrep; CodeQL where appropriate | Custom/fast rules and deeper semantic analysis |
| Dependency, image, IaC, secret scanning | Trivy plus ecosystem advisory feeds | Pin versions and normalize outputs |
| SBOM | CycloneDX | Software component inventory |
| Policy engine | OPA | Central policy decisions for remediation and delivery |
| Workload identity | Cloud-native workload identity; SPIFFE/SPIRE if multi-environment needs justify it | Avoid static shared service credentials |
| Sandbox | Ephemeral containers initially; gVisor/Firecracker according to threat model | Per-job isolation, limits, cleanup, restricted network |
| Telemetry | OpenTelemetry, Prometheus, Grafana, centralized logs | Trace every remediation workflow with tenant-safe attributes |
| Infrastructure | OpenTofu; managed cloud services initially | Keep modules reproducible and secrets out of state/logs |
| Deployment | Managed containers first; Kubernetes as operational requirements justify it; Argo CD for GitOps when applicable | Do not start with a large cluster platform before product-market evidence |
| Artifact integrity | Cosign, in-toto, SLSA-aligned provenance | Sign release artifacts and preserve build provenance |

### Language boundaries

- **Go:** APIs, connectors, workflow activities, orchestration services, CLI.
- **Rust:** components requiring strict memory-safety properties around complex or untrusted inputs.
- **Python:** model orchestration, patch synthesis, offline evaluation, ranking experiments.
- **TypeScript:** web console and developer-facing extensions.

Do not split the codebase into microservices solely because several languages are used. Begin with a modular repository and explicit package/service boundaries. Extract independently deployed services when isolation, scaling, or team ownership requires it.

## 4. Data model

Establish stable identifiers and versioned schemas before broad integration work.

Core entities:
- `Tenant`, `User`, `Membership`
- `Repository`, `Integration`, `CredentialReference`
- `Asset`, `Package`, `Artifact`, `SBOM`
- `ScanRun`, `Finding`, `FindingOccurrence`
- `RiskAssessment`, `RemediationPlan`, `PatchCandidate`
- `VerificationRun`, `VerificationCheck`, `EvidenceArtifact`
- `Policy`, `PolicyDecision`, `Approval`
- `PullRequest`, `RemediationOutcome`, `AuditEvent`

A finding should include:
- stable internal ID and source-tool fingerprint
- tenant and asset/repository references
- rule ID, category, severity, confidence, and status
- affected file/line or package coordinates
- affected and fixed version ranges where applicable
- evidence references and scanner/tool version
- first-seen, last-seen, and resolved timestamps
- links to risk assessments and remediation attempts

Store large reports and artifacts in object storage; keep metadata and access policy in PostgreSQL. Avoid putting secrets, full source files, or sensitive raw payloads into logs or analytics by default.

## 5. Finding normalization and risk

### Normalization

Create scanner adapters that convert tool outputs to a versioned internal schema. Preserve source-specific fields in a bounded, structured payload so the normalized model does not erase useful evidence.

Deduplicate by stable source fingerprints and normalized location/package coordinates. Do not merge findings across tenants. Track finding lifecycle explicitly: `open`, `triaged`, `planned`, `in_progress`, `verification_failed`, `remediated`, `accepted_risk`, and `reopened`.

### Prioritization

Start with transparent, deterministic rules. A future model may assist, but must not obscure the reason for priority.

Signals may include:
- CVSS or source severity
- exploit availability and trusted threat intelligence
- reachability and whether the vulnerable code path is used
- internet exposure and deployment context
- asset criticality and ownership
- confidence and evidence quality
- availability of a vendor-supported fixed version
- change risk and expected regression cost

Keep **security risk** distinct from **patch confidence**. A critical issue can have high security risk but low-confidence automation eligibility; it should be escalated rather than patched recklessly.

## 6. Autonomous remediation design

### Remediation strategy order

Prefer deterministic, maintainable fixes before unconstrained code generation:

1. Vendor-supported dependency upgrade within compatible constraints.
2. Configuration correction based on a known secure baseline.
3. Rule-backed source transformation for a well-understood pattern.
4. Model-generated patch for supported classes, constrained by the finding and repository context.
5. Human escalation when the fix is ambiguous or the change is high impact.

### Patch workflow

1. Pin the source revision and capture environment metadata.
2. Confirm the finding still applies to that revision.
3. Build a remediation plan with explicit allowed files and required checks.
4. Generate one or more candidate patches.
5. Reject patches that modify unrelated files, add suspicious scripts, weaken tests, disable security controls, or change protected workflow/configuration files without explicit policy.
6. Apply the patch in a disposable sandbox.
7. Run formatting, static checks, build, relevant unit/integration tests, and targeted security rescans.
8. Confirm the original finding is resolved and no new high-confidence findings were introduced.
9. Evaluate test coverage and patch scope; a green build alone is insufficient.
10. Run policy evaluation and request required approvals.
11. Open a pull request with a concise rationale, exact diff, test evidence, residual risk, and rollback guidance.
12. Observe the outcome and record acceptance, rejection, regression, or rollback.

### Model safety

- Treat repository text, issue descriptions, comments, dependencies, scanner output, and generated code as untrusted data, not system instructions.
- Use structured tool interfaces and explicit per-task permissions.
- Do not expose production secrets or broad cloud credentials to models or patch workers.
- Restrict outbound network access in execution sandboxes.
- Require deterministic policy checks outside the model.
- Preserve prompts/model identifiers only where appropriate and safe; redact sensitive inputs and define retention.
- Evaluate prompt-injection resistance, malicious repository behavior, patch regressions, and unsafe dependency changes.
- Never let the model authorize its own merge, raise its own permissions, or bypass failed verification.

## 7. Autonomy and approval tiers

Implement explicit action tiers rather than one global autonomous mode.

- **Tier 0: Observe.** Scan and report only.
- **Tier 1: Recommend.** Explain a fix without modifying the repository.
- **Tier 2: Prepare.** Generate and verify a patch, but require approval to open or update a PR.
- **Tier 3: PR automation.** Open/update pull requests automatically after all configured checks pass.
- **Tier 4: Policy-approved merge.** Only for narrowly defined, low-risk change classes with repository-owner opt-in, required CI checks, branch protection, and complete audit evidence.
- **Production changes:** separate capability requiring dedicated design, explicit customer authorization, scoped credentials, change controls, health verification, and rollback. Not part of the initial release.

Default new tenants to Tier 0 or Tier 1. Require explicit opt-in for higher tiers. A policy denial, timeout, missing evidence, or failed check must not silently promote a job to a more permissive tier.

## 8. Multi-tenancy, identity, and privacy

- Authenticate users through OIDC; add enterprise SSO and SCIM as commercial requirements demand.
- Authorize every request and background job against tenant and resource scope.
- Scope integration credentials to the minimum repository permissions required.
- Store secrets in a managed secret manager; store only references and metadata in the application database.
- Encrypt data in transit and at rest; rotate credentials and signing keys.
- Use per-tenant authorization checks for object storage and artifact retrieval.
- Apply quotas and fair scheduling to prevent noisy-neighbor exhaustion.
- Establish configurable retention and deletion policies for source-derived artifacts.
- Avoid training on customer code by default. Any future use for training must be explicit, contractually clear, and technically isolated.
- Design regional processing and residency boundaries before selling a regional guarantee.
- Maintain tested incident response, backup, restore, and key-compromise procedures.

## 9. API and integration strategy

### Initial integration
- GitHub App with least-privilege permissions.
- Webhooks for push, pull request, repository, and relevant security events.
- Scheduled scans and manual scan triggers.
- Pull request creation with evidence and status checks.

### Later integrations
- GitLab and Bitbucket.
- CI systems and developer portals such as Backstage.
- Cloud inventory and configuration providers.
- Container registries and Kubernetes.
- IDE extension and CLI enhancements.

Make integrations idempotent. Validate webhook signatures, deduplicate event deliveries, enforce replay protection where applicable, and treat provider payloads as untrusted. Provide clear connector permission descriptions and an immediate revocation path.

## 10. Reliability and observability

Every scan and remediation workflow should have a traceable correlation ID and state transitions.

Measure:
- scan queue delay and completion latency
- scanner error rate and tool timeout rate
- workflow retry and abandonment rate
- patch verification pass rate
- verified remediation rate
- false-positive and duplicate rates
- pull-request acceptance and merge rates
- regression and rollback rates
- cost per verified remediation
- tenant-level resource usage and quota violations

Use OpenTelemetry for traces and metrics instrumentation. Redact tokens, secrets, source snippets, and sensitive customer metadata before telemetry export. Define service-level objectives only after baseline behavior is measured.

## 11. Supply-chain and secure development

For ASL Pro's own development and release process:
- require code review and protected branches
- run tests, linting, SAST, dependency checks, secret scans, and IaC scans in CI
- generate SBOMs for shipped artifacts
- pin build dependencies and use reproducible builds where practical
- sign release artifacts and publish provenance attestations
- protect signing keys and CI identity with short-lived credentials
- run threat modeling for new privileged features
- maintain a vulnerability disclosure and security-response process
- align the development process with NIST SSDF and relevant OpenSSF guidance

The platform must be held to a higher security standard than the systems it scans.

## 12. Global deployment strategy

### Launch architecture
Start in one primary cloud and one region using managed services where they reduce operational burden. Keep application components portable, but do not pay the cost of active-active multi-cloud before customer demand requires it.

### Regional expansion
Add regional deployment units with explicit data placement, telemetry routing, key management, backup, and disaster-recovery rules. Determine which data may cross regions and which must remain local.

### Enterprise deployment options
1. **Multi-tenant SaaS:** default commercial product.
2. **Single-tenant VPC/VNet:** isolated control/data plane for larger customers.
3. **Private or air-gapped deployment:** later offering with a documented update, licensing, support, and model-hosting strategy.

Do not promise air-gapped or residency guarantees until tested deployment procedures and operational controls exist.

## 13. Phased implementation plan

### Phase 0 — Architecture and secure foundation

Deliver:
- product threat model and trust-boundary diagram
- architecture decision records for language, data, workflow, and sandbox choices
- repository structure and development conventions
- CI with formatting, tests, dependency scanning, and secret detection
- initial data schema and API conventions
- local development environment and reproducible setup

Acceptance:
- a new developer can run the documented checks from a clean checkout
- secrets are not required in source control
- threat model identifies repository execution, tenant isolation, credential handling, and model input as core risks

### Phase 1 — Repository intake and scan orchestration

Deliver:
- GitHub App installation and permission model
- signature-validated webhooks
- manual and scheduled scan jobs
- first scanner adapters, starting with Trivy and Semgrep
- versioned finding schema and PostgreSQL persistence
- minimal dashboard or CLI to inspect findings

Acceptance:
- duplicate webhook deliveries do not create duplicate jobs
- scan failures are visible and retryable
- findings link to a pinned revision and source evidence
- tenant scoping is tested at API and data-access boundaries

### Phase 2 — Finding quality and prioritization

Deliver:
- deduplication and lifecycle states
- dependency/SBOM ingestion using CycloneDX
- severity/confidence model with explainable ranking
- asset and repository ownership metadata
- baseline reporting and finding suppression with audit trail

Acceptance:
- priority decisions expose their contributing signals
- suppression cannot erase historical evidence
- metrics establish baseline false-positive, duplicate, and scan-completion rates

### Phase 3 — Verified pull-request remediation

Deliver:
- isolated ephemeral execution workers
- constrained patch-generation interface
- diff policy and protected-file checks
- build/test/rescan pipeline
- verification evidence bundle
- automated pull-request creation
- audit record for each transition

Acceptance:
- no sandbox has customer production credentials
- a failed test or rescan prevents the patch from being marked verified
- patch output is reproducible from a pinned source revision and recorded inputs where feasible
- malicious repository fixtures cannot access host credentials or other tenant artifacts
- every PR identifies what changed, why, and which checks passed

### Phase 4 — Policy and enterprise governance

Deliver:
- OPA policy integration
- autonomy tiers and approval requirements
- role and repository-level authorization
- audit export and retention controls
- SSO/SCIM based on customer demand
- tenant quotas and usage reporting

Acceptance:
- policy decisions are deterministic and auditable
- high-impact actions require explicit policy authorization
- denied or failed checks cannot fall back to a more permissive path

### Phase 5 — Scale, cloud context, and private deployments

Deliver only as validated by demand:
- Kafka for independently consumed high-volume events
- ClickHouse for analytical workloads
- multi-region deployment and disaster recovery
- cloud configuration and runtime signal correlation
- single-tenant and private deployment packaging
- additional Git providers and developer portal integrations

Acceptance:
- load, failure-recovery, restore, and regional isolation tests meet published targets
- data residency claims are validated against logs, backups, telemetry, and model-provider flows
- operating cost per verified remediation is measured

## 14. Initial repository organization

Prefer a modular monorepo until independent deployment boundaries are justified.

```text
aslpro/
├── apps/
│   └── web/                    # Next.js console
├── services/
│   ├── api/                    # Go control-plane API
│   ├── connector-github/       # GitHub App and webhook intake
│   ├── scanner-orchestrator/   # Scanner execution and normalization
│   ├── risk-engine/            # Deterministic risk ranking
│   ├── remediation/            # Patch planning and workflow activities
│   └── ai-worker/              # Python model/evaluation boundary
├── workers/
│   └── sandbox/                # Disposable build/test execution
├── packages/
│   ├── finding-schema/
│   ├── policy/
│   └── api-contracts/
├── deploy/
│   ├── containers/
│   ├── opentofu/
│   └── kubernetes/
├── docs/
│   ├── BUILD.md
│   ├── THREAT_MODEL.md
│   ├── SECURITY.md
│   └── adr/
├── tests/
│   ├── fixtures/
│   ├── integration/
│   ├── security/
│   └── evaluation/
└── README.md
```

This is a target layout, not a requirement to create empty directories immediately. Add folders with the first implementation that needs them.

## 15. Testing and evaluation

### Software correctness
- unit tests for schema, policy, normalization, and diff validation
- integration tests for provider webhooks and database transactions
- end-to-end tests for scan-to-PR workflows
- migration and backup/restore tests
- load and failure-injection tests as the service matures

### Security tests
- cross-tenant access attempts
- malicious repository and build-script fixtures
- prompt-injection content embedded in source and documentation
- secret exfiltration and network-egress tests in sandboxes
- patch attempts to weaken tests, remove security checks, or modify protected CI files
- replayed or forged webhook requests
- dependency confusion and malicious package scenarios
- policy bypass and fail-open behavior

### Remediation quality benchmark
Maintain a fixed, versioned evaluation set of real or safely constructed vulnerable fixtures. For each case, record:
- vulnerability class and ground truth
- expected remediation properties
- generated diff
- test and rescan results
- regression checks
- whether the patch is minimal and maintainable
- whether a human accepted the result
- compute/model cost and elapsed time

Optimize for **cost per verified, accepted remediation**, not patches generated per minute. Prevent benchmark leakage and manually inspect a sample of successful outcomes.

## 16. Key architecture decisions

1. Existing scanners are adapters, not the product's identity.
2. PostgreSQL is the initial system of record; add specialized stores only for demonstrated workloads.
3. Temporal manages durable, long-running remediation workflows.
4. Model output is a candidate, never an authorization decision.
5. The default delivery mechanism is a pull request.
6. Sandbox isolation is a launch requirement for running customer code.
7. Start with one cloud and region; expand from measured demand.
8. Use deterministic risk and policy controls before introducing learned decisions.
9. Track verified outcomes and regressions from the beginning.
10. Keep the first release narrow enough to validate end-to-end safety.

## 17. Open decisions to resolve before implementation

- Supported languages and package ecosystems for the first remediation classes.
- Initial cloud provider and deployment region.
- Exact GitHub App permission set and data-retention defaults.
- Supported model providers and whether private inference is required at launch.
- Sandbox technology based on threat model and operational budget.
- CI, test, and coverage requirements for each remediation class.
- Licensing and deployment constraints for each scanner and dependency.
- Commercial boundaries between SaaS, single-tenant, and private editions.

Resolve these through short Architecture Decision Records (ADRs). Avoid treating every candidate technology in this document as a launch dependency.

## Definition of a successful first release

A user connects a repository, ASL Pro detects a supported vulnerability, generates a constrained patch, verifies the change in an isolated environment, re-scans the result, and opens a pull request containing understandable evidence. A failed check blocks delivery. The complete process is auditable and tenant-isolated.

That end-to-end proof is the foundation on which broader code-to-cloud coverage and higher autonomy can safely be built.
