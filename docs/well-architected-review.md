# Well-Architected review

A self-assessment against the six pillars, with the Healthcare Industry Lens where it diverges from the general framework.

Self-assessments are worth roughly what they cost, so the useful content here is the **Gaps** sections. A pillar review with no findings has not been conducted.

---

## Operational excellence

**Implemented**

- All infrastructure as version-controlled CloudFormation; no console-created resources in the design
- Static validation on every commit — cfn-lint against 301 rules, Checkov against its CloudFormation policy set — with warnings treated as failures
- Weekly scheduled validation, so newly published rules surface against unchanged code as a build failure rather than as an audit finding
- Dependencies pinned with Dependabot updates, so tool changes arrive as reviewable diffs
- Architecture decisions recorded with their reversal conditions
- Step Functions execution logging with `IncludeExecutionData: false`, since job arguments can carry PHI

**Gaps**

- **No deployment pipeline.** The README documents manual `aws cloudformation deploy` commands. Production needs CodePipeline or GitHub Actions with OIDC, change sets reviewed before execution, and no long-lived deployment credentials.
- **No runbooks.** Nothing documents what to do when the pipeline fails at 2am, or who owns the decision to reprocess.
- **No operational metrics.** No dashboard showing pipeline latency, records processed, or de-identification error rate.

---

## Security

**Implemented**

