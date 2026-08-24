# CloudShield AI / DAIL — Resource 3: Technical Architecture & Implementation Plan

**Version:** 1.0  
**Status:** Final Phase 6 technical baseline  
**Use:** New implementation-mentor chat + GitHub engineering documentation

## 1. Executive Technical Summary

CloudShield AI is the broader product context. DAIL (Dependency-Aware Invariant Ledger) is the research core. The technical objective is to build a controlled Terraform remediation system in which an LLM can propose infrastructure changes, but a candidate state cannot become trusted merely because the newest security finding disappears. Previously verified security and functional properties are carried forward as explicit protection obligations; resource identity and supported dependency relationships are analyzed; affected obligations are re-established; and a promotion controller decides whether the candidate can replace the trusted state.

The MVP is a research prototype, not a production-grade CSPM platform. It targets AWS and a controlled Terraform subset focused on VPC, Subnet, Route Table, Route Table Association, Internet Gateway, EC2, Security Groups, Security Group Rules, and RDS. The selected scope supports controlled security and application-to-database reachability scenarios without claiming complete AWS semantics.

The implementation must be built in layers. First implement deterministic DAIL with fixed candidate patches. Then validate identity, dependency, impact, verification, and promotion. Only after the deterministic core is correct should the live LLM be connected. Finally, build the A/B/C experiment harness, validate an independent oracle, run a pilot, freeze the protocol, and run the main experiment.

## 2. Technical Baseline and Status

| Area | Final status |
|---|---|
| Research contribution | Locked: stateful, invariant-aware promotion of iterative LLM-generated Terraform states |
| Product boundary | Locked: CloudShield AI broader product; DAIL narrower research core |
| Candidate State Builder | Locked inside DAIL |
| AWS/Terraform scope | Locked at MVP scope; exact implementation details remain engineering-flexible |
| Invariant lifecycle | Locked logical state model |
| Resource identity | Locked at supported-rule level; conservative under uncertainty |
| Dependency graph | Locked at supported-rule level; not full AWS semantics |
| Impact/invalidation | Locked baseline algorithm; empirically validated during experiments |
| Verification | Provisional at exact implementation level; PASS/FAIL/UNKNOWN contract locked |
| Promotion | Locked baseline acceptance semantics |
| LLM interface | Locked at contract level |
| A/B/C experiment | Locked core design |
| Testing | Locked |
| Evidence pipeline | Locked |
| Implementation readiness | Phase 6 audit passed; ready to build |

Implementation readiness is not empirical proof. The experiment must determine whether DAIL actually reduces regressions and/or verification work.

## 3. System Boundary and Architecture

### CloudShield AI vs DAIL

```text
CloudShield AI
├── Detection / CSPM
├── Context / Risk
├── AI Remediation
└── DAIL Research Core
    ├── Candidate State Builder
    ├── Trusted State Manager
    ├── Protected Invariant State
    ├── Resource Identity Engine
    ├── Dependency Graph Engine
    ├── Change Analyzer
    ├── Change Impact / Invalidation Engine
    ├── Verification Engine
    ├── State-Promotion Controller
    └── DAIL Audit / Research Evidence
```

CloudShield AI can contain product/UI functionality. DAIL is the research engine and should be independently runnable for experiments.

### Trust boundary

```text
LLM
  ↓
UNTRUSTED PATCH
  ↓
Candidate State Builder (inside DAIL)
  ↓
UNTRUSTED CANDIDATE STATE
  ↓
Identity + Dependency + Impact + Verification
  ↓
State-Promotion Controller
  ↓
TRUSTED STATE
```

The LLM never writes trusted state directly.

## 4. End-to-End Processing Flow

```text
Trusted Terraform state
 → security remediation finding
 → LLM candidate patch
 → Candidate State Builder
 → structural checks
 → Change Analyzer
 → Resource Identity
 → Dependency Graph
 → Impact / Invalidation
 → new-resource baseline security checks
 → Verification
 → Promotion Controller
 → PROMOTE / REJECT / RETRY / ESCALATE
 → trusted-state update only after PROMOTE
 → evidence + feedback
```

