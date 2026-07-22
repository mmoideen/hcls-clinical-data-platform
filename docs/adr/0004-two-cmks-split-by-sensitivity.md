# 0004 — Two CMKs split by sensitivity, and Retain on all of them

**Status:** Accepted

## Context

Two questions, closely related enough to answer together: how many customer-managed keys, and what happens to them when a stack is deleted.

### How many keys

**One key for the account.** Simplest to operate. But every grant is then a grant to everything encrypted in the platform, and the access model degrades to "whoever can decrypt, can decrypt all of it". The distinction between a data scientist and a data engineer stops being expressible in key policy and has to live entirely in S3 policy — which means a single S3 policy error is a full PHI exposure.

**One key per bucket.** Maximum isolation. Six keys, which at roughly $1/month each is not the objection. The objection is grant management: the de-identification pipeline needs decrypt on conformed and encrypt on de-identified regardless, so the cross-boundary role ends up with multi-key grants either way, and the extra keys buy separation between buckets whose audiences are identical.

**Two keys, split by sensitivity.** The boundary that matters operationally is PHI versus de-identified, because that is the boundary the *audience* changes across. Landing and conformed are read by the same small group of data engineers. The de-identified zone is read by data scientists, analysts, and potentially external research collaborators — a materially larger and more fluid population.

### What happens on stack deletion

An `AWS::KMS::Key` without a `DeletionPolicy` is scheduled for deletion when its stack is deleted. After the pending window elapses, the key material is destroyed.

## Decision

Two customer-managed keys — `PhiEncryptionKey` and `DeIdentifiedEncryptionKey` — plus separate keys for CloudTrail and CloudWatch Logs, which are scoped by service rather than by data sensitivity.

Every key carries `DeletionPolicy: Retain` and `UpdateReplacePolicy: Retain`.

## Rationale

### On the split

Grant and revoke on the de-identified key never touches PHI. A research collaboration that ends, an analyst who changes teams, a Clean Rooms configuration that gets torn down — none of these require any operation against a key that can decrypt clinical records. That is the whole point.

The corollary is the control the architecture actually rests on: the data scientist role and the SageMaker execution role hold **no grant of any kind** on the PHI key. If a bucket policy were misconfigured tomorrow to permit them to read the conformed zone, they would retrieve ciphertext. Encryption is doing authorisation work here, not just protecting data at rest.

### On Retain

This is the one that would have caused real damage. The buckets carry `DeletionPolicy: Retain` specifically so the data survives a stack deletion. If the key does not carry the same policy, deleting the stack destroys the key while retaining the bucket — and every object in that retained bucket becomes permanently unreadable. Retained-but-undecryptable is arguably worse than deleted, because the storage bill continues.

`UpdateReplacePolicy` is separate and does not default from `DeletionPolicy`; without it, any change to an immutable property replaces the resource and deletes the original. The two policies must both be set.

## Consequences

**Teardown is manual.** Deleting all four stacks leaves six keys, seven buckets, and several log groups behind. There is no single command that cleans up, and the deployment documentation says so.

**Key deletion has a floor of seven days.** `PendingWindowInDays` is exposed as a parameter with a 7–30 range. Seven is the AWS minimum and exists so that a mistaken deletion is recoverable.

**The cross-boundary role holds grants on both keys.** `DeIdentificationPipelineRole` can decrypt PHI and encrypt de-identified data. This concentration is intentional — it is the single chokepoint every PHI-to-de-identified transition passes through — but it is also the highest-value role in the account and should be alarmed on accordingly. It is assumable only by the Glue and Step Functions service principals, never by a human identity.

## What would change this decision

If the de-identified zone moved to its own AWS account — which is where this architecture should go at production scale — the key split becomes an account boundary and this decision is superseded by something stronger. That is the right end state; two keys in one account is the version that fits a single-account reference implementation.
