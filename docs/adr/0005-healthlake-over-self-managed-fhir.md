# 0005 — HealthLake over a self-managed FHIR server

**Status:** Accepted

## Context

The conformed zone normalises everything to FHIR R4. That normalised data needs somewhere to live that speaks FHIR — a real REST API supporting `Patient/{id}`, `$everything`, resource search, and referential integrity — because downstream consumers (clinical applications, SMART-on-FHIR apps, the summarisation workload) expect FHIR semantics rather than Parquet files.

### Self-managed HAPI FHIR

HAPI is the reference JVM implementation, open-source and mature. Running it on EKS with Aurora PostgreSQL behind it gives complete control: custom FHIR profiles, custom search parameters, implementation guides specific to the organisation, and a cost model that is pure infrastructure with no per-request component.

Against: the platform team now operates a stateful clinical system. Patching, HA across AZs, backup and restore testing, FHIR conformance validation, load testing, and the security evidence for all of it. For a health system whose platform team is typically single-digit headcount, this consumes the capacity that was supposed to go into the actual differentiating work.

### HealthLake

Managed FHIR R4 with a REST API, integrated NLP that extracts entities from unstructured clinical text, HIPAA-eligible, encrypted with a customer-managed key. Import from S3 handles the bulk load path.

Against: consumption-based pricing that scales with stored data and request volume, and which can surprise a team that has only modelled storage. Less control over conformance profiles than HAPI. And **regional availability is narrower than S3 or Glue** — this is a real constraint that has ended the discussion for organisations with data-residency requirements in regions where HealthLake is not offered.

## Decision

HealthLake.

## Rationale

Operating a FHIR server is not where this organisation's advantage lies, and it is not what the project is being funded to produce. The differentiating work is the de-identification pipeline and the models it enables. Every engineer-month spent on FHIR server operations is a month not spent on either.

The integrated NLP is a genuine secondary benefit rather than a rounding error: extracting structured entities from unstructured notes is otherwise a substantial workstream on its own, and it feeds directly into both the readmission features and the de-identification pass.

## Consequences

**Cost needs monitoring from day one.** Consumption pricing on a dataset that grows monotonically is a line item that should have a budget alarm before it has a surprise. There is no such alarm in these templates — noted in the README's limitations.

**Region selection is constrained.** Any deployment region must be checked against HealthLake availability before the rest of the architecture is committed to it. This is the kind of constraint that is cheap to check first and expensive to discover late.

**FHIR profile customisation is limited.** An organisation with an established implementation guide and custom extensions may find the managed service does not accommodate them.

## What would change this decision

- A region requirement HealthLake does not serve. This is decisive and not negotiable around.
- Deep FHIR profile customisation that the managed service cannot express.
- Scale at which consumption pricing exceeds the fully loaded cost of running HAPI — including the operational headcount, which is the term most commonly left out of that comparison and the reason the crossover point is further out than it first appears.
