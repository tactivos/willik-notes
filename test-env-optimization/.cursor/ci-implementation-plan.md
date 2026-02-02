---
name: Optimize Test Env Provisioning
overview: Modify testing environment provisioning to wait only for Docker image builds instead of complete CI workflows, enabling faster environment availability for developers across all platform services.
todos:
  - id: audit-repos
    content: Audit all repositories to determine workflow patterns and required changes
    status: completed
  - id: add-commit-sha-tags-pattern1
    content: Add commit SHA tagging to Pattern 1 repos (mural-api, murally, mural-integrations)
    status: pending
  - id: verify-sha-tags-pattern2
    content: Verify Pattern 2 repos already have commit SHA tags (realtime, antivirus, notifier, upload, pdf-import, banksy, smooth-operator, test-envs-operator)
    status: pending
  - id: update-all-testing-env-yml
    content: Update testing-env.yml files in all applicable repos to point to build jobs
    status: pending
  - id: update-deployment-templates
    content: Update deployment templates to use CI registry (muralci.azurecr.io or mural.azurecr.io based on pattern)
    status: pending
  - id: configure-registry-access
    content: Verify and configure registry credentials in test-envs-operator and namespaces
    status: pending
  - id: test-provisioning
    content: Test end-to-end provisioning with PRs from different repo patterns
    status: pending
---

# Optimize Test Environment Provisioning Speed

## Repository Audit Results

### Repositories with Test Environment Deployments

Based on audit of [mural-test-envs-templates/core/components](mural-test-envs-templates/core/components), these 10 repositories deploy services to test environments:

1. **murally** - Frontend web application
2. **mural-api** - Backend API (also deploys worker, 8ball, migrations)
3. **mural-realtime** - Real-time websocket service
4. **mural-integrations** - Integrations service
5. **mural-antivirus** - Antivirus scanning service
6. **mural-notifier** - Notification service
7. **mural-upload** - File upload service
8. **pdf-import** - PDF import service
9. **test-envs-operator** - Test environment operator itself
10. **banksy** - New service being added

### Repositories NOT in Test Environments

These are tracked in [effort-center](effort-center/src/effort-versions.ts) but don't deploy to test envs:

- **mural-image** - Not found locally, no deployment template
- **mural-render** - Has workflows but no deployment template
- **smooth-operator** - Has workflows but no deployment template
- **mural-bert** - Has workflows but no deployment template

## Workflow Pattern Classification

### Pattern 1: Reusable Workflow with Separate Build/Publish (NEEDS COMMIT SHA TAGGING)

**Characteristics:**

- Uses `workflow_call` pattern
- Separate production build jobs that push to CI registry
- Separate publish jobs that retag from CI to production registry
- Build jobs run BEFORE tests, publish jobs run AFTER tests

**Repositories:**

1. **mural-api** - [.github/workflows/build.yml](mural/mural-api/.github/workflows/build.yml)

   - Jobs: `build-production-api`, `build-production-worker`, `build-production-migrations`, `build-production-eight-ball`
   - Current testing-env.yml: `build / Publish API Image / Publish Images`, `build / Publish Worker Image / Publish Images`
   - Registry: `muralci.azurecr.io` → `mural.azurecr.io`
   - Production tag: `${{ github.run_id }}-${{github.run_number}}`

2. **murally** - [.github/workflows/build.yml](mural/murally/.github/workflows/build.yml)

   - Jobs: `production-build`
   - Current testing-env.yml: `build / Production Publish / Publish Images`
   - Registry: `muralci.azurecr.io` → `mural.azurecr.io`
   - Production tag: `${{ github.run_id }}-${{github.run_number}}-${{ github.run_attempt }}`

3. **mural-integrations** - [.github/workflows/build.yml](mural/mural-integrations/.github/workflows/build.yml)

   - Jobs: `build-production-image`
   - Current testing-env.yml: `Build production and publish`
   - Registry: `muralci.azurecr.io` → `mural.azurecr.io`
   - Production tag: `${{ github.run_id }}-${{github.run_number}}`

**Required Changes:**

- ✅ Add commit SHA tagging to production build jobs
- ✅ Update testing-env.yml to point to build jobs instead of publish jobs
- ✅ Update deployment templates to use `muralci.azurecr.io`

