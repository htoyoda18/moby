# Comparison: Moby httpstatus vs containerd/errdefs/pkg/errhttp

## Overview
This document compares the current Moby implementation in `daemon/server/httpstatus/status.go` with the containerd/errdefs `pkg/errhttp/http.go` implementation.

## Mapping Differences

### Error Types Handled by BOTH

| Error Type | Moby Status | containerd Status | Match? |
|------------|-------------|-------------------|--------|
| `IsNotFound` | 404 Not Found | 404 Not Found | ✅ |
| `IsInvalidArgument` | 400 Bad Request | 400 Bad Request | ✅ |
| `IsConflict` | 409 Conflict | 409 Conflict | ✅ |
| `IsUnauthorized` | 401 Unauthorized | 401 Unauthorized | ✅ |
| `IsPermissionDenied` | 403 Forbidden | 403 Forbidden | ✅ |
| `IsResourceExhausted` | 429 Too Many Requests | 429 Too Many Requests | ✅ |
| `IsNotImplemented` | 501 Not Implemented | 501 Not Implemented | ✅ |
| `IsUnavailable` | 503 Service Unavailable | 503 Service Unavailable | ✅ |
| `IsNotModified` | 304 Not Modified | 304 Not Modified | ✅ |

### Error Types with DIFFERENT Mappings

| Error Type | Moby Status | containerd Status | Notes |
|------------|-------------|-------------------|-------|
| `IsFailedPrecondition` | 400 Bad Request | 412 Precondition Failed | ⚠️ **Different!** containerd uses more specific code |
| `IsAlreadyExists` | 409 Conflict | *Not handled* | Moby handles this explicitly |
| `IsOutOfRange` | 400 Bad Request | *Not handled* | Moby handles this explicitly |
| `IsAborted` | 409 Conflict | *Not handled* | Moby handles this explicitly |
| `IsInternal` | 500 Internal Server Error | 500 Internal Server Error | ✅ |
| `IsDataLoss` | 500 Internal Server Error | *Not handled* | Moby groups with IsInternal |
| `IsDeadlineExceeded` | 500 Internal Server Error | *Not handled* | Moby groups with IsInternal |
| `IsCanceled` | 500 Internal Server Error | *Not handled* | Moby groups with IsInternal |

## Additional Features

### Moby-specific Features
1. **gRPC error handling**: `statusCodeFromGRPCError()` - maps gRPC codes to HTTP status
2. **Distribution error handling**: `statusCodeFromDistributionError()` - handles Docker registry errors
3. **Recursive unwrapping**: Handles both single and multi-error unwrapping
4. **Debug logging**: Logs unexpected error types for debugging

### containerd-specific Features
1. **Bidirectional mapping**: `ToNative()` converts HTTP status codes back to errors
2. **Simpler implementation**: More focused, single responsibility
3. **Unexpected status handling**: Handles `cause.ErrUnexpectedStatus` with embedded status codes

## Key Differences Summary

### 1. FailedPrecondition Mapping
- **Moby**: Maps to `400 Bad Request`
- **containerd**: Maps to `412 Precondition Failed`
- **Impact**: containerd's mapping is more semantically correct per HTTP spec

### 2. Coverage
- **Moby**: Handles more errdefs types (`IsAlreadyExists`, `IsOutOfRange`, `IsAborted`, `IsDataLoss`, `IsDeadlineExceeded`, `IsCanceled`)
- **containerd**: More focused set of error types

### 3. Integration Points
- **Moby**: Integrated with gRPC and Docker Distribution error systems
- **containerd**: Pure errdefs-to-HTTP mapping

### 4. Error Resolution
- **Moby**: Uses `cerrdefs.Resolve(err)` at the beginning
- **containerd**: Direct error checking without explicit resolution

## Recommendation

### Option 1: Use containerd/errdefs/pkg/errhttp directly
**Pros:**
- Maintains consistency with containerd ecosystem
- Simpler, more focused implementation
- Bidirectional mapping included

**Cons:**
- Need to add wrapper for gRPC error handling
- Need to add wrapper for Distribution error handling
- Loses coverage of some error types (IsAlreadyExists, IsOutOfRange, etc.)
- `IsFailedPrecondition` mapping change might break existing behavior (400 → 412)

### Option 2: Keep current implementation with improvements
**Pros:**
- Maintains backward compatibility
- Comprehensive error type coverage
- Already integrated with gRPC and Distribution errors
- No behavior changes for existing error handling

**Cons:**
- Maintains divergence from containerd
- Less semantic correctness for `IsFailedPrecondition`

### Option 3: Hybrid approach
**Pros:**
- Use containerd's core mapping as foundation
- Add Moby-specific extensions (gRPC, Distribution)
- Maintain additional error type handling where needed

**Cons:**
- More complex to maintain
- Need to decide on conflicting mappings (like FailedPrecondition)

## Current Usage in Moby

### Client-side (already using containerd/errdefs)
- `client/errors.go`: Uses `errhttp.ToNative()` to convert HTTP status codes to errors
- `client/internal/jsonmessages.go`: Also imports `errhttp`

### Server-side (daemon)
`httpstatus.FromError()` is used in only 3 places:
1. `daemon/server/middleware/debug.go` - logging status code
2. `daemon/server/server.go` - setting HTTP status code
3. `daemon/server/router/container/container_routes.go` - setting HTTP status code

### Observation
**Client and Server are using different implementations!**
- Client: Uses `containerd/errdefs/pkg/errhttp.ToNative()` (HTTP → error)
- Server: Uses custom `httpstatus.FromError()` (error → HTTP)

## Symmetry Issue

For proper round-tripping, if client uses `errhttp.ToNative()`, the server should ideally use `errhttp.ToHTTP()` to ensure:
```
Server Error → HTTP Status (via ToHTTP) → Network → Client Error (via ToNative)
```

Currently:
```
Server Error → HTTP Status (via custom logic) → Network → Client Error (via ToNative)
```

This asymmetry could cause mismatches if mappings differ.

## Questions for Maintainer

1. Is changing `IsFailedPrecondition` from 400 to 412 acceptable?
   - containerd uses 412 (semantically correct)
   - current moby uses 400
   - **Impact**: Breaking change if clients depend on 400

2. Should we align server-side with client-side by using `errhttp.ToHTTP()`?
   - **Pro**: Symmetric error handling, consistent with containerd ecosystem
   - **Con**: Need to wrap it for gRPC/Distribution error handling
   - **Con**: Potential breaking changes in status code mappings

3. Are the extra error types (`IsAlreadyExists`, `IsOutOfRange`, `IsAborted`) used in practice?
   - These are not in containerd's errhttp
   - Can we audit actual usage to see if they can be simplified?

4. Should gRPC and Distribution error handling be separate middleware?
   - Instead of embedding in FromError(), could be cleaner separation of concerns
