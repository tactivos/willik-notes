# Platform Engineering Metrics: Gap Analysis and Architecture Proposal

## Summary

This document analyzes gaps and architectural limitations in the Platform Engineering team's current metrics collection, as documented in [Platform Engineering Metrics Inventory](platform-engineering-metrics-inventory.md).

It identifies coverage gaps, discusses the impact of fragmented data collection on analytical capabilities, and proposes a unified data pipeline architecture as a potential improvement.

---

## Known Gaps

- **No job-level build time tracking** in CI (only workflow totals)
- **Limited testing environment metrics** (counts only, no usage patterns)
- **CI metrics not enabled everywhere** (only on select repositories)
- **Incomplete DORA coverage** (not all services have entity.datadog.yaml configured)

---

## Data Architecture Limitations

### Fragmented Collection

Platform engineering metrics are distributed across multiple isolated systems:

| System | Data Type | Query Capability |
|--------|-----------|------------------|
| Datadog | CI/CD, DORA, SLOs, test env counts | Datadog-native queries only |
| Databricks | MDK telemetry | SQL via Databricks |
| MongoDB | Test results | Limited (unoptimized) |
| LinearB | Deployments (deprecated) | N/A |

Each system uses different data models, query languages, and retention policies. There is no unified schema or common identifier linking records across systems (e.g., correlating a CI build with its test results and subsequent deployment).

### Impact on Actionability

This fragmentation creates several challenges:

- **Manual data reconciliation**: Any analysis spanning multiple systems requires ad-hoc data exports and manual joining
- **No cross-system analysis**: Cannot answer questions like "What is the correlation between build time and test flakiness?" or "Which deployments followed builds with high failure rates?"
- **Inconsistent granularity**: Some systems track per-commit data, others per-workflow, others per-service—making alignment difficult
- **Limited historical analysis**: Each system has different retention periods and no centralized archive

### Potential Improvement: Unified Data Pipeline

A more robust approach would involve implementing a data engineering architecture:

1. **Extract-Load-Transform (ELT) pipelines**: Ingest raw data from each source system into a central data lake or warehouse
2. **Unified data model**: Define a common schema with shared dimensions (repository, commit, timestamp, service) enabling cross-system joins
3. **Analytics layer**: Build dashboards and reports from the unified dataset rather than querying siloed systems directly

This would enable:
- Holistic developer productivity metrics combining CI, test, and deployment data
- Trend analysis across the full software delivery lifecycle
- Self-service querying without specialized knowledge of each source system

**Trade-off:** This requires ongoing investment in data engineering infrastructure and maintenance. The current approach has lower overhead but limits analytical capabilities.
