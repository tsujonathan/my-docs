# Introduction

#### Summary

The proposal outlines adding support for multiple rate limits per descriptor (currently only one is supported). Key changes include:

#### Business Justification

**Use Case**: Prevent burst abuse while allowing sustained usage

**Example Scenario**: AI search endpoint
- **Minute limit**: 250 requests/minute (prevent burst abuse, DoS attacks)
- **Hour limit**: 5000 requests/hour (allow reasonable sustained usage)

**Rationale**: A single limit cannot satisfy both requirements:
- Only minute limit (250/min = 15,000/hour): Too restrictive for legitimate sustained use
- Only hour limit (5000/hour ≈ 83/min): Vulnerable to burst attacks within short time windows

**Expected Behavior**: If ANY limit is exceeded, the request is rejected (`CODE_OVER_LIMIT`)

#### New Features

* Multiple rate\_limit configs per descriptor, each should have a unique “limit\_window\_length\_in\_seconds” value  
* Modified response format with multiple statuses per descriptor using **flat array structure**
* New `descriptor_index` field in response statuses for correlation (backward compatible)
* Updated cache key generation: domain \+ "\_" \+ descriptor \+ "\_" \+ "limit\_window\_length\_in\_seconds" \+ "\_"

#### Design Principles

* **Backward Compatibility**: No breaking changes to protobuf definitions
* **Flat Array Response**: Maintains existing structure, adds optional `descriptor_index` field
* **Gradual Migration**: Old clients continue working without changes
* **Cache Key Uniqueness**: Each limit per descriptor gets unique cache key via limit window length

#### Details

