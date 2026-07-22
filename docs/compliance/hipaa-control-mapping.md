# HIPAA Security Rule control mapping

Maps applicable 45 CFR Part 164 Subpart C citations to the specific resource implementing them, and states plainly what this architecture does **not** address.

**This is not a compliance certification.** It is an architect's mapping, intended as input to a review by people qualified to sign off on it. A HIPAA compliance position is an organisational determination involving administrative and physical safeguards, workforce training, executed business associate agreements, and risk analysis — most of which no CloudFormation template touches.

**Precondition.** Everything below assumes an executed AWS Business Associate Addendum. Without one, the HIPAA-eligible status of the services used here is irrelevant, because AWS is not a business associate and the arrangement does not satisfy 164.502(e) regardless of technical configuration.

---

## Technical safeguards — 164.312

| Citation | Requirement | Type | Implementation |
|---|---|---|---|
| **164.312(a)(1)** | Access control | Required | Persona-scoped IAM roles (`DataEngineerRole`, `DataScientistRole`, `DeIdentificationPipelineRole`); Lake Formation LF-Tag grants; `PublicAccessBlockConfiguration` on every bucket; S3 gateway endpoint policy resolving only the de-identified zone |
| **164.312(a)(2)(i)** | Unique user identification | Required | IAM roles assumed through federation; no shared credentials; no IAM users provisioned by these templates |
| **164.312(a)(2)(ii)** | Emergency access procedure | Required | **Not addressed.** Break-glass access is an organisational procedure. The KMS root delegation in each key policy is what makes a break-glass role technically possible, but the role, its approval workflow, and its alarming are out of scope here |
| **164.312(a)(2)(iii)** | Automatic logoff | Addressable | **Partial.** SageMaker Studio idle shutdown is configured through lifecycle configurations, not present in these templates |
| **164.312(a)(2)(iv)** | Encryption and decryption | Addressable | SSE-KMS with customer-managed keys on every bucket; `DenyNonKmsEncryptionHeader` and `DenyIncorrectKmsKey` bucket policy statements preventing per-object override; HealthLake `CUSTOMER_MANAGED_KMS_KEY`; Glue security configuration covering job bookmarks and CloudWatch output |
| **164.312(b)** | Audit controls | Required | CloudTrail management events plus **object-level data events** on the PHI zones; S3 server access logging on every data bucket; VPC flow logs; Step Functions execution logging |
| **164.312(c)(1)** | Integrity | Required | S3 versioning on every bucket; CloudTrail `EnableLogFileValidation` producing SHA-256 digests; `DeletionPolicy: Retain` and `UpdateReplacePolicy: Retain` on all stateful resources |
| **164.312(c)(2)** | Mechanism to authenticate ePHI | Addressable | S3 checksums; CloudTrail digest files making log tampering detectable rather than merely unlikely |
| **164.312(d)** | Person or entity authentication | Required | `aws:MultiFactorAuthPresent` condition on human role assumption; `aws:SourceAccount` condition on service principal trust policies |
| **164.312(e)(1)** | Transmission security | Required | `aws:SecureTransport: false` deny on every bucket policy; VPC interface endpoints (TLS by default); no internet path from the ML VPC |
| **164.312(e)(2)(ii)** | Encryption in transit | Addressable | As above. TLS is enforced by policy rather than assumed |

### On the audit controls row

164.312(b) is the requirement most often satisfied on paper and not in fact. CloudTrail is on by default in most accounts, and it records management events — who created a bucket, who changed a policy. It does **not** record who read an object. Data events are opt-in, billed per event, and frequently disabled after the first bill.

Layer 2 enables data events scoped by ARN prefix to the PHI zones only. That scoping is what makes them affordable enough to leave on, and leaving them on is what makes the control real. See [ADR 0003](../adr/0003-zone-separated-buckets-over-prefixes.md) for why this requires separate buckets.

---

## Administrative safeguards — 164.308

Mostly outside infrastructure. Listed to make the boundary explicit rather than to claim coverage.

| Citation | Requirement | Status |
|---|---|---|
| 164.308(a)(1)(ii)(A) | Risk analysis | **Organisational.** This document is an input, not the analysis |
| 164.308(a)(1)(ii)(D) | Information system activity review | **Partial.** The logs exist and are retained; the review process — who reads them, how often, what triggers escalation — does not |
| 164.308(a)(3) | Workforce security | **Organisational.** The three personas are modelled here; joiner/mover/leaver processes are not |
| 164.308(a)(4) | Information access management | **Partial.** Lake Formation and IAM implement the technical half; access authorisation and periodic recertification are organisational |
| 164.308(a)(5) | Security awareness and training | **Organisational** |
| 164.308(a)(6) | Security incident procedures | **Not addressed.** No GuardDuty, Security Hub, or detective controls in these templates — a real gap for a production deployment |
| 164.308(a)(7) | Contingency plan | **Partial.** Versioning and `Retain` policies provide recovery primitives. No backup plan, no tested restore, no cross-region replication |

## Physical safeguards — 164.310

Inherited from AWS under the shared responsibility model and covered by the BAA and AWS's own attestations. Not addressable in a template.

---

## De-identification — 164.514

| Citation | Requirement | Implementation |
|---|---|---|
| 164.514(b)(2) | Safe Harbor method | Glue de-identification job; Comprehend Medical `DetectPHI` for narrative text; dates reduced to year; ages over 89 aggregated to `90+` |
| 164.514(c) | Re-identification code | **Random surrogate identifiers with a crosswalk held inside the PHI boundary.** Explicitly *not* a keyed hash of the MRN — see [ADR 0008](../adr/0008-safe-harbor-with-random-surrogate-identifiers.md) for why the common HMAC approach fails the "not derived from" condition |

The de-identification *methodology* — the field-by-field determination of what is removed, transformed, or retained — is a document the privacy officer owns. The architecture implements a methodology; it does not substitute for one.

---

## Documentation retention — 164.316(b)(2)(i)

Six years from creation or last effective date, whichever is later. Reflected in the defaults: `AccessLogRetentionDays` and `CloudTrailRetentionDays` both default to 2555 days (~7 years), slightly above the floor.

Note that this citation governs *documentation* of policies and procedures. It is not a stated retention requirement for audit logs, and the Security Rule does not fix one. Using it as the anchor is a common and defensible convention, not a literal reading — stated here so nobody inherits the assumption without seeing it.

---

## Known gaps

Consolidated for the reader who skips to the end:

1. **No detective controls.** No GuardDuty, no Security Hub, no Config rules. 164.308(a)(6) is unaddressed and this is the largest gap for production use.
2. **No backup plan.** Versioning is not backup. No AWS Backup configuration, no tested restore, no cross-region replication.
3. **No automatic logoff.** SageMaker Studio idle shutdown needs a lifecycle configuration.
4. **No break-glass role.** Technically possible via the key policies; not defined, not alarmed.
5. **No Config rule enforcing catalogue tagging.** A mis-tagged table fails *open* under LF-Tag access control — see [ADR 0006](../adr/0006-lake-formation-tag-based-access-control.md).
6. **De-identification output review is process, not infrastructure.** Comprehend Medical is a statistical model; sampling-based human review with a documented error rate is required and is not here.
