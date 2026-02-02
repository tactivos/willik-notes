# CI Run Time Analysis - murally vs mural-api

**Analysis Date**: January 26, 2026  
**Data Source**: Last 30 days of successful PR builds on GitHub Actions

## Executive Summary

**mural-api is ~2x slower than murally** (20 min vs 11 min average). However, **mural-api production images are built sequentially** after test images, adding unnecessary delay. If built in parallel, mural-api would complete in ~8-9 minutes, making **murally the bottleneck at 12 minutes**.

### Test Environment Readiness Impact
- **Current**: Test envs ready in ~17 min (waiting for mural-api)
- **Optimized**: Test envs ready in ~12 min (waiting for murally)
- **Potential speedup**: ~5 minutes (29% faster)

---

## Detailed CI Statistics

### murally Build Times
| Metric | Duration |
|--------|----------|
| **Average** | 11 minutes |
| **Median** | 9 minutes |
| **Range** | 7-18 minutes |
| **Sample size** | 5 successful PR builds |

**Top Time Consumers:**
1. Production Build (Webpack): ~12 min (734 sec)
2. Build test image: ~10 min (594 sec)
3. Package tests: ~8 min (490 sec)
4. React Rig v2 tests: ~7 min (403 sec)
5. Unit tests: ~5 min (329 sec)

**Parallelization**: ✅ Production build already runs in parallel with tests

### mural-api Build Times
| Metric | Duration |
|--------|----------|
| **Average** | 20 minutes |
| **Median** | 21 minutes |
| **Range** | 18-22 minutes |
| **Sample size** | 7 successful PR builds |

**Top Time Consumers:**
1. API integration tests: 10-13 min (609 sec)
2. Build API test image: 7-8 min (524 sec)
3. API Production Image: 7-8 min (488 sec)
4. Worker Production Image: ~6 min (377 sec)
5. Worker Tests: 4-6 min (369 sec)

**Parallelization**: ⚠️ Production images wait for test images to complete

---

## Timing Analysis from Actual Runs

### murally (Run 21377236691)
```
Build Start: 22:58:07

Parallel Execution:
├── Test image build:    22:58:07 → 23:08:01 (9 min)
└── Production build:    22:58:07 → 23:10:21 (12 min) ← BOTTLENECK

Production Ready: 23:10:21 (12 min 14 sec from start)
```

### mural-api Current Sequential (Run 21375030400)
```
Build Start: 21:42:04

Phase 1 - Test Images (parallel):
├── API test image:      21:42:04 → 21:50:48 (8 min) ← BLOCKS PRODUCTION
├── Worker test image:   21:42:04 → 21:46:14 (4 min)
├── Migrations test:     21:42:04 → 21:43:58 (1 min)
└── Modules test:        21:42:04 → 21:43:56 (1 min)

Phase 2 - Production Images (sequential, waits for test):
├── Migrations prod:     21:44:00 → 21:46:49 (2 min)
├── Worker prod:         21:46:16 → 21:52:33 (6 min)
└── API prod:            21:50:50 → 21:58:58 (8 min) ← BOTTLENECK

Production Ready: 21:58:58 (16 min 54 sec from start)
```

### mural-api Optimized Parallel (Proposed)
```
Build Start: 21:42:04

All Images (parallel):
├── API prod + test:     21:42:04 → ~21:50:04 (8 min) ← BOTTLENECK
├── Worker prod + test:  21:42:04 → ~21:48:04 (6 min)
├── Migrations prod+test: 21:42:04 → ~21:44:04 (2 min)
└── Modules test:        21:42:04 → 21:43:56 (1 min)

Production Ready: ~21:50:04 (8-9 min from start)
```

---

## Key Insights

### 1. Architecture Differences
| Aspect | murally | mural-api |
|--------|---------|-----------|
| **Images built** | 2 (test, prod) | 7 (4 test, 4 prod including 8ball) |
| **Test dependencies** | MongoDB, Redis, Elasticsearch | MongoDB, Redis, Elasticsearch, Postgres, OpenFGA |
| **Parallelization** | ✅ Optimal | ⚠️ Sequential production builds |
| **Main bottleneck** | Webpack build (12 min) | API integration tests (10-13 min) |

### 2. Why mural-api is Currently Slower
1. **Sequential production builds** - waits for test images unnecessarily
2. **More Docker images** - builds 4 production images vs murally's 1
3. **Complex integration tests** - spins up 5 database/service dependencies
4. **Parallel job matrix** - larger and more complex test matrix

### 3. The TODO Comment
In `mural-api/.github/workflows/build.yml:377`, there's a comment:
```yaml
# TODO use the test image as the first cache image
```
This suggests awareness that production could leverage test image caching, but there's no dependency requirement blocking parallel execution.

---

## Optimization Opportunities

### Immediate Win: Parallel Production Builds (mural-api)

**Self-Hosted Runner Note:** You use self-hosted runners. Parallel builds will:
- Use 7 concurrent runners (vs 4 staggered) for 8 minutes
- Free runners 9 minutes sooner for other work
- Same infrastructure cost (runners already provisioned)
- See `parallel-build-cost-analysis.md` for detailed runner capacity analysis

**Change Required**: Remove `needs: [build-test-image-*]` dependencies from production build jobs

**Current Code** (`mural-api/.github/workflows/build.yml:351-353`):
```yaml
build-production-api:
  name: API Production Image
  needs: [build-test-image-api]  # ← REMOVE THIS
  runs-on: [self-hosted, linux, medium]
```

