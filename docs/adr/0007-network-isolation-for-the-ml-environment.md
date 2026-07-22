# 0007 — VpcOnly with no NAT gateway, accepting the package-management cost

**Status:** Accepted

## Context

SageMaker Studio runs in one of two network modes.

**`PublicInternetOnly`** is the default and it works immediately. Notebooks reach PyPI, conda, GitHub, and Hugging Face without configuration. The catch is where the traffic goes: it leaves through a SageMaker-managed network interface in an AWS-owned VPC. Your VPC flow logs do not see it, your endpoint policies do not apply to it, and your security group does not constrain it. For an environment processing clinical data, the honest description is that the network layer is outside your control and outside your audit.

**`VpcOnly`** places every Studio ENI in your subnets. Traffic is subject to your security groups, appears in your flow logs, and reaches AWS services through endpoints you configure and can attach policies to.

Within `VpcOnly` there is a second choice: provide a NAT gateway for internet egress, or provide none and reach AWS services exclusively through VPC endpoints.

## Decision

`VpcOnly`, with no NAT gateway and no internet gateway. Ten VPC endpoints — S3 as a gateway endpoint, the rest as interface endpoints.

The S3 gateway endpoint carries a policy resolving only the de-identified zone bucket and the SageMaker service buckets.

## Rationale

Egress filtering is a control that has to be correct continuously. No egress path is a control that is correct structurally. In an environment where the consequence of exfiltration is a reportable breach, the second is worth real inconvenience.

The endpoint policy is the part that makes this more than theatre. Even granting that IAM policies could be misconfigured, and that the SageMaker execution role carries the broad `AmazonSageMakerFullAccess` managed policy, the PHI zone buckets are **not routable from this VPC**. There is no network path, independent of what any identity policy says. Combined with the explicit `Deny` on the PHI key and PHI buckets in the execution role, and the fact that the role holds no grant on the PHI CMK, that is three independent controls that each individually prevent the same exposure.

Removing the NAT gateway also removes its hourly and per-GB charges, which on a training account moving real volumes of data is not a trivial line.

## Consequences

**`pip install` does not work, and this will be the first thing a data scientist notices.**

This is a genuine cost and should be stated without softening. On day one, someone opens a notebook, types `pip install lifelines`, and it hangs. If the answer to that is "file a ticket", the platform has traded a security property for an adoption problem, and adoption problems are how shadow IT starts.

The mitigation is CodeArtifact configured with a PyPI upstream, reachable through its own interface endpoint, with the Studio lifecycle configuration pointing pip at it. That gives self-service installs, a reviewable dependency inventory, and the ability to block a package after a supply-chain advisory — which is a better answer than open PyPI on its own merits. **It is not in these templates**, and until it is, this decision is incomplete rather than merely inconvenient.

**Ten endpoints is a lot of them, and the list is unforgiving.** Interface endpoints bill hourly per AZ, so this is a real monthly cost. More painfully, omitting any one of them breaks Studio in a way that presents as an unexplained timeout rather than a permission error. The list in `04-ml-platform.yaml` is deliberately complete rather than minimal for exactly that reason: `sts`, `logs`, `kms`, `sagemaker.api`, `sagemaker.runtime`, `aws.sagemaker.<region>.studio`, `ecr.api`, `ecr.dkr`, `bedrock-runtime`, and the S3 gateway.

**`EnableDnsHostnames` is required.** Interface endpoint private DNS does not resolve without it, and the resulting failure looks like a networking problem rather than a configuration one.

## What would change this decision

If the workload genuinely required reaching external services — a third-party model API, an external terminology server — then a NAT gateway with an egress firewall becomes necessary and the structural guarantee downgrades to a filtering one. That is a real trade and it should be made explicitly, not arrived at by someone adding a route to unblock themselves.
