# Platform Engineering Metrics Inventory

Author: Willis Kirkham, Staff Software Engineer, Platform Engineering  
Date updated: 2025-02-02

## Summary

This document catalogs the metrics data currently collected by the Platform Engineering team across developer tooling, CI/CD, testing, and deployment tracking.

---

## What We Collect

### Build & Development Tooling

Data collected from the local development environment tool (MDK):

- Build duration and outcome for local builds
- Which commands developers run
- Command success/failure and error messages
- Basic machine info (OS, memory, CPU count)
- MDK version and configuration

**Storage:** Databricks (with Segment/S3 dependencies)

**Comments:** We have one or two dashboards showing specific insights into the data, like number of users on each preset, or number of users on each version of mdk. We have not explored how to extract more value from this data, or strategized about what data we need to make effective decisions about mdk product development. 

---

### CI/CD Pipeline Data

Data collected when automated builds run in GitHub Actions:

- Workflow name and duration
- Job names and durations
- Pass/fail outcomes
- Queue wait time (time before job started)

**Storage:** Datadog

**Comments:** Only enabled on a subset of repositories (murally, mural-api, mural-render, mdk). We have some alerting for regressions in CI job duration. We do not have dashboards for this data.

---

### Deployment Events - Datadog (DORA Metrics)

Datadog tracks deployment events using its native DORA metrics capability:

- Commit SHA associated with deployment
- Repository URL
- Service name
- Deployment timestamp (inferred from pod rollout)

**Storage:** Datadog

**Comments:** This is automatic and requires no manual reporting. Coverage depends on whether services have been istrumented. Implementation is incomplete; the following services have it: murally, mural-api, worker services.

---

### Test Results

Test execution data is collected after CI runs:

- Test name, file, and line number
- Pass/fail/skip status
- Test duration
- Associated commit, branch, and PR

**Storage:** Custom database (MongoDB)

**Comments:** A daily automated process computes "flakiness scores" and generates reports ranking the most unreliable tests. These reports are published to a static site. Metrics are also published to Datadog, and we have alerts for flakiness regressions. Data is stored to a MongoDB Atlas database, however querying the database for more insights is not possible due to poor database performance. There is a lot of data and we've never optimized the performance.

---

### Testing Environment Counts

Basic counts when testing environments are provisioned:

- Number of environments created
- Number of environments restarted

**Storage:** Datadog

**Comments:** Only tracks creation/restart events. Does not track environment lifecycle, duration of use, or utilization.

---

### Service Level Objectives (SLOs)

Configured thresholds for background job queue processing times:

| Queue Type | Latency Target | Goal |
|------------|----------------|------|
| General worker | 10 seconds | 95% |
| Exports | 2 minutes | 95% |
| Thumbnails | 17 minutes | 95% |
| Image processing | 2 minutes | 95% |
| Webhooks | 10 seconds | 95% |

Also includes search system latency monitoring.

**Storage:** Datadog

---

### Deployment Events - LinearB **DEPRECATED**

When code is deployed to production, deployment events were reported:

- Repository
- Commit reference
- Timestamp
- Deployment stage

**Storage:** LinearB

**Comments:** These metrics are no longer being collected since LinearB was deprecated. Deployment tracking is now handled by Datadog DORA metrics (see above).

---

## Where the Data Lives

| Data Type | Storage Location | Consolidated View? |
|-----------|------------------|-------------------|
| MDK telemetry | Databricks | No |
| CI/CD metrics | Datadog | No |
| Deployments (DORA) | Datadog | Partial (Datadog DORA dashboard) |
| Test results | MongoDB | Partial (daily HTML reports) |
| Testing env counts | Datadog | No |
| SLOs | Datadog | Yes (SLO dashboard) |
| Deployments (legacy) | LinearB | Deprecated |