## 5. Exact MVP Scope

- AWS only.
- Terraform/HCL controlled subset.
- `aws_vpc`
- `aws_subnet`
- `aws_route_table`
- `aws_route_table_association`
- `aws_internet_gateway`
- `aws_instance`
- `aws_security_group`
- `aws_security_group_rule`
- `aws_db_instance`
- Selected security exposure/public-access predicates.
- Selected EC2-to-RDS/network reachability predicates under the supported model.
- Multi-cloud, universal Terraform support, full AWS semantics, Kubernetes, multi-account and cross-region support are out of scope for MVP.

## 6. Terraform Normalized Model

All DAIL components should consume a normalized Terraform model rather than independently interpreting raw HCL.

```text
NormalizedResource
├── resource_id / canonical key
├── terraform_address
├── provider
├── resource_type
├── attributes (normalized)
├── source metadata
└── plan/state metadata

CanonicalSecurityRule
├── parent security-group identity
├── direction
├── protocol
├── from_port / to_port
├── sources / destinations
└── provenance
```

Inline and separate Security Group rules must normalize to one canonical representation.

## 7. Trusted and Candidate State

### Trusted State

```text
TrustedState
├── state_id
├── parent_state_id
├── Terraform reference
├── normalized resources
├── dependency snapshot
├── protected invariant references
├── created_at
└── provenance / software version
```

### Candidate State

```text
S_candidate = ApplyPatch(S_trusted, LLM_patch)
```

The Candidate State Builder is the only component that constructs the candidate from the trusted state and patch. A candidate is untrusted until promoted.

## 8. Invariant Model and Lifecycle

```text
Invariant
├── id / version
├── name / description
├── type: SECURITY | FUNCTIONAL
├── machine-verifiable predicate
├── resource scope
├── dependency scope
├── applicability rules
├── verification policy
├── lifecycle status
├── verification status
├── protection status
├── verified state ID
├── dependency snapshot
├── evidence reference
├── verifier/configuration version
└── provenance
```

Lifecycle:

```text
REGISTERED
   ↓
VERIFYING
   ↓
VERIFIED / PROTECTED
   ↓
AFFECTED
   ↓
REVERIFYING
   ├── PASS → PROTECTED
   ├── FAIL → VIOLATED
   └── UNKNOWN → UNCERTAIN
```

Hard rule:

```text
ProtectionStatus=PROTECTED
→ VerificationStatus=PASS
→ VerifiedStateID=current trusted state
```

Invalidation means prior evidence is no longer sufficient; it does not itself mean the property is false.

## 9. Resource Identity Engine

Identity states:

- `SAME`
- `DIFFERENT`
- `UNCERTAIN`

Change classes:

- `UNCHANGED`
- `MODIFIED`
- `CREATED`
- `DELETED`
- `REPLACED`
- `UNCERTAIN`

Evidence hierarchy:

1. Terraform plan/change semantics.
2. Address + type.
3. Supported stable identifiers.
4. Limited semantic matching.
5. Otherwise `UNCERTAIN`.

Uncertainty must never silently preserve historical evidence.

## 10. Dependency Graph Engine

Three relationship sources are supported:

1. Terraform structural references.
2. Infrastructure-semantic relationships.
3. Invariant-semantic relationships.

```text
G = (V, E)
```

Edges should contain type, provenance/evidence and confidence. The dependency engine supports impact analysis; it does not make promotion decisions.

## 11. Change Impact and Invalidation

For protected invariant `I`:

```text
Affected(I) iff
    Scope_R(I) ∩ ΔR ≠ ∅
    OR
    Scope_G(I) ∩ ΔG ≠ ∅
```

Additional triggers include replacement, deletion, identity uncertainty, dependency uncertainty and supported restructuring.

The output includes:

- affected invariant set;
- reasons;
- confidence/uncertainty;
- verification scope.

### New-resource rule

Applicable baseline security invariants must also be evaluated on newly created in-scope resources, even when no historical invariant is attached to that resource.

## 12. Verification Engine

Layers:

1. Structural — Terraform validate/plan.
2. Security — explicit security predicates.
3. Functional — selected reachability/connectivity properties.
4. Optional formal — SMT/Z3 only when justified.

