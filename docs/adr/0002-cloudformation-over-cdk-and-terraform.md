# 0002 — CloudFormation over CDK and Terraform

**Status:** Accepted

## Context

Three credible options for provisioning this platform, and the usual arguments do not settle it because the deciding constraint here is unusual: **the primary reviewer of these templates is not an engineer.**

In a health system, infrastructure that touches PHI is reviewed by a security officer and, at audit time, by someone from compliance or an external assessor. Those readers need to answer "is this bucket encrypted with a customer-managed key, and who can decrypt it" by reading the artefact. That requirement pulls against most of the usual reasons to prefer a higher-level tool.

### AWS CDK

Real language, real abstractions. The four near-identical bucket definitions in `01-foundation-data-lake.yaml` would collapse into a loop over a zone list, and the L2 constructs bring sensible security defaults for free.

Against: CDK synthesises to CloudFormation anyway, and the synthesised output is what actually gets deployed and what an auditor would have to read. That output is machine-generated, with logical IDs like `LandingZoneBucketPolicy8A1B2C3D`, and it is materially harder to review than a hand-written template. The repository would also gain a build step and a Node toolchain, which is a supply-chain surface in an environment where that matters.

### Terraform

Mature, excellent plan output, strong module ecosystem, and genuinely better than CloudFormation at expressing this kind of repetition.

Against: state. Terraform state contains a full description of every resource, and for a healthcare platform "where does the state file live, who can read it, and is it encrypted" becomes a control that needs its own review — a new problem created by the choice of tool. It also introduces a third-party dependency into the provisioning path for a regulated workload, which is defensible but is one more thing to justify.

### CloudFormation

Native, no state to manage, drift detection built in, StackSets for the multi-account structure this would grow into. The template *is* the reviewable artefact — no synthesis step between what is written and what is read.

## Decision

CloudFormation, hand-written YAML.

## Consequences

**The templates are verbose.** Four bucket definitions differ only in name, key, prefix, and lifecycle, and CloudFormation offers no way to express that as a loop. This is real duplication and it will drift if someone edits three of four blocks. The mitigation is CI: cfn-lint and Checkov run on every commit, so a bucket that loses its TLS policy fails the build rather than shipping.

**No conditionals worth the name.** `Fn::If` exists but nests badly. Anything genuinely conditional becomes a separate template.

**Cross-stack references are brittle.** `Fn::ImportValue` creates a hard dependency: an exported value cannot be changed while another stack imports it. This is the price of splitting into four layers, and it is why the layer boundaries follow lifecycle (things that change at different rates) rather than tidiness.

## What would change this decision

- **CDK** becomes the right answer if the resource count grows past what is reviewable by hand, or if the organisation already standardises on it. Generating reviewable CloudFormation via `cdk synth` and committing the output is a reasonable middle path that keeps the auditor story intact.
- **Terraform** becomes the right answer if the platform must span AWS and another cloud, or if the team's existing operational muscle is entirely Terraform — tool familiarity beats tool theory most of the time.
