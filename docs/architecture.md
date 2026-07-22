# Reference architecture

Detail behind the summary in the [README](../README.md).

---

## Trust boundary

The architecture has one boundary that matters. Everything else is supporting structure.

On one side: identified clinical data, readable by a small population of data engineers and by the pipeline that transforms it. On the other: de-identified data, readable by a much larger and more fluid population of data scientists, analysts, and potentially external collaborators.

The boundary is enforced four times over, by mechanisms that fail independently:

| Control | Mechanism | What it survives |
|---|---|---|
| Key material | Data scientist and SageMaker roles hold no grant on the PHI CMK | An S3 bucket policy error |
| Identity policy | Explicit `Deny` on PHI buckets and the PHI key in the execution role | A broad managed policy being attached |
| Network | S3 gateway endpoint policy resolves only the de-identified zone | Both of the above being wrong simultaneously |
| Audit | CloudTrail data events on PHI zones, retained ~7 years | Nothing — it is detection, not prevention |

Explicit `Deny` in IAM cannot be overridden by any subsequent `Allow`, which is what makes the second row hold even though the SageMaker execution role carries `AmazonSageMakerFullAccess`. That managed policy is broad by design, and pairing it with a targeted deny is more maintainable than reconstructing an equivalent least-privilege policy that would need updating with every SageMaker feature release.

Exactly one principal crosses the boundary: `DeIdentificationPipelineRole`. It is assumable only by `glue.amazonaws.com` and `states.amazonaws.com`, never by a human identity, and it is the only role in the account holding decrypt on the PHI key and write on the de-identified zone at the same time.

---

## Data flow

### 1. Arrival

Source systems write into the landing zone. Three formats in practice:

- **HL7v2** from the EHR interface engine — ADT (admit/discharge/transfer), ORU (results), ORM (orders)
- **C-CDA** documents for encounter summaries and care transitions
- **FHIR bundles** from systems that already speak it, increasingly common post-Cures Act

Objects land under a prefix scheme of `{source-system}/{message-type}/{ingest-date}/`. The landing zone is append-only by convention: nothing rewrites a raw message, because the raw message is the answer to "what did the source system actually send us" in any future dispute.

The bucket emits to EventBridge on object creation. An EventBridge rule starts the Step Functions pipeline.

**Why EventBridge rather than S3 → Lambda notifications:** S3 bucket notification configuration is a bucket-level property with a limited number of destinations, and adding a second consumer later means modifying the bucket. Routing through the bus means the ECG-adjacent team who wants a copy of ADT messages next quarter subscribes to the bus and never touches the storage layer.

### 2. Normalisation

A Glue job reads raw messages and emits FHIR R4 resources as Parquet into the conformed zone. This is where the genuinely hard clinical data engineering lives — HL7v2 segment mapping, terminology normalisation to LOINC and SNOMED CT, patient matching across source systems, and the long tail of site-specific Z-segments that every interface engine accumulates.

The output is still PHI. Normalisation changes the shape of data, not its sensitivity, and a conformed zone treated as safe is the most common way a "de-identified lake" turns out not to be one.

HealthLake is loaded from the conformed zone and provides the FHIR REST API for applications that need FHIR semantics rather than columnar files.

### 3. De-identification

The boundary crossing. Covered in detail in [ADR 0008](adr/0008-safe-harbor-with-random-surrogate-identifiers.md); in outline:

- Structured identifier columns are dropped or replaced with random surrogates, with the crosswalk written back into the PHI zone
- Dates are removed to year precision; intervals that models need are computed **before** the pass and carried forward as integers
- Ages over 89 are aggregated to `90+`
- Free-text notes go through Comprehend Medical `DetectPHI`, because identifiers in narrative text appear inline and unpredictably

Output lands in the de-identified zone under a different CMK.

### 4. Consumption

SageMaker Studio, in a VPC with no internet path, reads the de-identified zone. Models are registered in a model package group rather than deployed from notebooks, so the provenance question — which data, which code, which approval produced the model now influencing care — has an answer.

Bedrock handles clinical note summarisation, fronted by a guardrail that anonymises identifiers surviving into prompts and blocks prompt-injection attempts against a model with clinical context in scope.

---

## Layer boundaries

The four templates are split by **rate of change**, not by tidiness. Anything crossing a boundary becomes a CloudFormation export, and exports are rigid — a value cannot change while another stack imports it. That rigidity is the cost, so the boundaries are placed where change is least likely.

| Layer | Changes when | Rough cadence |
|---|---|---|
| 01 Foundation | Zone model or retention policy changes | Rarely — measured in years |
| 02 Governance | Personas or audit requirements change | Occasionally |
| 03 Ingestion | Source systems or pipeline logic change | Frequently |
| 04 ML platform | Tooling or network requirements change | Frequently |

Layer 1 is deliberately the most stable thing in the repository. It holds the buckets and keys that everything else references, and it is the layer where a mistake is least recoverable.

---

## Failure modes

Considered explicitly, because an architecture is partly a set of claims about what happens when things break.

**Glue job fails mid-run.** Step Functions catches, transitions to `PipelineFailed`, and the failure message says the de-identified zone may be stale. Job bookmarks mean the next run resumes rather than reprocessing. The important property is that a partial conformed dataset never reaches de-identification — the stages are strictly sequential for that reason, not for simplicity.

**De-identification produces bad output.** The worst case in the architecture: identifiers leaking into the de-identified zone. Versioning on the bucket means the bad objects are recoverable and removable, and CloudTrail records what read them in the interim. Prevention is sampling-based human review of the pipeline output, which is process rather than infrastructure and is not in these templates.

**A key is compromised.** Rotation is enabled, but rotation does not re-encrypt existing objects — it changes the key material used for new encryptions. Genuine compromise means re-encrypting the affected zone, which for the de-identified zone is a bulk copy and for the PHI zones is a longer operation. The two-key split limits the scope of that operation to one side of the boundary.

**A stack is deleted.** Every stateful resource is `Retain` on both delete and replace. Data survives; the stack does not. Teardown is manual, and the README says so.

**HealthLake is unavailable in a required region.** This ends the discussion and forces the self-managed FHIR path in [ADR 0005](adr/0005-healthlake-over-self-managed-fhir.md). Worth checking before anything else is committed to a region.

---

## What this would become at production scale

The reference implementation is single-account, single-region. A production deployment differs in three ways that matter:

**Multi-account.** The de-identified zone belongs in its own account, so the boundary is an account boundary — the strongest isolation AWS offers — rather than an IAM one. Separate accounts per environment, under an Organization with SCPs preventing the disabling of CloudTrail or the deletion of KMS keys. This supersedes much of ADR 0004.

**Multi-region.** No DR story exists here. Cross-region replication of the landing zone at minimum, since it is the system of record from which everything else can be rebuilt. Note that CRR requires a replica key in the destination region and adds cross-region KMS costs.

**Cost guardrails.** Budgets, anomaly detection, and SageMaker instance-type restrictions via SCP or IAM condition. An unconstrained Studio domain is an expensive thing to hand a team, and HealthLake's consumption pricing on a monotonically growing dataset deserves an alarm before it deserves a surprise.