Results:

- `PASS`
- `FAIL`
- `UNKNOWN`
- `VERIFIER_ERROR`
- `UNSUPPORTED`

`UNKNOWN`, `VERIFIER_ERROR`, and `UNSUPPORTED` cannot silently become `PASS`.

Initial invariants:

- `INV-001`: no public SSH under the defined security predicate.
- `INV-002`: application EC2 retains TCP/5432 reachability to the designated RDS under the supported model.
- `INV-003`: designated RDS satisfies the defined non-public accessibility property.

## 13. State-Promotion Controller

The controller is the final acceptance authority.

```text
PROMOTE iff:
  structural verification passes
  AND new remediation requirement passes
  AND all required affected protected invariants pass
  AND applicable baseline security checks pass
  AND identity/dependency conditions are acceptable
  AND no blocking unresolved uncertainty remains
```

Decisions:

- `PROMOTE`
- `REJECT`
- `RETRY`
- `ESCALATE`

Promotion is transactional. Trusted state, protected-state updates and promotion evidence should commit together. A rejected candidate never overwrites trusted state.

## 14. LLM Remediation Interface

The LLM is a proposal generator.

```text
RemediationRequest
├── finding
├── trusted Terraform context
├── allowed/supported scope
├── previous feedback (retry only)
├── iteration number
└── generation configuration

LLMResponse
├── patch
├── target-resource hint (advisory)
└── intent/summary
```

Target-resource hints are not authoritative; actual changed resources are computed by DAIL.

On rejection, structured evidence should include what passed, which invariant failed, affected resources, failure evidence and the property that must be preserved. Retries start from the last trusted state.

## 15. Experiment Harness

Three conditions:

| Condition | Persistent protection | Verification |
|---|---|---|
| A — Stateless | No | Current/new requirement |
| B — Stateful Full | Yes | All protected invariants |
| C — DAIL | Yes | Affected protected obligations + required baseline security checks |

A/B/C goals:

- **A vs B:** value of persistent state.
- **B vs C:** value of dependency-aware selective verification.
- **A vs C:** overall DAIL effect.

The benchmark must contain scenarios where the affected set is meaningfully smaller than the full protected set, otherwise H2 cannot be tested.

## 16. Independent Ground-Truth Oracle

The oracle is separate from the DAIL promotion path.

```text
Candidate
  ├── DAIL path → decision
  └── independent oracle path → actual property outcome
```

The oracle should use a separately implemented evaluation path and, for selected functional scenarios, independent runtime validation in a controlled AWS environment.

## 17. Test Strategy

Layers:

- schema/configuration;
- unit;
- integration;
- end-to-end;
- adversarial;
- research-readiness.

Mandatory safety properties:

- failed protected invariants never promote;
- unknown never silently becomes pass;
- rejected candidates never overwrite trusted state;
- promotion requires evidence;
- identity/dependency uncertainty cannot silently preserve old evidence;
- new resources receive applicable baseline security checks;
- LLM cannot directly alter trusted state.

Canonical must-pass unsafe case:

```text
Security requirement → PASS
Functional invariant → FAIL
Promotion → REJECT
```

Canonical safe case:

```text
Security requirement → PASS
Functional invariant → PASS
Promotion → PROMOTE
```

## 18. Research Logging and Evidence Pipeline

```text
Paper claim
 → metric
 → processed dataset
 → run
 → transition
 → patch
 → candidate state
 → identity
 → dependency
 → impact
 → verification evidence
 → promotion decision
 → independent oracle
```

Raw evidence should be immutable. Processed datasets are derived separately.

## 19. Data and Storage Architecture

Use one logical research database for:

- states;
- invariant definitions and versions;
- invariant state;
- resources;
- dependency edges;
- verification runs;
- promotion records;
- experiment runs;
- events.

Use artifact storage for large files such as Terraform sources, patches, plans, raw LLM responses and verifier outputs.

## 20. Recommended GitHub Repository Structure

