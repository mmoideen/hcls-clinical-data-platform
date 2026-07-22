# 0008 — Safe Harbor with random surrogate identifiers, not keyed hashes

**Status:** Accepted

## Context

HIPAA provides exactly two routes to de-identified data, at 45 CFR 164.514(b).

**Expert Determination**, 164.514(b)(1). A person with appropriate statistical training determines that the risk of re-identification is very small, and documents the methods and results. Preserves far more analytic utility — date shifting with consistent per-patient offsets, retained geography, retained age — because the standard is a risk judgement rather than a fixed list.

**Safe Harbor**, 164.514(b)(2). Remove eighteen enumerated identifiers. Mechanical, checkable, and requires no expert. Among the eighteen: names, geography finer than state, all elements of dates except year, and all ages over 89.

Two properties of the resulting dataset are non-negotiable for the intended workloads:

1. **Longitudinal linkage.** A readmission model must join encounters belonging to the same patient. A de-identification pass that emits unlinked rows produces a dataset in which the target variable cannot be constructed at all.
2. **Interval precision.** "30-day unplanned readmission" is a statement about days. Year-only dates cannot express it.

## Decision

Safe Harbor, with:

- **Randomly generated surrogate patient identifiers**, mapped in a crosswalk table that stays inside the PHI boundary and is never disclosed with the de-identified dataset.
- **Derived intervals computed before de-identification**, carried into the de-identified zone as integer day counts rather than as dates.
- **Ages over 89 aggregated** into a single `90+` category, per the rule.
- **Comprehend Medical `DetectPHI`** over free-text clinical notes, where identifiers appear inline and unpredictably.

## Rationale

### Why a random surrogate and not a keyed hash

The obvious implementation of stable linkage is an HMAC of the medical record number under a secret key held in KMS. It is deterministic, so the same patient produces the same token across runs; it is stable across the dataset; and without the key it is computationally infeasible to reverse. This is common practice and it appears in a lot of reference material.

It does not satisfy 164.514(c), which governs re-identification codes and imposes two conjunctive conditions. The first:

> **Derivation.** The code or other means of record identification is not derived from or related to information about the individual and is not otherwise capable of being translated so as to identify the individual.

An HMAC of the MRN is *derived from* information about the individual. That the derivation is computationally hard to invert satisfies the second half of the clause — "not otherwise capable of being translated" — but not the first. The conditions are joined by "and", and HHS's de-identification guidance is explicit that codes derived from identifiers, including hashes, do not meet this standard.

A randomly generated surrogate has no derivational relationship with the source identifier at all. Linkage is preserved by the crosswalk, and the crosswalk is exactly the "means of record identification" the regulation contemplates — held by the covered entity, used for no other purpose, and not disclosed.

The practical cost is that the pipeline becomes stateful. An HMAC is a pure function; a surrogate assignment requires reading the crosswalk to determine whether this patient has been seen before, and writing to it when they have not. That state lives inside the PHI boundary and is one more thing to back up and secure. It is worth it, because the alternative is a dataset that is not de-identified in the sense the regulation means, discovered by an auditor rather than by us.

### Why derived intervals rather than dates

Safe Harbor removes date elements. It does not enumerate durations, and a "days between discharge and readmission" integer is not one of the eighteen identifiers.

Computing intervals **before** the de-identification pass — while dates are still present in the conformed zone — and carrying only the integer forward preserves the entire signal the readmission model needs without retaining any date element in the de-identified output.

**This determination is not the architect's to make.** Whether a specific derived field is acceptable under Safe Harbor for a specific dataset is a question for the privacy officer, and it should be documented in the de-identification methodology alongside the field list. What the architecture provides is the ability to implement either answer: the derivation happens inside the PHI boundary, so restricting it later is a pipeline change, not a re-architecture. Flagging it here rather than quietly shipping it is the point.

### Why not Expert Determination

It is better, and it is where this should end up. Expert Determination would permit consistent date shifting, which preserves seasonality and day-of-week effects that Safe Harbor destroys and that genuinely matter for admissions modelling.

It is deferred because it is a recurring engagement rather than a one-time cost — the determination is specific to a dataset and a release, and each material change to the pipeline output needs revisiting. Starting with Safe Harbor means the platform can deliver a usable de-identified zone in the first quarter rather than after a procurement cycle for a qualified statistician.

## Consequences

**Seasonality is lost.** Year-only dates mean no month, no season, no day of week. Admissions have strong seasonal structure — respiratory illness in winter, trauma patterns by weekday — and Safe Harbor removes the ability to model it. This is the single largest analytic cost of the decision and it is the strongest argument for moving to Expert Determination once the platform has proven its value.

**The 90+ bucket compresses a clinically distinct population.** Patients over 89 have materially different readmission risk profiles, and the rule collapses them into one category. Nothing to be done about that under Safe Harbor.

**Comprehend Medical is not perfect.** `DetectPHI` is a statistical model, not a guarantee. A production deployment needs sampling-based human review of de-identified notes, with a documented error rate, and a defined process for what happens when an identifier is found downstream. That review process is outside these templates.

**The crosswalk is now a high-value target.** It is the only artefact that re-links the de-identified zone to real patients. It belongs in the PHI zone under the PHI CMK, with its own CloudTrail data events, and it should never be readable by anything the data scientist role can reach.

## What would change this decision

- Model performance that is demonstrably limited by the loss of temporal resolution. That is the evidence that justifies funding Expert Determination — and it is evidence this platform can produce, which is a better position than arguing for it in the abstract.
- Any data-sharing arrangement with an external research partner, where the recipient's own risk assessment may require Expert Determination regardless of what we would accept internally.
