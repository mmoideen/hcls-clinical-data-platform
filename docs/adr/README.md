# Architecture decision records

Format and rationale in [0001](0001-record-architecture-decisions.md).

| # | Decision | Status |
|---|---|---|
| [0001](0001-record-architecture-decisions.md) | Record architecture decisions | Accepted |
| [0002](0002-cloudformation-over-cdk-and-terraform.md) | CloudFormation over CDK and Terraform | Accepted |
| [0003](0003-zone-separated-buckets-over-prefixes.md) | Separate buckets per zone, not prefixes in one bucket | Accepted |
| [0004](0004-two-cmks-split-by-sensitivity.md) | Two CMKs split by sensitivity, and Retain on all of them | Accepted |
| [0005](0005-healthlake-over-self-managed-fhir.md) | HealthLake over a self-managed FHIR server | Accepted |
| [0006](0006-lake-formation-tag-based-access-control.md) | Lake Formation LF-Tags over named-resource grants | Accepted |
| [0007](0007-network-isolation-for-the-ml-environment.md) | VpcOnly with no NAT gateway | Accepted |
| [0008](0008-safe-harbor-with-random-surrogate-identifiers.md) | Safe Harbor with random surrogate identifiers | Accepted |

## Adding one

Next number, same structure: context that forced the choice, the decision, the consequences including the bad ones, and what would change it.

Accepted records are immutable. A decision that no longer holds gets a new record that supersedes it; the old file stays and is marked superseded.