```text
cloudshield-ai/
├── README.md
├── docs/
│   ├── research/
│   │   ├── resource-1-project-handover.md
│   │   └── resource-3-technical-architecture.md
│   ├── architecture/
│   ├── decisions/
│   └── experiments/
├── dail/
│   ├── state/
│   ├── invariants/
│   ├── terraform_model/
│   ├── identity/
│   ├── dependency/
│   ├── impact/
│   ├── verification/
│   ├── promotion/
│   ├── llm/
│   └── evidence/
├── experiments/
│   ├── scenarios/
│   ├── baselines/
│   ├── oracle/
│   ├── harness/
│   └── analysis/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   ├── adversarial/
│   └── regression/
├── terraform/
│   ├── fixtures/
│   └── modules/
├── scripts/
└── configs/
```

Secrets and private cloud data must never be committed.

## 21. Implementation Roadmap

1. Project foundation + normalized Terraform model.
2. Trusted State + Invariant Store.
3. Resource Identity.
4. Dependency Graph.
5. Impact / Invalidation.
6. Verification.
7. Promotion Controller.
8. Fixed-patch end-to-end DAIL.
9. LLM integration.
10. Research logging + independent oracle.
11. Experiment harness.
12. Full test suite.
13. Pilot.
14. Protocol freeze.
15. Main experiment.

## 22. First Coding Objective

Do not begin with the live LLM.

Build:

```text
Trusted Terraform
 +
Fixed candidate patch
 ↓
Candidate State Builder
 ↓
Identity
 ↓
Dependency
 ↓
Impact
 ↓
Verification
 ↓
Promotion
```

The first complete demonstration must show:

- a candidate that fixes public SSH but breaks App→RDS:5432 is rejected;
- a candidate that fixes public SSH and preserves App→RDS:5432 is promoted.

## 23. Research Metrics

- Regression Rate
- Missed Regression Rate
- False Rejection Rate
- Verification Work
- Verification Savings
- Repair Success Rate
- Iterations to Convergence
- Identity Error Rate
- Resource Churn
- Unsafe Promotion Rate

Raw event data must be collected before computing summary metrics.

## 24. Engineering Rules

- Do not redesign DAIL without an explicitly justified implementation contradiction.
- Keep the research contribution stable.
- Build deterministic core before live LLM integration.
- Treat LLM output as untrusted input.
- Never allow a rejected candidate to become trusted.
- Keep PASS, FAIL, UNKNOWN, VERIFIER_ERROR and UNSUPPORTED distinct.
- Handle identity/dependency uncertainty conservatively.
- Do not claim full AWS/Terraform semantics.
- Keep product/UI features outside the research MVP unless required for the experiment.
- Create regression tests for discovered bugs.
- Record deviations from this plan.
- Do not optimize before correctness and ground truth are validated.
- Do not treat implementation success as empirical research success.

## 25. Change-Control Rules

| Change | Rule |
|---|---|
| Engineering implementation choice | Flexible if research semantics remain unchanged |
| MVP scope change | Must be documented and reviewed |
| Research contribution change | Stop and return to audit/research review |
| Experiment baseline change | Document before main experiment; never silently change after results |
| Invariant change | Version it; preserve historical evidence |
| Verifier change | Version it and rerun affected validation tests |
| Promotion policy change | Safety-test before further experiments |

## 26. Implementation Readiness Gate

Before main experiments, verify:

- Candidate State Builder is inside DAIL.
- Supported AWS/Terraform scope is frozen.
- Invariant state machine is legal and testable.
- Identity/dependency uncertainty is conservative.
- Impact separates AFFECTED from VIOLATED.
- Verification is evidence-backed.
- New-resource security checks exist.
- Promotion is deterministic and transactional.
- Independent oracle exists.
- A/B/C baselines are fair.
- Evidence traces from raw event to paper metric.
- Unit/integration/e2e/adversarial/pilot gates pass.

## 27. Final Technical Position

Phase 6 converted the research idea into an implementation-ready blueprint. The next objective is to build the deterministic DAIL core with fixed Terraform candidates, validate it, integrate the LLM, and then run controlled experimentation. Any implementation issue that changes the research semantics or experimental treatment must be escalated to the audit/research review before proceeding.
