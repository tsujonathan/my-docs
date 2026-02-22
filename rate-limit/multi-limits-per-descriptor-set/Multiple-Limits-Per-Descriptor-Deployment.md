## Deployment Strategy

### Environment Progression
Deploy using standard DevOps pipeline with extended validation at each stage:

**1. Development Environment (`development`)**
- **Scope:** PR #1-2 deploy immediately after merge
- **Validation:** Config manager can parse new formats
- **Duration:** 1 day observation

**2. Integration Environment (`integration`)**
- **Scope:** PR #3-8 deploy with feature flag OFF
- **Validation:**
  - Existing configs continue working
  - No performance regression (p95 latency < 50ms)
  - No error rate increase
- **Duration:** 2 days observation
- **Rollback Trigger:** Any error rate increase > 0.1%

**3. Feature Flag Enablement in Integration**
- **Scope:** Turn ON `ENABLE_MULTIPLE_LIMITS_PER_DESCRIPTOR` feature flag
- **Validation:**
  - Deploy test config with multiple limits (shadow mode)
  - Verify descriptor_index appears in responses
  - Confirm old clients (positional matching) still work
- **Duration:** 3 days observation
- **Rollback Trigger:** Any backward compatibility issue

**4. Stage Environment (`stage`)**
- **Scope:** Full deployment with feature flag ON
- **Validation:**
  - Load testing with 10x production traffic
  - Simulate 5+ production config patterns
  - Validate metrics and alerts work correctly
- **Duration:** 5 days observation
- **Rollback Trigger:** Performance degradation > 10%

**5. Production Canary Deployment (`production`)**
- **Scope:** 10% traffic to canary pods
- **Selection:** Start with low-traffic regions (japanwest-a, canadaeast-b)
- **Validation:**
  - Monitor error rates, latency, Redis operations
  - Compare canary vs. stable metrics (< 5% difference)
  - Verify backward compatibility with production clients
- **Duration:** 7 days observation
- **Rollback Trigger:** Canary error rate > stable + 0.05%

**6. Production Full Rollout**
- **Scope:** Gradual rollout: 25% → 50% → 100% (2 days per step)
- **Regions:** Prioritize US regions, then EMEA, then APAC
- **Validation:** Continuous monitoring of golden signals
- **Rollback Trigger:** Any region error rate > baseline + 0.1%

### Feature Flag Management
- **Flag Name:** `ENABLE_MULTIPLE_LIMITS_PER_DESCRIPTOR`
- **Type:** Boolean environment variable
- **Default:** `false` (disabled)
- **Lifecycle:**
  - Week 1-2: OFF in all environments (dormant code)
  - Week 3: ON in integration for testing
  - Week 4: ON in stage
  - Week 5-6: ON in production canary → full rollout
  - Week 10+: Remove flag after stable for 4 weeks

---

## Monitoring & Observability

### New Metrics to Add

**Rate Limit Processing Metrics:**
```
# Total limits evaluated per request (1 for single, N for multiple)
ratelimit.limits_evaluated.count
  Tags: service, descriptor_key

# Distribution of limits per descriptor
ratelimit.limits_per_descriptor.histogram
  Tags: service, descriptor_count_bucket (1, 2-3, 4-5, 6+)

# Cache operations per request (should increase proportionally)
ratelimit.redis_operations.count
  Tags: operation_type (get, set, decrement), limits_count

# Response building latency
ratelimit.response_build.duration_ms
  Tags: descriptor_count, total_limits
```

**Backward Compatibility Metrics:**
```
# Responses with multiple limits (for monitoring adoption)
ratelimit.multi_limit_responses.count
  Tags: descriptor_count, max_limits_per_descriptor

# Descriptor index usage (track new vs old clients)
ratelimit.descriptor_index_populated.count
  Tags: has_descriptor_index (true/false)
```