- Four independent controls on the PHI boundary — key material, identity policy, network reachability, and audit — each of which individually prevents the primary exposure ([architecture.md](architecture.md#trust-boundary))
- Customer-managed keys split by data sensitivity, with annual rotation
- TLS enforced by explicit deny rather than assumed
- Per-object encryption override blocked, closing the gap bucket default encryption leaves open
- ACLs disabled everywhere (`BucketOwnerEnforced`)
- No internet path from the ML environment
- MFA required on human role assumption; `aws:SourceAccount` on every service principal trust policy
- CloudTrail data events on PHI zones with log file validation

**Gaps**

- **No detective controls.** GuardDuty, Security Hub, and Config are all absent. Preventive controls without detective ones assume the preventive controls are correct, which is the assumption most worth testing.
- **No SCPs.** Single-account by design, but a production deployment wants organisation-level guardrails that prevent disabling CloudTrail or scheduling key deletion regardless of account-level permissions.
- **`AmazonSageMakerFullAccess` is broad.** Mitigated by an explicit `Deny` on PHI resources, which cannot be overridden — but a purpose-built policy would be better, at the cost of maintenance with every SageMaker feature release. The trade is deliberate; it is still a trade.
- **No secrets management.** Nothing in these templates needs a secret today. The moment a source-system credential appears, Secrets Manager with rotation does too.

---

## Reliability

**Implemented**

- `DeletionPolicy: Retain` and `UpdateReplacePolicy: Retain` on every stateful resource, both set — the second does not default from the first
- S3 versioning everywhere, with `NewerNoncurrentVersions` guaranteeing a floor of recoverable copies regardless of overwrite rate
- Step Functions retry with exponential backoff on transient Glue failures, and a catch that leaves an explicit failure state
- Two Availability Zones for the ML VPC
- Strictly sequential pipeline stages, so a partial conformed dataset never reaches de-identification

**Gaps**

- **No DR story.** Single region, no cross-region replication. The landing zone at minimum should replicate, since everything downstream can be rebuilt from it.
- **No tested restore.** Versioning provides recovery primitives; nobody has proven a restore works.
- **No dead-letter queue** on the EventBridge rule. A failed pipeline start is currently a silent loss.
- **HealthLake is a single point of failure** for FHIR API consumers, with no documented recovery objective.

---

## Performance efficiency

**Implemented**

- Parquet in the conformed and de-identified zones — columnar layout suits the analytic access pattern
- S3 Bucket Keys enabled, cutting KMS API calls substantially on high-volume reads
- Glue worker type and count parameterised rather than fixed
- Storage classes matched to actual access patterns rather than applied uniformly: Glacier IR for write-once landing data, Intelligent-Tiering where access is bursty and unpredictable

**Gaps**

- **No partitioning strategy documented.** Parquet without sensible partitioning is a full scan with extra steps. Partitioning by encounter date and resource type is the obvious starting point and is not specified anywhere.
- **No compaction.** FHIR normalisation naturally produces many small files, which is the standard way to make a Parquet lake slow.
- **No Athena workgroup configuration**, so no query result location, no data scanned limits, no cost controls on ad-hoc analysis.

---

## Cost optimisation

**Implemented**

- Lifecycle policies on every bucket, differentiated by access pattern
- `AbortIncompleteMultipartUpload` at 7 days on all buckets — failed uploads are invisible in the console and billed regardless
- Noncurrent version expiry, so versioning does not grow without bound
- S3 Bucket Keys reducing KMS request charges
- No NAT gateway, which removes both its hourly charge and its per-GB processing

**Gaps**

- **No budgets or anomaly detection.** This is the most consequential gap in the pillar. HealthLake's consumption pricing on a monotonically growing dataset, and an unconstrained SageMaker Studio domain, are both capable of producing a surprising bill.
- **Ten VPC interface endpoints bill hourly per AZ.** Twenty ENIs' worth of endpoint charges is a real monthly line, and it is the direct cost of the network isolation in [ADR 0007](adr/0007-network-isolation-for-the-ml-environment.md). Worth stating rather than discovering.
- **No SageMaker instance-type restrictions.** Nothing prevents a data scientist launching a `p4d` for exploratory work.
- **No tag-based cost allocation report.** Resources are tagged consistently; nothing consumes those tags.

---

## Sustainability

**Implemented**

- Lifecycle transitions to cold storage reduce the energy footprint of infrequently accessed data
- Managed services (HealthLake, Glue, Bedrock) over self-managed compute, which raises utilisation relative to dedicated infrastructure
- Serverless orchestration with no idle capacity

**Gaps**

- No region selection criteria weighing carbon intensity
- No right-sizing analysis for Glue workers; the default of five G.1X workers is a guess with a comment attached, not a measurement

---

## Healthcare Industry Lens

Where the industry lens asks questions the general framework does not.

| Lens consideration | Position |
|---|---|
| **PHI segmentation** | Zone-separated buckets under different CMKs, with a single auditable crossing point |
| **De-identification methodology** | Safe Harbor, with the re-identification code chosen to satisfy 164.514(c) rather than convention ([ADR 0008](adr/0008-safe-harbor-with-random-surrogate-identifiers.md)) |
| **Interoperability** | FHIR R4 as the conformed representation; HealthLake providing a real FHIR API rather than files that happen to be FHIR-shaped |
| **Clinical model governance** | Model package group with a documented bias-report requirement. **Asserted, not enforced** — nothing blocks a promotion lacking one, and making it a hard gate needs an EventBridge rule on model package state change plus a Lambda that inspects the report |
| **Algorithmic bias** | Clarify evaluation named against age, sex, race, and payer. Payer is included deliberately: it proxies socioeconomic status, and published work on clinical risk algorithms has shown proxies of this kind encoding disparities that appear as legitimate signal |
| **Consent management** | **Not addressed.** Patient consent and research authorisation are the significant gap for any secondary-use scenario beyond operations and quality improvement |
| **Audit trail for clinical decisions** | CloudTrail covers data access. It does not cover which model version influenced which clinical decision — that lineage lives in the model registry and inference logs, and is not wired up here |

### The largest lens gap

Consent. Everything above governs *access* to data. None of it governs whether a given patient's data may be used for a given purpose.

For operations and quality improvement, HIPAA's treatment/payment/operations provisions generally cover secondary use. For research, they generally do not, and the gap between those two sentences is where a real deployment either builds a consent model or restricts its scope. The architecture does not currently express "this cohort excluded patients who declined research use", and adding it means consent status becoming a first-class attribute in the conformed zone and a filter in the de-identification pass.

Worth naming as the next significant piece of work rather than leaving as an assumption.
