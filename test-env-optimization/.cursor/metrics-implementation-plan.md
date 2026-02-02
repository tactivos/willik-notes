---
name: Test Env Metrics Implementation Plan
overview: Technical implementation plan for instrumenting, measuring, and reporting on test environment provisioning performance metrics.
todos:
  - id: baseline-instrumentation
    content: Add timing metrics to mural-kraken provision.ts
    status: pending
  - id: operator-instrumentation
    content: Add timing metrics to test-envs-operator reconciliation
    status: pending
  - id: create-dashboard
    content: Create Datadog dashboard for provisioning metrics
    status: pending
  - id: collect-baseline
    content: Collect 2-4 weeks of baseline metrics before optimization
    status: pending
  - id: post-optimization-metrics
    content: Add optimization-specific metrics (CI tests complete flag)
    status: pending
  - id: analyze-impact
    content: Compare baseline vs. post-optimization metrics and calculate business impact
    status: pending
---

# Test Environment Provisioning Metrics - Implementation Plan

## Overview

This is the technical implementation plan for instrumenting comprehensive metrics to measure test environment provisioning performance. For the business case and project proposal, see [Test Environment Optimization Proposal](proposal.md).

**Scope**: This plan covers metric definitions, instrumentation approach, data collection, dashboards, and analysis methodology.

## Key Metrics to Track

### 1. Primary Metric: Provisioning Lead Time (P50, P90, P99)

**Definition**: Time from provisioning request to environment ready.

**Measurement Points**:

- **Start**: When `/mural-env up` command is received (or auto-provision triggered)
- **End**: When environment status becomes "Ready"

**Why It Matters**: This is the core developer experience metric - how long they wait for an environment.

**Current State**: Unknown - likely 15-20 minutes

**Target State**: 5-10 minutes (estimated 10-15 minute improvement)

**Business Value**: Reduced developer wait time = faster iteration cycles = more productive development

### 2. CI-to-Provisioning Delay

**Definition**: Time from Docker image build complete to provisioning start.

**Measurement Points**:

- **Start**: When production image build job completes (GitHub Actions timestamp)
- **Checkpoint**: When CI checks are satisfied (Kraken polling detects)
- **End**: When provisioning actually begins

**Why It Matters**: Measures the specific delay we're eliminating.

**Current State**: Unknown - likely 10-15 minutes (test duration)

**Target State**: ~30 seconds (polling interval)

**Business Value**: Shows the direct impact of not waiting for CI tests

### 3. Environment Provisioning Rate

**Definition**: Number of environments provisioned per day/week.

**Why It Matters**: Shows usage patterns and helps justify infrastructure investment.

**Business Value**: Higher provisioning rate = more active development/testing

### 4. Failed Provisioning Rate (Categorized)

**Definition**: Percentage of provisioning requests that fail, broken down by failure type.

**Failure Categories**:

- `timeout_waiting_builds` - Builds took longer than 30 minutes
- `k8s_error` - Kubernetes provisioning failed (infrastructure issue)
- `no_pr_found` - Invalid branch/PR (user error)
- `unknown` - Other errors requiring investigation

**Success Rate Calculation**: `success / (success + failed)`

**Why It Matters**: Different failure types require different actions. Timeouts indicate slow builds, K8s errors indicate infrastructure problems.

**Business Value**: Helps identify system reliability issues and distinguish between infrastructure problems vs. slow builds

### 5. Manual Re-provisioning Rate

**Definition**: How often users manually deprovision (`/mural-env down`) and then re-provision (`/mural-env up`) the same branch within a short time window (e.g., within 2 hours).

**Distinguishes from Automatic Updates**: When new commits are pushed to a branch with an existing environment, `test-envs-operator` automatically updates the environment with new images. This is normal behavior and NOT counted as "re-provisioning".

**Why It Matters**:

- High re-provisioning rate indicates reliability problems (environments getting stuck, networking issues, pods not starting)
- Reflects user frustration and wasted time debugging infrastructure