**Error & Validation Metrics:**
```
# Config validation errors (if multiple limits misconfigured)
ratelimit.config_validation.errors
  Tags: error_type (duplicate_window, missing_unit, invalid_limit)

# Cache key uniqueness violations (should be zero)
ratelimit.cache_key_collisions.count
  Tags: descriptor_key
```

### Alerts to Configure

**Critical Alerts (P1 - Immediate Action):**
1. **Error Rate Increase:** `ratelimit.errors.rate > baseline + 0.5%` for 5 minutes
2. **Cache Key Collisions:** `ratelimit.cache_key_collisions.count > 0` (any occurrence)
3. **Response Build Failures:** `ratelimit.response_build.errors > 10/min`

**Warning Alerts (P2 - Investigation Required):**
1. **Latency Degradation:** `p95(ratelimit.response_build.duration_ms) > 50ms` for 10 minutes
2. **Redis Operations Spike:** `ratelimit.redis_operations.count > 3x average` for 5 minutes
3. **Config Load Failures:** `ratelimit.config_validation.errors > 5/hour`

**Informational Alerts (P3 - Awareness):**
1. **Multi-Limit Adoption:** `ratelimit.multi_limit_responses.count` (daily report)
2. **Limits Per Descriptor Distribution:** Weekly histogram report

### Dashboard Additions

**"Multiple Limits Overview" Dashboard:**
- **Panel 1:** Requests with multiple limits over time (line chart)
- **Panel 2:** Distribution of limits per descriptor (histogram)
- **Panel 3:** Average Redis operations per request (line chart)
- **Panel 4:** Response building latency by limit count (heatmap)

**"Backward Compatibility" Dashboard:**
- **Panel 1:** Old client requests (descriptor_index not used) vs new clients
- **Panel 2:** OVER_LIMIT ordering validation (first N statuses = N descriptors)
- **Panel 3:** Error rate comparison: single vs multi-limit configs

**Existing Dashboard Updates:**
- Add "Limits Per Descriptor" filter to main RateLimit dashboard
- Include multi-limit response counts in executive summary panel

---

## Migration Guide for Config Owners

### When to Use Multiple Limits

**Use Cases:**
1. **Burst Protection:** Short-term (1 second) + medium-term (1 minute) + long-term (1 hour) limits for the same operation
2. **Tiered Throttling:** Different thresholds for different time windows (e.g., 100/minute AND 5000/hour)
3. **Progressive Rate Limiting:** Allow bursts but enforce daily caps

**When NOT to Use:**
- Different operations (use separate descriptors instead)
- Redundant time windows (e.g., both 60-second and 1-minute limits)

### Config Migration Examples

**Before (Single Limit):**
```yaml
descriptors:
  - key: ACCOUNT_ID
    rate_limit:
      unit: hour
      requests_per_unit: 1000
```

**After (Multiple Limits):**
```yaml
descriptors:
  - key: ACCOUNT_ID
    rate_limit:
      - unit: second
        requests_per_unit: 20      # Burst protection
      - unit: minute
        requests_per_unit: 500     # Medium-term limit
      - unit: hour
        requests_per_unit: 1000    # Original long-term limit
```

**Nested Descriptor Example:**
```yaml
descriptors:
  - key: UNIQUE_SERVICE_ID
    value: "my-service"
    descriptors:
      - key: ACCOUNT_ID
        rate_limit:
          - unit: second
            requests_per_unit: 5
            name: "account-burst-limit"
          - unit: hour
            requests_per_unit: 1000
            name: "account-hourly-limit"
```

### Validation Checklist

Before deploying configs with multiple limits:

- [ ] **Unique Time Windows:** No two limits have identical `unit` + `length_of_time` (causes cache key collision)
- [ ] **Logical Ordering:** Limits are ordered from strictest to most relaxed (e.g., second → minute → hour)
- [ ] **Reasonable Thresholds:** Each limit makes business sense (e.g., 10/sec = 600/min, ensure 600 < hourly limit)
- [ ] **Name Field Populated:** Each limit has descriptive `name` for debugging (recommended)
- [ ] **Shadow Mode Testing:** Test with `shadow_mode: true` first to validate behavior without blocking traffic
- [ ] **Client Compatibility:** Verify downstream clients support either positional or descriptor_index matching
- [ ] **Cache Key Uniqueness:** Confirm no other descriptors would generate identical cache keys

