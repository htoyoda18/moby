# Recommendation: httpstatus Implementation Strategy

## Executive Summary

After analyzing the containerd/errdefs implementation and current Moby usage, I recommend **keeping and refining the current Moby implementation** rather than directly using `errhttp.ToHTTP()`.

## Key Findings

### 1. Asymmetric Client-Server Implementation
- **Client**: Already uses `containerd/errdefs/pkg/errhttp.ToNative()` (HTTP → error)
- **Server**: Uses custom `httpstatus.FromError()` (error → HTTP)
- **Risk**: Mismatched mappings could cause round-trip issues

### 2. Error Type Usage Analysis

| Error Type | Used in Moby? | containerd/errhttp support? | Notes |
|------------|---------------|----------------------------|-------|
| `IsAlreadyExists` | ✅ **YES** (10 uses) | ❌ NO | Used heavily in image operations |
| `IsFailedPrecondition` | ❌ Only in httpstatus | ✅ YES (maps to 412) | No actual usage found |
| `IsOutOfRange` | ❌ Only in httpstatus | ❌ NO | No actual usage found |
| `IsAborted` | ❌ Only in httpstatus | ❌ NO | No actual usage found |

**Important**: `IsAlreadyExists` is actively used in daemon code for image operations, so this mapping must be preserved.

### 3. Mapping Differences

| Error Type | Moby (current) | containerd/errhttp | Semantic Correctness |
|------------|----------------|-------------------|---------------------|
| `IsFailedPrecondition` | 400 Bad Request | 412 Precondition Failed | containerd is more correct |
| `IsAlreadyExists` | 409 Conflict | *not handled* | Moby is correct, containerd missing |

### 4. Moby-Specific Requirements
1. **gRPC error handling**: Must handle gRPC codes from containerd backend
2. **Distribution errors**: Must handle Docker registry error codes
3. **Multi-error unwrapping**: Complex error chains need recursive handling

## Recommendation: Hybrid Approach

### Option A: Wrapper around containerd/errhttp (Recommended)

Create a wrapper that:
1. Delegates to `errhttp.ToHTTP()` for standard error types
2. Adds Moby-specific error type handling
3. Maintains gRPC and Distribution error handling

**Pros:**
- Aligns with containerd ecosystem
- Maintains compatibility with client-side `ToNative()`
- Preserves Moby-specific functionality
- More semantic HTTP status codes (e.g., 412 for FailedPrecondition)

**Cons:**
- Breaking change: `IsFailedPrecondition` changes from 400 → 412
- Need to verify no clients depend on current mappings

**Implementation sketch:**
```go
func FromError(err error) int {
    if err == nil {
        log.G(context.TODO()).WithError(err).Error("unexpected HTTP error handling")
        return http.StatusInternalServerError
    }

    rerr := cerrdefs.Resolve(err)
    
    // Handle Moby-specific error types first
    switch {
    case cerrdefs.IsAlreadyExists(rerr):
        return http.StatusConflict
    }
    
    // Delegate to containerd for standard types
    if code := errhttp.ToHTTP(rerr); code != http.StatusInternalServerError {
        return code
    }
    
    // Fall back to gRPC and Distribution error handling
    if statusCode := statusCodeFromGRPCError(err); statusCode != http.StatusInternalServerError {
        return statusCode
    }
    if statusCode := statusCodeFromDistributionError(err); statusCode != http.StatusInternalServerError {
        return statusCode
    }
    
    // ... rest of unwrapping logic
}
```

### Option B: Keep Current Implementation (Simpler)

Keep the current implementation and just add the missing error types found in our audit.

**Pros:**
- No breaking changes
- No new dependencies
- Already works well
- Current PR is almost ready

**Cons:**
- Maintains divergence from containerd
- Client and server use different logic

## Breaking Change Assessment

### IsFailedPrecondition: 400 → 412
**Risk**: Low
- Not used anywhere in codebase except httpstatus itself
- No evidence of external clients depending on this
- More semantically correct per HTTP spec

### IsAlreadyExists: 409 (no change)
**Risk**: None
- Already mapped to 409
- containerd doesn't handle this, so no conflict

## Migration Path

If we choose Option A (wrapper approach):

1. **Phase 1**: Add wrapper around `errhttp.ToHTTP()` with Moby-specific extensions
2. **Phase 2**: Add deprecation notice for old behavior
3. **Phase 3**: Update tests to reflect new status codes
4. **Phase 4**: Document breaking changes in release notes

## Final Recommendation

Given the current state:
1. Current PR adds necessary error type mappings
2. Very limited actual usage of problematic error types
3. Risk of breaking changes with wrapper approach

**I recommend: Keep current implementation (Option B) for now**

However, add this to the code documentation:
```go
// TODO: Consider migrating to containerd/errdefs/pkg/errhttp.ToHTTP() as the base
// implementation, with Moby-specific extensions for gRPC and Distribution errors.
// This would align server-side error handling with the client-side usage of
// errhttp.ToNative(). Main blocker: IsFailedPrecondition mapping change (400→412).
```

## Response to Reviewer

Suggested response to thaJeztah's comment:

> Thank you for pointing out the containerd/errdefs utility. I've done a detailed comparison:
>
> **Key findings:**
> 1. The client already uses `errhttp.ToNative()`, so there's value in aligning server-side
> 2. However, Moby needs to handle `IsAlreadyExists` which containerd/errhttp doesn't support (used 10+ times in image operations)
> 3. The main mapping difference is `IsFailedPrecondition`: containerd uses 412, we use 400
> 
> **Options:**
> A. Wrap `errhttp.ToHTTP()` and add Moby-specific extensions (better alignment, but breaking change for FailedPrecondition)
> B. Keep current implementation (simpler, no breaking changes)
>
> Which approach would you prefer? I can implement either way.

## Files for Reference
- Detailed comparison: `daemon/server/httpstatus/containerd-comparison.md`
- This recommendation: `daemon/server/httpstatus/recommendation.md`
