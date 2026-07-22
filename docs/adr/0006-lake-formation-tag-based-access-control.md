# 0006 — Lake Formation LF-Tags over named-resource grants

**Status:** Accepted

## Context

Lake Formation offers two ways to express a grant.

**Named resource grants** name the database, table, and optionally columns: "grant SELECT on `conformed.encounters` to `DataEngineerRole`". Explicit, immediately legible, and easy to audit while the catalogue is small.

**LF-Tag based access control** attaches key/value tags to catalogue resources and grants against tag expressions: "grant SELECT where `DataClassification = DeIdentified` to `DataScientistRole`".

## Decision

LF-Tags, with two tag keys: `DataClassification` (`PHI`, `DeIdentified`) and `Zone` (`Landing`, `Conformed`, `DeIdentified`).

`CreateDatabaseDefaultPermissions` and `CreateTableDefaultPermissions` are both set to empty lists.

## Rationale

### On tags versus named resources

Named-resource grants scale as principals × tables. A clinical FHIR catalogue is not small — R4 has over 140 resource types, and a realistic conformed zone lands somewhere in the dozens of tables before anyone has done anything ambitious. Every new table then requires a grant decision on every role, and the answer is made under time pressure by whoever registered the table.

The failure mode is not that this is laborious. It is that the permission set drifts from intent silently. Nobody notices the fourteenth table that was granted too broadly, because there is no single place where the intent was ever written down — the intent is distributed across hundreds of individual grants.

With LF-Tags the intent is written once, as an expression, and a newly registered table inherits correct access the moment it is tagged. The grant does not change as the catalogue grows.

### On the empty default permissions

This is the part that is easy to miss and expensive to get wrong.

Lake Formation ships with `IAMAllowedPrincipals` granted by default on new databases and tables. Under that setting, any principal whose IAM policy permits the underlying Glue and S3 actions bypasses Lake Formation entirely — Lake Formation grants are evaluated, find nothing to deny, and the IAM policy carries the day.

The result is a deployment that appears to have fine-grained access control, produces no errors, and enforces nothing. Setting both default permission lists to empty is what makes Lake Formation authoritative rather than advisory. It is one line, and without it the rest of this ADR is decorative.

## Consequences

**Tagging must be automated and audited.** A table that is registered but not tagged is invisible to every tag-based grant — which fails safe, but presents to the user as "the table exists and I cannot read it", with no obvious cause. Worse, a *mis*-tagged table fails open: tag a conformed-zone table `DataClassification = DeIdentified` and every data scientist can read PHI, silently and immediately.

The mitigation is that tagging belongs in the crawler configuration and in a Config rule that alarms on any catalogue resource lacking a `DataClassification` tag. Neither is in these templates, and the gap is listed in the README.

**Debugging is less direct.** "Why can this role read this table" requires resolving the tag expression rather than reading a grant. Lake Formation's `GetEffectivePermissionsForPath` helps, but it is a step removed from the named-resource case.

**Lake Formation administrators are extremely powerful.** An admin can grant themselves access to anything catalogued, including PHI. This is why `DataLakeAdminPrincipalArn` has no default value in `02-governance.yaml` — the parameter is deliberately awkward, so that the value is a decision rather than an inheritance.

## What would change this decision

A catalogue that stays genuinely small — under twenty tables, changing rarely — is better served by named-resource grants, where the explicitness is an asset rather than a maintenance burden.