### Pattern 2: Direct Build with docker/metadata-action (ALREADY HAS COMMIT SHA)

**Characteristics:**

- Uses `docker/metadata-action` with `type=sha,prefix=,format=long`
- Builds and pushes directly to production registry with commit SHA tag
- No separate publish step

**Repositories:**

1. **mural-realtime** - [.github/workflows/build.yml](mural/mural-realtime/.github/workflows/build.yml)

   - Job: `build` → "Build and test Realtime"
   - Current testing-env.yml: `Build and test Realtime`
   - Registry: `mural.azurecr.io` (production registry)
   - Tags via metadata-action: `type=sha,prefix=,format=long`
   - ✅ Already has commit SHA tag

2. **mural-antivirus** - [.github/workflows/build.yml](mural/mural-antivirus/.github/workflows/build.yml)

   - Job: `build` → "Build mural-antivirus"
   - Current testing-env.yml: `Build mural-antivirus`
   - Registry: `mural.azurecr.io`
   - Tags via metadata-action: `type=sha,prefix=,format=long`
   - ✅ Already has commit SHA tag

3. **mural-notifier** - [.github/workflows/build.yml](mural/mural-notifier/.github/workflows/build.yml)

   - Job: `build` → "Build mural-notifier"
   - Registry: `mural.azurecr.io`
   - Tags via metadata-action: `type=sha,prefix=,format=long`
   - ✅ Already has commit SHA tag
   - ❓ No testing-env.yml file found

4. **mural-upload** - [.github/workflows/build.yml](mural/mural-upload/.github/workflows/build.yml)

   - Job: `build` → "Build mural-upload"
   - Current testing-env.yml: `Build mural-upload`
   - Registry: `mural.azurecr.io`
   - Tags via metadata-action: `type=sha,prefix=,format=long`
   - ✅ Already has commit SHA tag

5. **pdf-import** - [.github/workflows/build.yml](pdf-import/.github/workflows/build.yml)

   - Job: `build` → "Build"
   - Registry: `mural.azurecr.io`
   - Tags via metadata-action: `type=sha,prefix=,format=long`
   - ✅ Already has commit SHA tag
   - ❓ No testing-env.yml file found

6. **banksy** - [.github/workflows/build.yml](banksy/.github/workflows/build.yml)

   - Job: `build` → "Build"
   - Registry: `mural.azurecr.io`
   - Tags via metadata-action: `type=sha,prefix=,format=long` AND `type=raw,value=${{ github.event.pull_request.head.sha }}`
   - ✅ Already has commit SHA tag
   - ❓ No testing-env.yml file found

7. **smooth-operator** - [.github/workflows/build.yml](smooth-operator/.github/workflows/build.yml)

   - Job: `build` → "Run tests and build"
   - Registry: `mural.azurecr.io`
   - Tags via metadata-action: `type=sha,prefix=,format=long`
   - ✅ Already has commit SHA tag
   - ❌ No deployment template (not used in test envs)

8. **test-envs-operator** - [.github/workflows/build.yml](test-envs-operator/.github/workflows/build.yml)

   - Job: `build` → "Run tests and build"
   - Registry: `mural.azurecr.io`
   - Tags via metadata-action: `type=sha,prefix=,format=long`
   - ✅ Already has commit SHA tag
   - ❓ No testing-env.yml file found

**Required Changes:**

- ❌ No commit SHA tagging needed (already present)
- ✅ Create testing-env.yml files for repos that need them
- ✅ Update testing-env.yml to point to build job (most already do)
- ❌ No deployment template changes needed (already using production registry)

### Pattern 3: Complex Multi-Stage (mural-render)

**mural-render** - [.github/workflows/ci.yml](mural-render/.github/workflows/ci.yml)

- Complex multi-stage build (base-image → builder-image → build → output-containers)
- Outputs: engine, cli artifacts
- Registry: `mural.azurecr.io`
- Tags via metadata-action: `type=raw,value=${{ github.sha }}`
- ✅ Already has commit SHA tag
- ❌ No deployment template (not used in test envs)

## Current Behavior Confirmed

The system **currently waits for Docker image publishing to complete**, which happens AFTER all CI tests pass.

### Current CI Pipeline Flow (Pattern 1 - mural-api/murally)