### Testing New Configs

**Step 1: Deploy to Integration with Shadow Mode**
```yaml
rate_limit:
  - unit: second
    requests_per_unit: 10
    shadow_mode: true  # No actual blocking
```

**Step 2: Validate Responses**
- Check logs for `descriptor_index` values (0, 1, 2, etc.)
- Verify each limit status appears in response
- Confirm OVER_LIMIT statuses appear first (if triggered)

**Step 3: Monitor Metrics**
- `ratelimit.limits_evaluated.count` should equal number of limits
- `ratelimit.redis_operations.count` should match expected cache hits

**Step 4: Remove Shadow Mode**
```yaml
rate_limit:
  - unit: second
    requests_per_unit: 10
    shadow_mode: false  # Enable blocking
```

**Step 5: Gradual Rollout to Production**
- Deploy to 1 region first
- Monitor for 24 hours
- Expand to all regions

---

## Rollback Plan

### Rollback Scenarios & Procedures

**Scenario 1: Proto Breaking Change Detected**
- **Symptom:** Clients fail to deserialize responses
- **Procedure:**
  1. Revert PR #1 (proto changes)
  2. Redeploy previous proto version
  3. Clear any cached proto definitions
- **Impact:** None - field was added, not modified
- **Likelihood:** Very Low (optional field is backward compatible)

**Scenario 2: Config Manager Fails to Parse New Configs**
- **Symptom:** ConfigMap generation errors in integration
- **Procedure:**
  1. Revert PR #2 (config manager changes)
  2. Redeploy config-manager sidecar
  3. Revert any multi-limit configs to single limit format
- **Impact:** Config updates blocked until fixed
- **Likelihood:** Low (extensive unit testing)

**Scenario 3: Feature Flag Enabled - Performance Degradation**
- **Symptom:** Latency increase > 10%, Redis connection saturation
- **Procedure:**
  1. Set `ENABLE_MULTIPLE_LIMITS_PER_DESCRIPTOR=false` via environment variable
  2. Restart ratelimit service pods (rolling restart)
  3. Validate latency returns to baseline
- **Impact:** Multi-limit configs revert to single limit behavior (first limit only)
- **Likelihood:** Low (tested in stage with 10x traffic)

**Scenario 4: Production Canary Shows Increased Errors**
- **Symptom:** Canary error rate > stable + 0.05%
- **Procedure:**
  1. Route 100% traffic to stable pods
  2. Terminate canary deployment
  3. Investigate root cause in logs
  4. Fix issue, redeploy to stage for validation
- **Impact:** None to users (canary handles minimal traffic)
- **Likelihood:** Medium (production environment variations)

**Scenario 5: Full Production Rollout - Critical Bug**
- **Symptom:** Widespread rate limiting failures, incorrect blocking decisions
- **Procedure:**
  1. **Immediate:** Set feature flag OFF in all production regions (5 minutes)
  2. **Short-term (1 hour):** Revert PRs #3-8, redeploy stable version
  3. **Medium-term (24 hours):** Revert all multi-limit configs to single limit
  4. **Post-mortem:** Analyze logs, add missing tests, fix bug
- **Impact:** Multi-limit configs stop working, single-limit configs unaffected
- **Likelihood:** Very Low (extensive testing + canary validation)

### Backward Compatibility Guarantees

**Old Clients (No descriptor_index support):**
- ✅ Continue working via positional matching
- ✅ Receive one status per descriptor (first OVER_LIMIT or first OK)
- ✅ No code changes required
- ✅ Performance unchanged

**New Clients (descriptor_index aware):**
- ✅ Receive all limit statuses for each descriptor
- ✅ Can implement finer-grained logic (e.g., retry after shortest limit resets)
- ✅ Optional migration timeline