1. [New limit config format](#new-limit-config-format)  
2. [Request & response](#request--response)  
3. [Cache key generation](#cache-key-generation)  
4. [ETA](#eta)

# New Limit Config Format

#### New limit config format

* Each descriptor can have multiple limit configs  
* Each limit config should have a unique logical “limit\_window\_length\_in\_seconds”

```
configMap:
  enabled: true
  name: app-configmap
  filename: config
  mountpath: /default-configs/ratelimit/config/
  content: |
    domain: sample-domain
    descriptors:
    - key: URLPATH
      detailed_metric: true
      value: "/webforms-api"
      rate_limit:
      - unit: minute
        requests_per_unit: 1
      - unit: hour
        requests_per_unit: 2
    - key: remote_address
      detailed_metric: true
      descriptors:
      - key: USERAGENT
        descriptors:
          - key: URLPATH
            value: "/webforms-ui"
            rate_limit:
            - unit: minute
              requests_per_unit: 1
            - unit: second
              requests_per_unit: 2
            shadow_mode: true
```

# Request & Response

Request:

```json
{
  "domain": "api-gateway",
  "descriptors": [
    {
      "entries": [
        {
          "key": "user_id",
          "value": "user_12345"
        }
      ]
    }
  ],
  "hits_addend": 1,
  "stop_increment_over_limit": false
}
```

Response (user has 2 limits configured: per-minute and per-hour):

```json
{
  "overall_code": "CODE_OVER_LIMIT",
  "statuses": [
    {
      "code": "CODE_OVER_LIMIT",
      "current_limit": {
        "name": "user_per_minute",
        "requests_per_unit": 100,
        "unit": "MINUTE",
        "length_of_time": 1
      },
      "limit_remaining": 0,
      "duration_until_reset": "15s",
      "descriptor_index": 0
    },
    {
      "code": "CODE_OK",
      "current_limit": {
        "name": "user_per_hour",
        "requests_per_unit": 5000,
        "unit": "HOUR",
        "length_of_time": 1
      },
      "limit_remaining": 4456,
      "duration_until_reset": "1847s",
      "descriptor_index": 0
    }
  ],
  "raw_body": "UmF0ZSBsaW1pdCBleGNlZWRlZC4gUGxlYXNlIHRyeSBhZ2FpbiBpbiAxNSBzZWNvbmRzLg=="
}
```

**Note**: Since descriptor 0 has OVER_LIMIT status, it appears first (priority ordering for old client compatibility)

#### Response Structure Notes

* `statuses` is a **flat array** - maintains backward compatibility
* Each status includes `descriptor_index` to correlate with request descriptors (0-based)
  * `descriptor_index` refers to position in `request.descriptors` array
  * One descriptor with multiple entries (e.g., `[{"key":"URLPATH"},{"key":"user_id"}]`) = 1 descriptor, not 2
* Multiple statuses can have the same `descriptor_index` (multiple limits per descriptor)
* **Status ordering for backward compatibility with old clients**:
  * First, include one status for each descriptor in order: descriptor_index 0, 1, 2, ..., N
  * Within each descriptor's statuses, **OVER_LIMIT statuses have priority** (appear first)
  * Then append remaining statuses (additional limits) for each descriptor
  * This ensures old clients (using positional matching) get at least one status per descriptor
* **`overall_code` logic**:
  * `CODE_OVER_LIMIT` if **ANY** limit is exceeded across all descriptors
  * `CODE_OK` **only if ALL** limits pass for all descriptors
  * This ensures the request is rejected if any configured limit is violated

**Example ordering**: Request with 2 descriptors [2 limits each]
```json
{
  "statuses": [
    {"code": "CODE_OK", "descriptor_index": 0},         // Descriptor 0, limit 1 (first for old clients)
    {"code": "CODE_OVER_LIMIT", "descriptor_index": 1},  // Descriptor 1, limit 1 (first for old clients)
    {"code": "CODE_OVER_LIMIT", "descriptor_index": 0}   // Descriptor 0, limit 2 (additional)
  ]
}
```
Old clients: Get statuses[0] for descriptor 0 (OK) and statuses[1] for descriptor 1 (OVER_LIMIT) ✅

#### Multi-Descriptor Example

Request with 2 descriptors:

```json
{
  "domain": "api-gateway",
  "descriptors": [
    {
      "entries": [{"key": "user_id", "value": "user_123"}]
    },
    {
      "entries": [{"key": "api_key", "value": "key_456"}]
    }
  ]
}
```

Response (descriptor 0 has 2 limits, descriptor 1 has 2 limits):

```json
{
  "overall_code": "CODE_OK",
  "statuses": [
    {
      "code": "CODE_OK",
      "current_limit": {"name": "user_per_minute", "requests_per_unit": 100, "unit": "MINUTE"},
      "limit_remaining": 45,
      "descriptor_index": 0
    },
    {
      "code": "CODE_OK",
      "current_limit": {"name": "api_per_second", "requests_per_unit": 10, "unit": "SECOND"},
      "limit_remaining": 8,
      "descriptor_index": 1
    },
    {
      "code": "CODE_OK",
      "current_limit": {"name": "user_per_hour", "requests_per_unit": 5000, "unit": "HOUR"},
      "limit_remaining": 4321,
      "descriptor_index": 0
    },
    {
      "code": "CODE_OK",
      "current_limit": {"name": "api_per_day", "requests_per_unit": 100000, "unit": "DAY"},
      "limit_remaining": 99876,
      "descriptor_index": 1
    }
  ]
}
```

**Note**: Ordering ensures old clients get statuses[0] for descriptor 0 and statuses[1] for descriptor 1

# Cache Key Generation

#### Cache key generation

* **Format**: `domain + "_" + descriptor + "_" + limit_window_length_in_seconds + "_"`
* **Purpose**: Each limit per descriptor gets a unique cache key to track independently
* This approach ensures ordering reliability and proper isolation between different time windows

#### Limit Window Length Calculation

`limit_window_length_in_seconds = unit_in_seconds × length_of_time`

**Unit conversions:**
- `second` (proto: `SECOND`) = 1
- `minute` (proto: `MINUTE`) = 60
- `hour` (proto: `HOUR`) = 3600
- `day` (proto: `DAY`) = 86400
- `infinite` (proto: `INFINITE`) = no expiration

**Examples:**
```yaml
- unit: minute
  requests_per_unit: 250
  length_of_time: 1      # 60 × 1 = 60 seconds

- unit: hour
  requests_per_unit: 5000
  length_of_time: 1      # 3600 × 1 = 3600 seconds
  
- unit: minute
  requests_per_unit: 500
  length_of_time: 5      # 60 × 5 = 300 seconds
```

#### Cache Key Examples

For a descriptor `user_id=user_12345` with 3 limits:

```
# Minute limit (60 seconds)
domain_user_id_user_12345_60_

# Five-minute limit (300 seconds)
domain_user_id_user_12345_300_

# Hour limit (3600 seconds)
domain_user_id_user_12345_3600_
```

For a nested descriptor `URLPATH=/search` → `user_id=user_12345` with 2 limits:

```
# Minute limit
domain_URLPATH_/search_user_id_user_12345_60_

# Day limit
domain_URLPATH_/search_user_id_user_12345_86400_
```

# Edge Cases & Constraints

#### Configuration Constraints

* **Max limits per descriptor**: Recommended 5, maximum 10 (to control response size and cache lookups)
* **Unique time windows required**: Each limit must have unique `limit_window_length_in_seconds`
  * **Validation**: Happens at **config load time** (startup)
  * **Error handling**: **Reject entire config** if duplicate window lengths detected
  * **Implementation**: Reuse current validation approach in C# config-manager code
  * **Example invalid config**: Two limits with `unit: minute, length_of_time: 1` → Config rejected
* **Limit name field**: Optional (can be omitted)
* **Shadow mode**: Applies to **all limits for a descriptor** (not per-limit)
  * Per-limit shadow mode is a future enhancement
* **Performance optimization**: Multi-cache-key operations, response size limits - tracked as TODO item

#### Behavior Specification

**Descriptor with no limits configured:**
- Returns **one empty status record** for that descriptor (required for fail-open behavior)
- Status will have minimal fields: `{"code": "CODE_OK", "descriptor_index": N}`
- `overall_code = CODE_OK` if no other descriptors are over limit

**Mixing single and multiple limit configs:**
```yaml
descriptors:
  - key: user_id
    rate_limit:          # Single limit (backward compatible)
      unit: minute
      requests_per_unit: 100
  - key: api_key
    rate_limit:          # Multiple limits
      - unit: second
        requests_per_unit: 10
      - unit: hour
        requests_per_unit: 10000
```
✅ **Supported**: Both formats work in same configuration

**Decrement operation:**
- Decrements **all** limits for the affected descriptor(s)
- Decrement amount applies independently to each limit (e.g., decrement by 5 → each limit reduced by 5)
- Uses same cache key generation logic
- If a limit has expired, skip decrement for that cache key only

**Example**: Decrement request with amount=5, descriptors=[2 limits, 3 limits]
- Generates 5 cache keys total
- Decrements 5 from each cache entry

**GetLimit function returns:**
- `[]*RateLimit` (array of pointers)
- Returns `nil` or empty array `[]` when no limits configured
- Ordering: Within a descriptor, OVER_LIMIT limits appear first for priority

#### Old Client Behavior

**Old client receives multiple statuses:**
```go
// Old client code typically does:
for i, descriptor := range request.Descriptors {
    status := response.Statuses[i]  // Gets first status only
    if status.Code == CODE_OVER_LIMIT {
        return error
    }
}
```
✅ **Works correctly**: Gets first status per descriptor (positional), may miss additional limits but still enforces at least one

**Old server → New client:**
- **Old server returns 1 status per descriptor** (single limit only)
- New client receives statuses without `descriptor_index` field (defaults to 0)
- Client should fall back to positional matching when all descriptor_index == 0
- Detectable: `len(statuses) == len(descriptors)` and all `descriptor_index == 0`

**GetCount Response:**
- Uses same structure as ShouldRateLimit response
- Flat array with `descriptor_index`
- Multiple statuses per descriptor
- Same ordering rules apply

#### Response Building Algorithm (Pseudo-code)

```go
func buildResponse(request *RateLimitRequest) *RateLimitResponse {
    allStatuses := []DescriptorStatus{}
    firstStatusPerDescriptor := []DescriptorStatus{}  // For old client compatibility
    additionalStatuses := []DescriptorStatus{}
    overallCode := CODE_OK
    
    // Step 1: Evaluate all limits for all descriptors
    for descriptorIndex, descriptor := range request.Descriptors {
        limits := GetLimit(descriptor)  // Returns []*RateLimit
        
        if len(limits) == 0 {
            // No limits configured - add empty status for fail-open
            firstStatusPerDescriptor = append(firstStatusPerDescriptor, DescriptorStatus{
                Code: CODE_OK,
                DescriptorIndex: descriptorIndex,
            })
            continue
        }
        
        // Evaluate each limit
        statuses := []DescriptorStatus{}
        for _, limit := range limits {
            status := evaluateLimit(descriptor, limit)
            status.DescriptorIndex = descriptorIndex
            statuses = append(statuses, status)
            
            if status.Code == CODE_OVER_LIMIT {
                overallCode = CODE_OVER_LIMIT
            }
        }
        
        // Step 2: Separate first status (for old clients) from additional
        // Priority: OVER_LIMIT statuses first
        firstStatus, remaining := extractFirstStatus(statuses)  // Picks OVER_LIMIT if exists
        firstStatusPerDescriptor = append(firstStatusPerDescriptor, firstStatus)
        additionalStatuses = append(additionalStatuses, remaining...)
    }
    
    // Step 3: Build final flat array with proper ordering
    // First: one status per descriptor (old client compatibility)
    // Then: remaining statuses
    allStatuses = append(allStatuses, firstStatusPerDescriptor...)
    allStatuses = append(allStatuses, additionalStatuses...)
    
    return &RateLimitResponse{
        OverallCode: overallCode,
        Statuses: allStatuses,
    }
}

func extractFirstStatus(statuses []DescriptorStatus) (first DescriptorStatus, remaining []DescriptorStatus) {
    // Find first OVER_LIMIT status (priority)
    for i, status := range statuses {
        if status.Code == CODE_OVER_LIMIT {
            first = status
            remaining = append(statuses[:i], statuses[i+1:]...)
            return
        }
    }
    // No OVER_LIMIT found, take first status
    first = statuses[0]
    remaining = statuses[1:]
    return
}
```

**Key points:**
- Ensures first N statuses correspond to N descriptors (positional for old clients)
- OVER_LIMIT statuses have priority within each descriptor
- `descriptor_index` populated for all statuses
- Descriptors with no limits get empty status record

# ETA

### Required Changes to Support Multiple Limits per Descriptor

The following changes will roughly require 25 working days (unit tests are included)

#### Backward Compatibility Strategy

* **Non-breaking change**: Add optional `descriptor_index` field to `DescriptorStatus` message
* **Flat array structure**: Maintains existing `repeated DescriptorStatus statuses` (no nesting)
* **Old clients**: Continue working - ignore new `descriptor_index` field, use positional matching
* **New clients**: Use `descriptor_index` to group multiple statuses per descriptor
* **No forced migration**: Clients can upgrade gradually at their own pace

#### Implementation Tasks

* Add `descriptor_index` field to proto definitions (non-breaking). (1d)
  * **Proto change**: Add `uint32 descriptor_index = 5;` to DescriptorStatus messages
  * Use next available field number (verify field 5 is not used)
  * Field is **optional** by default in proto3 (omitted = defaults to 0)
  * **Unit enum values** (for reference): `SECOND`, `MINUTE`, `HOUR`, `DAY`, `MONTH`, `YEAR`, `INFINITE`
  * Common proto
    * Add field to RateLimitResponse.DescriptorStatus ([link](https://github.docusignhq.com/Microservices/ratelimit/blob/18c0cd118faac1fb9a082c29a03dd655601acb95/ratelimit-service/protos/common/ratelimitservice.proto#L137))
    * Add field to GetCountResponse.DescriptorStatus ([link](https://github.docusignhq.com/Microservices/ratelimit/blob/18c0cd118faac1fb9a082c29a03dd655601acb95/ratelimit-service/protos/common/ratelimitservice.proto#L200))
  * V1 proto
    * Add field to ShouldRateLimitResponse.DescriptorStatus ([link](https://github.docusignhq.com/Microservices/ratelimit/blob/18c0cd118faac1fb9a082c29a03dd655601acb95/ratelimit-service/protos/v1/ratelimitservice.proto#L50)) 
    * Add field to GetCountResponse.DescriptorStatus ([link](https://github.docusignhq.com/Microservices/ratelimit/blob/18c0cd118faac1fb9a082c29a03dd655601acb95/ratelimit-service/protos/v1/ratelimitservice.proto#L100))


* Update the Config Manager to support multiple limits in the generated YAML file. (3d)  
  * RateLimitConfig needs to support multiple RateLimit ([link](https://github.docusignhq.com/Microservices/ratelimit/blob/18c0cd118faac1fb9a082c29a03dd655601acb95/config-manager/src/RateLimit.ConfigManager/Models/RateLimitConfig.cs#L44))  
  * The CreateConfig method in the RateLimitConfigGenerator class needs to support creating multiple RateLimit instances ([link](https://github.docusignhq.com/Microservices/ratelimit/blob/18c0cd118faac1fb9a082c29a03dd655601acb95/config-manager/src/RateLimit.ConfigManager/Services/RateLimitConfigGenerator.cs#L88))  
  * Please refer to the "New Limit Config Format" section for the new limit yaml format. 

   

* Update the ratelimit services to support multiple limits per descriptor in responses. (3d)  
   * Update response building logic to populate `descriptor_index` for each status
   * Ensure flat array structure with proper correlation between statuses and descriptors
   * Update the vNext service ([link](https://github.docusignhq.com/Microservices/ratelimit/blob/18c0cd118faac1fb9a082c29a03dd655601acb95/ratelimit-service/src/service/vNext/ratelimitvnext.go#L32))  
   * Update the common service ([link](https://github.docusignhq.com/Microservices/ratelimit/blob/18c0cd118faac1fb9a082c29a03dd655601acb95/ratelimit-service/src/service/ratelimit.go#L214))

   

* Update the logic for loading limit configurations from YAML. (3d)  
   * type rateLimitDescriptor needs to support multiple RateLimit instances. ([link](https://github.docusignhq.com/Microservices/ratelimit/blob/18c0cd118faac1fb9a082c29a03dd655601acb95/ratelimit-service/src/config/config_impl.go#L44))
      
      **New Data Structure:**
      
      The diagram below shows the updated relationship where `rateLimitDescriptor` now supports multiple `RateLimit` instances:
      
      ```mermaid
      %%{init: {'theme':'base', 'themeVariables': { 'fontSize':'12px'}}}%%
      classDiagram
          rateLimitConfigImpl *-- rateLimitDomain
          rateLimitDomain *-- rateLimitDescriptor
          rateLimitDescriptor *-- rateLimitDescriptor : nested
          rateLimitDescriptor *-- "0..*" RateLimit : limits
          
          class rateLimitConfigImpl {
              +domains
              +statsManager
          }
          
          class rateLimitDomain
          
          class rateLimitDescriptor {
              +descriptors
              +limits []*RateLimit
          }
          
          class RateLimit {
              +Name
              +Limit
              +ShadowMode
          }
          
          style RateLimit fill:#f96,stroke:#333,stroke-width:2px
      ```
      
      **Key Change:** `rateLimitDescriptor.limit` (single `*RateLimit`) → `rateLimitDescriptor.limits` (array `[]*RateLimit`)
      
   * func loadDescriptors needs to load multiple limits per descriptor. ([link](https://github.docusignhq.com/Microservices/ratelimit/blob/18c0cd118faac1fb9a082c29a03dd655601acb95/ratelimit-service/src/config/config_impl.go#L157))

* Update the logic for finding limit configurations based on descriptors defined in a request. (2d)  
   * func GetLimit needs to be updated to return array of limits instead of single limit ([link](https://github.docusignhq.com/Microservices/ratelimit/blob/18c0cd118faac1fb9a082c29a03dd655601acb95/ratelimit-service/src/config/config_impl.go#L321))
   * Return signature: `GetLimit(...) []*RateLimit` instead of `GetLimit(...) *RateLimit`

   

* Update the logic for processing ShouldRateLimit and GetCount requests. (3d)  
   * func processRateLimit needs to iterate through multiple limits per descriptor ([link](https://github.docusignhq.com/Microservices/ratelimit/blob/18c0cd118faac1fb9a082c29a03dd655601acb95/ratelimit-service/src/service/ratelimit.go#L214))
   * Evaluate each limit independently and create separate status entries in flat array
   * Populate `descriptor_index` field for proper correlation
        
* Update the logic for processing Decrement requests. (1d)  
   * func Decrement needs to support multiple limits per descriptor ([link](https://github.docusignhq.com/Microservices/ratelimit/blob/18c0cd118faac1fb9a082c29a03dd655601acb95/ratelimit-service/src/redis/fixed_cache_impl.go#L202))

* Confirm whether the de-microservice-shared-chart needs to be updated for this change. ([link](https://github.docusignhq.com/Microservices/ds-microservice-shared-chart/blob/3b484ae31f5241a457e31ad2aa533bd21ef07069/templates/_helpers.tpl#L635)) (1d)  
     
* The Ratelimit-crd needs to be updated to support this change. ([link](https://github.docusignhq.com/Microservices/ratelimit-crd/blob/765777e97fb9827a9c32bcd03a29bbd4bb4ed51b/config/shared/app-values.yaml#L2)) (2d)  
     
* Integration and periodic tests need to be updated to accommodate this change. (4d)  
    * Test scenarios:
      1. Single descriptor, multiple limits (2-3 limits), one over limit → Verify overall_code, descriptor_index, and OVER_LIMIT appears in first position
      2. Multiple descriptors with varying limit counts ([1 limit, 3 limits], [2 limits]) → Verify flat array structure and ordering
      3. All limits OK → Verify overall_code = CODE_OK
      4. Shadow mode descriptor with multiple limits → Verify all limits shadowed
      5. **Backward compatibility**: Simulate old client (positional matching) with multi-limit response → Verify gets one status per descriptor
      6. Decrement operation with multiple limits, amount=5 → Verify all cache keys decremented by 5
      7. Edge cases: Descriptor with no limits (empty status record), single vs multiple limit config mixing
      8. **Response ordering**: Verify first N statuses match N descriptors for old client compatibility
    * Please refer to the "Request & Response" section for the new response format
         
* Update the logic for generating Redis cache keys - please refer to the "Cache Key Generation" section for details. (2d)

---

## PR Breakdown Strategy

To minimize risk and enable incremental review, the implementation will be split across 9 Pull Requests organized into 4 phases:

### Phase 1: Foundation (No Behavioral Changes)
**Goal:** Add new data structures without changing runtime behavior

**PR #1: Proto Definitions Update** (1 day)
- Add `descriptor_index` field to all DescriptorStatus messages
- Regenerate proto bindings for Go and C#
- Update proto documentation with field descriptions
- **Testing:** Verify proto compilation, no runtime tests needed
- **Risk:** Low - field is optional and backward compatible

**PR #2: Config Manager Model Updates** (2 days)
- Change `RateLimit` property to `List<RateLimit>` in `RateLimitConfig.cs`
- Add support for parsing both single and array formats (backward compatible)
- Update serialization logic to maintain existing YAML structure for single limits
- **Testing:** Unit tests for config parsing (single limit, multi-limit, mixed scenarios)
- **Risk:** Low - changes isolated to config manager

### Phase 2: Core Logic Updates (Controlled Rollout)
**Goal:** Update internal processing logic with feature flag protection

**PR #3: Go Config Loading Logic** (3 days)
- Update `rateLimitDescriptor` struct to use `limits []*RateLimit`
- Modify `loadDescriptors()` to parse limit/rate_limit arrays
- Update `GetLimit()` return signature to `[]*RateLimit`
- Add backward compatibility for single limit configs
- **Testing:** Unit tests for config loading with various YAML structures
- **Risk:** Medium - core config loading, but isolated from request processing

**PR #4: Cache Key Generation Updates** (2 days)
- Update cache key generation to include `limit_window_length_in_seconds`
- Ensure unique keys for limits with different time windows
- Add migration logic for existing cache keys (if needed)
- **Testing:** Unit tests for cache key uniqueness, integration tests
- **Risk:** Medium - cache key changes require careful validation
- **Note:** Must come before PR #5-7 as they depend on updated cache keys

**PR #5: Response Building Logic - ShouldRateLimit** (3 days)
- Update `processRateLimit()` to iterate through multiple limits
- Implement response building algorithm with `descriptor_index` population
- Add OVER_LIMIT priority ordering for backward compatibility
- Protect with feature flag `ENABLE_MULTIPLE_LIMITS_PER_DESCRIPTOR` (default: false)
- **Testing:** Unit tests for response building, multi-descriptor scenarios
- **Risk:** Medium-High - changes request processing, mitigated by feature flag
- **Dependencies:** Requires PR #3 (limits array) and PR #4 (cache keys)

**PR #6: Response Building Logic - GetCount** (2 days)
- Update GetCount endpoint to support multiple limits per descriptor
- Align response structure with ShouldRateLimit changes
- Reuse response building logic from PR #5
- **Testing:** Unit tests for GetCount with multiple limits
- **Risk:** Low-Medium - similar logic to PR #5

**PR #7: Decrement Logic Updates** (1 day)
- Update `Decrement()` to iterate through multiple limits per descriptor
- Ensure all cache keys are decremented properly
- **Testing:** Unit tests for decrement with multiple limits
- **Risk:** Low - straightforward iteration change
- **Dependencies:** Requires PR #4 (cache keys)

### Phase 3: Config Generation (Deployment Ready)
**Goal:** Complete end-to-end functionality for new configs

**PR #8: Config Manager YAML Generation** (3 days)
- Update `CreateConfig()` in `RateLimitConfigGenerator.cs`
- Generate YAML with `rate_limit` array for multiple limits
- Maintain single `rate_limit` format for configs with one limit (optional)
- **Testing:** Integration tests with config-manager sidecar
- **Risk:** Low-Medium - visible config changes, but backward compatible

### Phase 4: Testing & Validation (Production Ready)
**Goal:** Comprehensive testing and canary deployment preparation

**PR #9: Integration & Periodic Tests** (4 days)
- Add integration tests for all 8 test scenarios (see Implementation Tasks)
- Update periodic tests to validate production behavior
- Add performance benchmarks for multi-limit scenarios
- **Testing:** Full integration test suite, canary validation tests
- **Risk:** Low - test-only changes