**Measurement**: Track pairs of `down` → `up` commands for the same branch within N hours (e.g., 2 hours).

**Business Value**: Lower rate = more reliable environments = less developer frustration and wasted time

### 6. Parallel Development Time

**Definition**: Time developers can work while CI tests run in background.

**Measurement**:

- Environment ready time - CI test completion time = parallel work window

**Why It Matters**: Quantifies productivity gain from parallel work.

**Business Value**: Developer can start testing/debugging while CI runs

## Current Measurement Capabilities

### Existing Infrastructure

**Observability Stack**:

- **Percibir** ([@tactivos/percibir](https://github.com/tactivos/percibir)) - Observability library used across services
- **Datadog** - Metrics backend (hot-shots StatsD client)
- Available metric types: `gauge`, `increment`, `distribution`, `timer`, `asyncTimer`

**Current Logging**:

- [mural-kraken/src/worker/events/testing-env/provision.ts](mural-kraken/src/worker/events/testing-env/provision.ts)
  - Uses `observe('testing-env:provision')` for logging
  - Logs check status polling
  - No timing metrics currently emitted

- [test-envs-operator/src/controllers/test-env-bootstrap](test-envs-operator/src/controllers/test-env-bootstrap)
  - Uses `observe('test-env-bootstrap-controller')`
  - Logs provisioning steps
  - No timing metrics currently emitted

**Key Events Tracked**:

- Provisioning request received
- Check polling status
- Provisioning success/failure
- Environment ready notification

### What's Missing

❌ No timing measurements for provisioning duration

❌ No correlation between GitHub Actions timestamps and provisioning

❌ No metrics for CI wait time

❌ No Datadog dashboards for provisioning performance

❌ No alerting on provisioning delays

## Implementation Plan

### Phase 1: Baseline Measurement (Before Optimization)

**Goal**: Establish current performance metrics to compare against post-optimization.

#### Step 1.1: Add Timing Metrics to mural-kraken

**Location**: [mural-kraken/src/worker/events/testing-env/provision.ts](mural-kraken/src/worker/events/testing-env/provision.ts)

**Approach**: Instrument the `handleProvision` function to track timing at key points:

1. Record start time when provisioning request begins
2. Track duration of CI checks polling phase
3. Track duration of Kubernetes provisioning
4. Track duration waiting for pods to be ready
5. Categorize failures by type (timeout, k8s error, no PR, unknown)

**Metrics to Emit**:

- `testenv.checks.duration` - Time spent polling for CI checks
- `testenv.provision.total_duration` - End-to-end provisioning time
- `testenv.provision.kubernetes_duration` - Time to provision K8s resources
- `testenv.provision.wait_ready_duration` - Time waiting for pods to be ready
- `testenv.provision.success` - Count of successful provisions (increment)
- `testenv.provision.failed` - Count of failed provisions (increment with `failure_type` tag)

#### Step 1.2: Add Timing Metrics to test-envs-operator

**Location**: [test-envs-operator/src/controllers/test-env-bootstrap/index.ts](test-envs-operator/src/controllers/test-env-bootstrap/index.ts)

**Approach**: Instrument the reconciliation loop to track how long it takes the operator to process each test environment.

**Metrics to Emit**:

- `testenv.operator.reconcile_duration` - Time to reconcile a TestEnv resource
- `testenv.operator.reconcile_success` - Count of successful reconciliations (increment)
- `testenv.operator.reconcile_failed` - Count of failed reconciliations (increment with `status: failed` tag)

#### Step 1.3: Create Datadog Dashboard

Create dashboard in Datadog UI to visualize:

- P50/P90/P99 for `testenv.provision.total_duration`
- P50/P90/P99 for `testenv.checks.duration`
- Success/failure rates
- Provisioning volume over time
- Breakdown by repository

#### Step 1.4: Collect Baseline Data

Run for 2-4 weeks to establish baseline:

- Typical provisioning duration
- Peak usage times
- Common failure modes
- Repository-specific patterns

### Phase 2: Post-Optimization Measurement

**Goal**: Measure the same metrics after implementing the optimization to quantify improvements.

#### Step 2.1: Deploy Optimization Changes

Deploy all changes from the main optimization plan.

#### Step 2.2: Continue Metric Collection

Keep the same metrics running - no changes needed to instrumentation.

#### Step 2.3: Add Optimization-Specific Metrics

**Goal**: Track whether environments were provisioned before or after CI tests complete.

**Location**: [mural-kraken/src/worker/events/testing-env/provision.ts](mural-kraken/src/worker/events/testing-env/provision.ts)

**Approach**: After required build checks pass, query GitHub for test job status to determine if tests are complete or still running.

**Metrics to Emit**:

- `testenv.provision.started` - Increment with tag `ci_tests_complete: true/false`

This allows segmenting metrics to see:

- How many environments are provisioned early (before tests)
- Whether early provisioning correlates with different failure rates
- Actual time saved by not waiting for tests

### Phase 3: Analysis & Reporting

#### Step 3.1: Compare Metrics

Compare baseline vs. post-optimization:

- Median provisioning time reduction
- P90/P99 improvements
- Change in failure rates
- Change in re-provisioning frequency

#### Step 3.2: Calculate Business Impact

**Time Saved Per Provision**:

- Baseline median: 18 minutes (example)
- Optimized median: 7 minutes
- **Savings: 11 minutes per provision**

**Developer Productivity Impact**:

- Provisions per week: 50 (example from metrics)
- **Time saved per week: 550 minutes = 9.2 hours**
- **Annual savings: ~480 developer hours**

**Cost Impact** (if desired):

- Average developer cost: $100/hour (example)
- Annual savings: $48,000

#### Step 3.3: Create Executive Summary

Template for business case:

```markdown
## Test Environment Provisioning Optimization Results

### The Problem
Developers waited an average of X minutes for test environments while CI tests completed.

### The Solution
Modified provisioning to start as soon as Docker images are built, without waiting for tests.

### Results
- ⏱️ **X% faster provisioning** (Y minutes → Z minutes median)
- 📊 **ABC developer hours saved annually**
- 💰 **$XYZ,000 in productivity gains**
- 🎯 **N% improvement in P90 latency**

### Developer Experience
- Developers can now test in parallel with CI
- Faster iteration cycles
- Reduced context switching
```

## Datadog Dashboard Structure

### Recommended Dashboard Sections

**Section 1: Provisioning Performance**

- Graph: Provisioning duration (P50, P90, P99) over time
- Graph: Distribution of provisioning times
- Single-value: Current median provisioning time
- Single-value: Week-over-week change

**Section 2: Breakdown by Phase**

- Graph: CI checks wait time
- Graph: Kubernetes provisioning time
- Graph: Pod readiness wait time
- Stacked area: Time breakdown over time

**Section 3: Volume & Reliability**

- Graph: Provisions per day
- Graph: Success vs. failure rate
- Graph: Re-provisioning frequency
- Table: Top 10 slowest provisions (with links to logs)

**Section 4: Optimization Impact** (Post-deployment)

- Graph: Provisions with/without CI complete
- Graph: Before/after comparison
- Single-value: Time savings per provision
- Single-value: Total time saved this week

**Section 5: Repository Breakdown**

- Table: Median provisioning time by repository
- Graph: Provision volume by repository
- Graph: Failure rate by repository

## Alternative/Additional Metrics

### Developer Survey Metrics

- "How satisfied are you with test environment speed?" (1-5 scale)
- Before/after comparison

### Indirect Metrics

- PR cycle time (might decrease with faster env access)
- Number of test environment related support requests
- Time between PR creation and first developer test

### Infrastructure Metrics

- CI registry vs. production registry pull rates
- Early image pulls (before CI complete)
- Failed image pulls (would indicate problems)

## Risks & Considerations

### Metric Collection Risks

**Risk 1: Performance Overhead**

- Concern: Adding timing code could slow down provisioning
- Mitigation: Metrics are fire-and-forget, minimal overhead
- Percibir is designed for production use

**Risk 2: Metric Cardinality**

- Concern: Too many unique tags could overwhelm Datadog
- Mitigation: Limit tags to essential dimensions (repo, env_type)
- Avoid high-cardinality tags (branch names, user IDs)

**Risk 3: Baseline Data Quality**

- Concern: Not enough baseline data to compare
- Mitigation: Collect for 2-4 weeks before optimization
- Ensure representative usage patterns

### Interpretation Risks

**Risk 1: Seasonality**

- Concern: Usage patterns change over time
- Mitigation: Compare same time periods (week-over-week)
- Account for holidays, release cycles

**Risk 2: Confounding Factors**

- Concern: Other changes might affect provisioning speed
- Mitigation: Track deployment dates and correlate with metric changes
- Monitor infrastructure changes

## Success Criteria

### Must-Have Metrics

- ✅ P50 provisioning time reduced by at least 30%
- ✅ No increase in failure rate (within 5% margin)
- ✅ Datadog dashboard available with real-time data

### Nice-to-Have Metrics

- ⭐ P90/P99 provisioning time improvements
- ⭐ Developer satisfaction survey results
- ⭐ Cost savings calculation with executive summary

## Data Retention Considerations

### Datadog Metrics Retention

**Standard Retention Periods**:

- **Metrics at full granularity**: 15 months
- **Custom metrics**: Same 15-month retention
- **High-resolution metrics** (1-second): Rolled up to 1-minute after 7 days, then standard retention

**Implications for Our Use Case**:

- ✅ **Sufficient for before/after comparison**: 15 months covers baseline collection + post-optimization analysis
- ✅ **Good for trend analysis**: Can track improvements over quarters
- ⚠️ **Limited for long-term historical analysis**: Cannot compare year-over-year beyond 15 months
- ⚠️ **Cost**: Datadog custom metrics can be expensive at scale

**Best Practices**:

- Collect baseline for 2-4 weeks minimum
- Keep dashboards and reports with screenshots for historical records
- Export key metrics to spreadsheets/docs for long-term reference
- Consider downsampling less critical metrics to reduce costs

### Databricks as Long-Term Metrics Store

**Databricks Capabilities**:

- **Unlimited retention**: Store metrics data indefinitely in Delta Lake
- **Cost-effective storage**: Much cheaper than Datadog for long-term retention
- **SQL analytics**: Query historical data with Spark SQL
- **Advanced analysis**: ML models, anomaly detection, trend forecasting
- **Data warehouse integration**: Join with other business metrics (PR data, incident data, etc.)

**Architecture Options**:

#### Option 1: Datadog → Databricks Pipeline (Hybrid Approach)

**How It Works**:

```
Percibir → Datadog → Export to S3 → Databricks Ingestion
         ↓
    Real-time dashboards
```

**Pros**:

- Keep real-time Datadog dashboards for operational monitoring
- Long-term storage in Databricks for historical analysis
- Best of both worlds: real-time + long-term

**Cons**:

- Maintains two systems
- Additional pipeline to manage
- Data export lag (typically daily)

**Implementation**:

1. Set up Datadog metric export to S3 (via Datadog's AWS integration)
2. Create Databricks job to ingest from S3 daily
3. Create Delta tables with schema:
   ```sql
   CREATE TABLE testenv_metrics (
     timestamp TIMESTAMP,
     metric_name STRING,
     metric_value DOUBLE,
     tags MAP<STRING, STRING>,
     ingestion_date DATE
   ) USING DELTA
   PARTITIONED BY (ingestion_date);
   ```

4. Build Databricks SQL dashboards for long-term trends

#### Option 2: Direct to Databricks (Replace Datadog for This Use Case)

**How It Works**:

```
Percibir → Custom HTTP Endpoint → Databricks Ingestion → Delta Lake
         → Databricks SQL Dashboards
```

**Pros**:

- Single source of truth
- Lower cost for long-term storage
- More flexible querying
- Can join with other data sources easily

**Cons**:

- No real-time dashboards (dashboards refresh on schedule)
- Need to build custom ingestion pipeline
- Loses Datadog's operational monitoring features
- Team needs Databricks expertise

**Implementation**:

1. Create metrics ingestion API endpoint
2. Modify Percibir config to send to custom endpoint instead of/in addition to Datadog
3. Stream to Databricks (via Kafka or direct API)
4. Build Databricks dashboards

#### Option 3: Structured Logging to Databricks (Simplest)

**How It Works**:

```
Logger → Datadog Logs → Export to S3 → Databricks
       (Already exists)
```

**Pros**:

- Minimal new infrastructure
- Leverages existing log pipeline
- Structured logs can be parsed for metrics
- Already have timing data in logs

**Cons**:

- Not true metrics (aggregations happen at query time)
- Slower query performance
- Manual aggregation required

**Implementation**:

1. Ensure all timing data is in structured log format (already done)
2. Export Datadog logs to S3 (if not already configured)
3. Create Databricks notebook to parse logs and extract metrics:
   ```python
   from pyspark.sql.functions import *
   
   logs_df = spark.read.json("s3://logs-bucket/mural-kraken/")
   
   # Extract provisioning metrics from logs
   metrics_df = logs_df.filter(
     col("message").contains("Provisioning completed")
   ).select(
     col("timestamp"),
     col("branch"),
     col("totalDuration").cast("double"),
     col("checksDuration").cast("double"),
     col("provisionDuration").cast("double")
   )
   
   # Calculate P50, P90, P99
   metrics_df.approxQuantile("totalDuration", [0.5, 0.9, 0.99], 0.01)
   ```


### Recommended Approach

**For This Project**: **Option 1 (Hybrid)** or **Option 3 (Structured Logging)**

**Rationale**:

- **Near-term**: Use Datadog for real-time monitoring during optimization rollout
  - Need immediate feedback on changes
  - Operational alerting if provisioning breaks
  - Team already familiar with Datadog

- **Long-term**: Export to Databricks for historical analysis
  - Compare year-over-year trends
  - Join with developer velocity metrics
  - Cost-effective storage for indefinite retention
  - Advanced analytics (correlate with release velocity, incident rates, etc.)

**Phased Implementation**:

1. **Phase 1** (Immediate): Add Datadog metrics as described in main plan
2. **Phase 2** (After baseline): Set up log export to Databricks (Option 3)
3. **Phase 3** (If needed): Add proper metrics export pipeline (Option 1)

### Cost Comparison

**Datadog**:

- Custom metrics: ~$0.05 per metric per month
- Our metrics: ~10 custom metrics × 12 months = $6/month
- ✅ Minimal cost for this use case

**Databricks**:

- Storage: ~$0.023 per GB per month (Delta Lake)
- Estimated data: ~10MB/month for metrics = ~$0.25/month storage
- Compute: Only when querying (pay-per-query)
- ✅ More cost-effective for long-term retention

**Conclusion**: For our relatively low-volume metrics, Datadog cost is acceptable. Databricks makes sense if:

- We want to keep data >15 months
- We add many more metrics (high cardinality)
- We want to correlate with other business metrics

## Open Questions

1. What is the current provisioning rate? (Need baseline data)
2. Are there existing Datadog dashboards we should integrate with?
3. What's the acceptable failure rate threshold?
4. Should we set up automated alerts for slow provisioning?
5. Do we need Linear/JIRA integration to track which tickets use test envs?
6. **Is there already a Datadog → Databricks export pipeline we can leverage?**
7. **What's the team's preference: Datadog-only vs. hybrid approach?**
8. **Do we have Databricks access for the development/platform team?**