**Rollback Impact on Clients:**
- **Feature Flag OFF:** Responses revert to single status per descriptor (old behavior)
- **Proto Revert:** New clients must ignore descriptor_index field or handle missing field
- **Config Revert:** Multi-limit configs emit warning and use first limit only

### Data Consistency During Rollback

**Redis Cache:**
- No cleanup needed - old cache keys remain valid
- New cache keys (with window length) expire naturally per TTL
- No data migration required

**Configuration Files:**
- Multi-limit YAML configs remain valid even with feature flag OFF
- Service will use first limit from array when flag is disabled
- Manual config revert only needed if permanently abandoning feature

---

## Performance Benchmarks

### Expected Performance Impact

**Request Processing Latency:**
| Scenario | Current (Single Limit) | Target (Multiple Limits) | Overhead |
|----------|------------------------|--------------------------|----------|
| 1 descriptor, 1 limit | 5ms p95 | 5ms p95 | 0% |
| 1 descriptor, 3 limits | 5ms p95 | 8ms p95 | +60% |
| 1 descriptor, 5 limits | 5ms p95 | 12ms p95 | +140% |
| 3 descriptors, 2 limits each | 8ms p95 | 15ms p95 | +87% |
| 5 descriptors, 1 limit each | 12ms p95 | 12ms p95 | 0% |

**Redis Operations:**
| Limits Per Descriptor | Operations Per Request | Notes |
|----------------------|------------------------|-------|
| 1 | 1-2 (GET, optional SET) | Current baseline |
| 2 | 2-4 (2x GET, optional 2x SET) | Linear scaling |
| 3 | 3-6 | Linear scaling |
| 5 | 5-10 | Expected max in practice |

**Memory Usage:**
| Component | Current | With Multi-Limits (avg 2.5 limits/desc) | Increase |
|-----------|---------|------------------------------------------|----------|
| Config Manager | 100 MB | 150 MB | +50% |
| RateLimit Service (Go) | 250 MB | 300 MB | +20% |
| Redis Cache Keys | 1 GB | 2.5 GB | +150% |

### Acceptance Criteria

**Must Meet Before Production Rollout:**
1. ✅ **Latency:** p95 < 50ms for requests with ≤5 limits per descriptor (tested in stage)
2. ✅ **Error Rate:** No increase > 0.1% compared to baseline (validated in canary)
3. ✅ **Redis Load:** Connection pool utilization < 70% under 10x traffic (load tested)
4. ✅ **Memory:** Service memory usage < 500 MB per pod (headroom for traffic spikes)
5. ✅ **Throughput:** 10,000 req/sec sustained with 3 limits per descriptor (stage test)

**Performance Optimization Opportunities:**
- **Parallel Redis Operations:** Fetch all limits concurrently instead of sequentially (future PR)
- **Cache Response Building:** Cache entire response for identical descriptor sets (future PR)
- **Limit Evaluation Short-Circuit:** Stop evaluating after first OVER_LIMIT if client doesn't need all statuses (opt-in)

### Load Testing Plan

**Stage Environment Tests (Before Production):**
1. **Baseline Traffic:** 5,000 req/sec, single limit configs → Record p50, p95, p99 latencies
2. **Multi-Limit Traffic:** 5,000 req/sec, 50% single limit / 50% multi-limit (2-3 limits) → Compare to baseline
3. **Peak Traffic:** 10,000 req/sec, 100% multi-limit (3 limits) → Validate < 50ms p95
4. **Stress Test:** 20,000 req/sec (2x production peak) → Identify breaking point
5. **Sustained Load:** 5,000 req/sec for 24 hours → Validate no memory leaks

**Production Canary Validation:**
- Monitor real traffic patterns for 7 days
- Compare canary vs stable metrics (latency, error rate, Redis ops)
- Acceptance: Canary metrics within ±5% of stable

**Post-Rollout Monitoring:**
- Daily latency reports for first 30 days
- Weekly performance review meetings
- Adjust thresholds based on real-world usage patterns