```
1. Build Test Images → muralci.azurecr.io (CI registry)
2. Run All Tests (linting, unit, integration) ← BLOCKS HERE
3. Build Production Images → muralci.azurecr.io (5 mins)
4. Publish Images → mural.azurecr.io ← requiredChecks points here (20 mins)
5. Environment provisions after step 4
```

### Current CI Pipeline Flow (Pattern 2 - realtime/antivirus/etc)

```
1. Build Test Image
2. Run All Tests ← BLOCKS HERE
3. Build Production Image → mural.azurecr.io with SHA tag (15 mins)
4. Environment provisions after step 3
```

### Key Files Involved

- [mural-kraken/src/worker/events/testing-env/checks.ts](mural-kraken/src/worker/events/testing-env/checks.ts) - validates checks via `checkPullRequestBuildImageChecks()`
- [mural-kraken/src/worker/events/testing-env/provision.ts](mural-kraken/src/worker/events/testing-env/provision.ts) - polls for checks in `handleProvision()`
- [test-envs-operator](test-envs-operator) - creates Kubernetes resources
- [effort-center/src/effort-versions.ts](effort-center/src/effort-versions.ts) - resolves commit SHAs

## Proposed Solution by Pattern

### Pattern 1 Repos: Add Commit SHA Tagging

For **mural-api**, **murally**, and **mural-integrations**, add commit SHA tagging to production build jobs so CI registry images can be referenced by SHA.

#### Example: mural-api

[mural-api/.github/workflows/build.yml](mural/mural-api/.github/workflows/build.yml):

```yaml
build-production-api:
  name: API Production Image
  needs: [build-test-image-api]
  runs-on: [self-hosted, linux, medium]
  steps:
    # ... existing checkout and login steps ...
    
 - name: Build
      id: build
      uses: tactivos/private-actions/build-test-image@main
      with:
        registry: ${{ env.CI_REGISTRY }}  # muralci.azurecr.io
        name: ${{ env.API_IMAGE }}
        tag: ${{ env.PRODUCTION_TAG }}  # runId-runNumber
        file: docker/api.Dockerfile
        target: production
    
    # NEW STEP: Tag with commit SHA
 - name: Tag with commit SHA
      run: |
        docker tag ${{ steps.build.outputs.image-uri }} \
                   ${{ env.CI_REGISTRY }}/${{ env.API_IMAGE }}:${{ github.sha }}
        docker push ${{ env.CI_REGISTRY }}/${{ env.API_IMAGE }}:${{ github.sha }}
```

Apply same pattern to:

- `build-production-worker`
- `build-production-migrations`
- `build-production-eight-ball`
- murally `production-build`
- mural-integrations `build-production-image`

#### Update testing-env.yml

[mural-api/testing-env.yml](mural/mural-api/testing-env.yml):

```yaml
requiredChecks:
- build / API Production Image
- build / Worker Production Image
```

[murally/testing-env.yml](mural/murally/testing-env.yml):

```yaml
requiredChecks:
- build / Production Build
```

[mural-integrations/testing-env.yml](mural/mural-integrations/testing-env.yml):

```yaml
requiredChecks:
- build / Build production image
```

#### Update Deployment Templates

Change registry from `mural.azurecr.io` to `muralci.azurecr.io`:

- [mural-test-envs-templates/core/components/api/deployment.yaml](mural-test-envs-templates/core/components/api/deployment.yaml)
- [mural-test-envs-templates/core/components/worker*/deployment.yaml](mural-test-envs-templates/core/components/) (all worker variants)
- [mural-test-envs-templates/core/components/integrations/deployment.yaml](mural-test-envs-templates/core/components/integrations/deployment.yaml)
- webapp deployment (murally)
```yaml
# OLD:
image: 'mural.azurecr.io/tactivos/mural-api:{{ versions["mural-api"] | default("latest") }}'

# NEW:
image: 'muralci.azurecr.io/tactivos/mural-api:{{ versions["mural-api"] | default("latest") }}'
```


### Pattern 2 Repos: Minimal Changes

For repos already using `docker/metadata-action` with commit SHA tags, only need to:

1. **Create testing-env.yml** for repos that don't have one:

   - mural-notifier
   - pdf-import
   - banksy
   - test-envs-operator (if it needs one)