**Impact**:
- mural-api: 17 min → 8-9 min (47% faster)
- Test env readiness: 17 min → 12 min (29% faster)
- **No changes to test reliability** - tests still run against test images

**Risk**: Low - production builds use the same Dockerfile, just different target stage

### Next Optimization: murally Webpack Build
Once mural-api is optimized, murally becomes the bottleneck at 12 minutes.

**Potential approaches**:
1. Webpack build caching improvements
2. Splitting production build into smaller chunks
3. Leveraging test image layers for production build
4. Running webpack in parallel stages

---

## Test Environment Dependencies

For a test environment to be "ready", it needs:
1. ✅ murally production image
2. ✅ mural-api production image
3. ✅ mural-worker production image
4. ✅ mural-migrations production image

**Current**: All images ready at ~17 min (mural-api completes last)  
**Optimized**: All images ready at ~12 min (murally completes last)

---

## Recommendations

### Priority 1: Implement Parallel Production Builds (mural-api)
- **Effort**: Low (workflow configuration change)
- **Impact**: High (5 min faster test environments)
- **Risk**: Low (no functional changes)
- **Timeline**: Can be implemented immediately

### Priority 2: Optimize murally Webpack Build
- **Effort**: Medium-High (requires build system changes)
- **Impact**: Medium (could save 2-4 min)
- **Risk**: Medium (changes to build process)
- **Timeline**: Requires analysis and testing

### Priority 3: Docker Build Caching
Both repositories could benefit from improved Docker layer caching:
- Use test images as cache sources for production builds
- Implement build cache persistence across runs
- Optimize Dockerfile layer ordering

### Priority 4: Test Parallelization
- Split large test suites into smaller parallel jobs
- Use test sharding for integration tests
- Consider test result caching for unchanged code

---

## Data Collection Details

**Sample PR Runs Analyzed:**

murally:
- Run ID: 21377236691 (primary analysis)
- Event type: pull_request
- Conclusion: success

mural-api:
- Run IDs: 21375030400, 21373200255, 21373100172
- Event type: pull_request
- Conclusion: success

**Query Methods:**
```bash
# Get CI statistics
gh run list --limit 200 --workflow="CI (if needed)" \
  --json databaseId,conclusion,startedAt,updatedAt,event

# Get job-level timing
gh run view <run-id> --json jobs | \
  jq '.jobs | map({name, duration: (completed - started)})'
```

---

## Appendix: Current Workflow Structure

### murally Workflow
```yaml
jobs:
  build-test-image:      # 9-10 min
  quality:               # depends on test-image
    - lint               # parallel
    - unit tests         # parallel
    - package tests      # parallel
    - react rig tests    # parallel
  production-build:      # 12 min (NO DEPENDENCY - runs parallel)
  publish:               # depends on production-build
```

### mural-api Workflow (Current)
```yaml
jobs:
  # Phase 1: Build test images (parallel)
  build-test-image-api:        # 8 min
  build-test-image-worker:     # 4 min
  build-test-image-migrations: # 1 min
  build-test-image-modules:    # 1 min
  
  # Phase 2: Run tests (parallel, depend on test images)
  lint-and-unit-api:           # depends on api-test-image
  test-api-integration:        # depends on api-test-image (10-13 min)
  lint-worker:                 # depends on worker-test-image
  test-worker:                 # depends on worker-test-image
  lint-migrations:             # depends on migrations-test-image
  test-migrations:             # depends on migrations-test-image
  
  # Phase 3: Build production (sequential!, depend on test images)
  build-production-api:        # depends on api-test-image (8 min) ⚠️
  build-production-worker:     # depends on worker-test-image (6 min) ⚠️
  build-production-migrations: # depends on migrations-test-image (2 min) ⚠️
  build-production-eight-ball: # no dependency (0 min)
  
  # Phase 4: Publish (parallel, depend on prod images)
  publish-api:        # depends on all prod images
  publish-worker:     # depends on all prod images
  publish-migrations: # depends on all prod images
  publish-eight-ball: # depends on all prod images
```

### mural-api Workflow (Optimized - Proposed)
```yaml
jobs:
  # Phase 1: Build ALL images (parallel)
  build-test-image-api:        # 8 min
  build-test-image-worker:     # 4 min
  build-test-image-migrations: # 1 min
  build-test-image-modules:    # 1 min
  build-production-api:        # 8 min ✅ NO DEPENDENCY
  build-production-worker:     # 6 min ✅ NO DEPENDENCY
  build-production-migrations: # 2 min ✅ NO DEPENDENCY
  build-production-eight-ball: # 0 min (already no dependency)
  
  # Phase 2: Run tests (parallel, depend on test images)
  lint-and-unit-api:        # depends on api-test-image
  test-api-integration:     # depends on api-test-image
  lint-worker:              # depends on worker-test-image
  test-worker:              # depends on worker-test-image
  lint-migrations:          # depends on migrations-test-image
  test-migrations:          # depends on migrations-test-image
  
  # Phase 3: Publish (parallel, depend on prod images AND tests passing)
  publish-api:        # depends on prod-api AND all tests
  publish-worker:     # depends on prod-worker AND all tests
  publish-migrations: # depends on prod-migrations AND all tests
  publish-eight-ball: # depends on prod-8ball AND all tests
```

**Key Change**: Remove test image dependencies from production builds, but ensure publish jobs still depend on tests passing for safety.
