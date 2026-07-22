# 0003 — Separate buckets per zone, not prefixes in one bucket

**Status:** Accepted

## Context

The lake has three data zones — landing, conformed, de-identified. These can be modelled as three buckets, or as one bucket with `landing/`, `conformed/`, and `deid/` prefixes.

The single-bucket form is genuinely attractive. Fewer resources, one name to remember, one bucket policy, and S3 prefixes support most of the controls you would want: IAM policies can be prefix-scoped, and lifecycle rules take a prefix filter.

## Decision

Three separate buckets, one per zone, plus separate buckets for access logs, CloudTrail, and pipeline artefacts.

## Rationale

The argument that settles it is narrow and technical: **default server-side encryption is a bucket-level property.** `BucketEncryption` applies to the whole bucket. There is no way to configure `landing/` to default to one KMS key and `deid/` to another.

Since ADR 0004 puts PHI and de-identified data under different CMKs — and that key separation is the control the whole access model rests on — the zones must be different buckets. The bucket split follows from the key split, not the other way around.

Once that is forced, the secondary benefits are worth having:

- **Blast radius per policy.** A mistake in a prefix condition (`deid/*` versus `deid*`, or a missing trailing slash) is a class of bug that silently widens access. A mistake in a bucket policy affects one zone.
- **Separate access log streams.** Each bucket logs to its own prefix, so "who read the conformed zone last Tuesday" is a scoped query rather than a filter across everything.
- **CloudTrail data event scoping by ARN prefix.** Data events are billed per event. Layer 2 logs object-level activity on the PHI zones and not on the de-identified zone, which is only expressible if they are separate buckets. An unscoped data-event selector on a busy account is the usual reason teams turn data events off again — and turning them off is how a 164.312(b) audit finding happens.
- **Per-zone lifecycle without filter juggling.** Landing goes to Glacier IR at 90 days because it is written once and read rarely. Conformed and de-identified use Intelligent-Tiering because access is bursty and unpredictable. Both are expressible with prefix filters, but the intent is clearer at bucket scope.

## Consequences

More resources to name and manage, and the S3 bucket namespace is global — hence the `${AWS::AccountId}-${AWS::Region}` suffix on every name, and the 12-character cap on `ProjectName` that keeps the longest composed name inside the 63-character limit.

Any job spanning zones needs grants on multiple buckets and multiple keys. This is visible in `DeIdentificationPipelineRole`, which is the only role holding that combination — and making that role conspicuous is a feature, not an accident.

## What would change this decision

If the zone model collapsed to a single sensitivity tier — no de-identified zone, one key, one audience — the argument evaporates and one bucket with prefixes would be the simpler correct answer.