2. **Verify testing-env.yml** points to build job (most already do)

3. **No deployment template changes** - already using production registry with SHA tags

#### Example: Create testing-env.yml for mural-notifier

[mural-notifier/testing-env.yml](mural/mural-notifier/testing-env.yml):

```yaml
requiredChecks:
- Build mural-notifier
```

## Why This Works

### Version Resolution Flow (Unchanged)

1. **effort-center** fetches commit SHA from GitHub: `abc123def456`
2. **test-envs-operator** uses that as image tag: `tag: abc123def456`
3. **Deployment template** references: `image: {registry}/tactivos/{service}:abc123def456`
4. **Kubernetes** pulls from specified registry with commit SHA tag

### Image Tagging After Changes

**Pattern 1 (mural-api) - CI Registry:**

```
muralci.azurecr.io/tactivos/mural-api:12345-67     ← run ID tag (existing)
muralci.azurecr.io/tactivos/mural-api:abc123def456 ← commit SHA tag (NEW)
```

**Pattern 2 (realtime) - Production Registry:**

```
mural.azurecr.io/tactivos/mural-realtime:feature-branch   ← branch tag (existing)
mural.azurecr.io/tactivos/mural-realtime:abc123def456     ← commit SHA tag (existing)
```

Both patterns end up with commit SHA tags, just in different registries.

## Implementation Steps

### Phase 1: Pattern 1 Repos (Breaking Change)

1. **Add commit SHA tagging to production build jobs:**

   - mural-api (4 jobs: api, worker, migrations, 8ball)
   - murally (1 job: production-build)
   - mural-integrations (1 job: build-production-image)

2. **Update testing-env.yml files** to point to build jobs

3. **Update deployment templates** to use CI registry

4. **Verify CI registry credentials** in test-envs-operator

5. **Test with single PR** to validate end-to-end flow

### Phase 2: Pattern 2 Repos (Minor Changes)

1. **Create missing testing-env.yml files:**

   - mural-notifier
   - pdf-import
   - banksy
   - test-envs-operator (if needed)

2. **Verify existing testing-env.yml** point to correct jobs

3. **Test with PRs** from these repos

## Benefits

- **Faster provisioning**: Environments available 5-15 minutes earlier
- **Parallel work**: Developers test while CI runs
- **Same images**: CI/production registry images are identical
- **Pattern-aware**: Leverages existing tagging where available
- **Minimal disruption**: Pattern 2 repos need almost no changes

## Risks & Mitigation

### Risk 1: Untested Code in Environments

**Risk**: Developers may provision environments before tests pass.

**Mitigation**:

- Desired behavior for faster feedback
- Tests still run and report failures
- Can re-provision after tests pass
- Consider notification when tests fail

### Risk 2: CI Registry Access (Pattern 1 Only)

**Risk**: Test environments may lack `muralci.azurecr.io` credentials.

**Mitigation**:

- Verify imagePullSecrets configuration
- Add CI registry credentials if needed
- Test with single environment first

### Risk 3: Job Name Mismatches

**Risk**: GitHub check names may not match testing-env.yml entries.

**Mitigation**:

- Verify exact job names from workflow runs
- Use job display names, not job IDs
- Test provisioning after changes

## Repository Change Summary

| Repository | Pattern | testing-env.yml | Deployment Template | Workflow Changes |

|------------|---------|-----------------|---------------------|------------------|

| mural-api | 1 | Update | muralci registry | Add SHA tagging |

| murally | 1 | Update | muralci registry | Add SHA tagging |

| mural-integrations | 1 | Update | muralci registry | Add SHA tagging |

| mural-realtime | 2 | Verify | No change | None |

| mural-antivirus | 2 | Verify | No change | None |

| mural-notifier | 2 | Create | No change | None |

| mural-upload | 2 | Verify | No change | None |

| pdf-import | 2 | Create | No change | None |

| banksy | 2 | Create | No change | None |

| test-envs-operator | 2 | Create? | No change | None |

## Open Questions

1. Does test-envs-operator need a testing-env.yml file to provision itself?
2. Are muralci.azurecr.io credentials already configured in test environments?
3. Should we add notifications when CI tests fail after provisioning?
4. What are the exact GitHub check display names vs job IDs?