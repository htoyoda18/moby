# httpstatus エラーハンドリング分析

**FIXME**: [httpstatus/status.go:64-69](../status.go#L64-L69)  
**GitHub Discussion**: https://github.com/moby/moby/pull/48359#discussion_r1725562802

## 問題の概要

`FromError()` 関数は、既知の errdefs 型にマッチしないエラーに遭遇した場合、デバッグログを出力して 500 Internal Server Error を返します。この FIXME コメントは、適切な HTTP ステータスコードが返せないケースが存在することを示しています。

## 現在のエラー型マッピング

### カバーされているerrdefs型

| errdefs 型 | HTTP ステータス |
|-----------|---------------|
| IsNotFound | 404 Not Found |
| IsInvalidArgument | 400 Bad Request |
| IsConflict | 409 Conflict |
| IsUnauthorized | 401 Unauthorized |
| IsUnavailable | 503 Service Unavailable |
| IsPermissionDenied | 403 Forbidden |
| IsNotModified | 304 Not Modified |
| IsNotImplemented | 501 Not Implemented |
| IsInternal | 500 Internal Server Error |
| IsDataLoss | 500 Internal Server Error |
| IsDeadlineExceeded | 500 Internal Server Error |
| IsCanceled | 500 Internal Server Error |

### カバーされていないerrdefs型

| errdefs 型 | 使用箇所数 | 推奨HTTP ステータス |
|-----------|---------|-----------------|
| IsAlreadyExists | **20箇所** | 409 Conflict |
| IsResourceExhausted | 0箇所 | 429 Too Many Requests |
| IsFailedPrecondition | 0箇所 | 400 Bad Request |
| IsOutOfRange | 0箇所 | 400 Bad Request |
| IsAborted | 0箇所 | 409 Conflict |

**注**: `IsAlreadyExists` は 20箇所で使用されているが、HTTP ステータスにマッピングされていない。

## 実際の問題事例

### Issue #48205 - OCI runtime pause エラー
- **エラー型**: `*errors.errorString`
- **状況**: コンテナの pause 操作が失敗
- **メッセージ**: "OCI runtime pause failed: unable to freeze: unknown"
- **問題**: 標準エラー型が errdefs でラップされていないため、FIXME ログが発生
- **リンク**: https://github.com/moby/moby/issues/48205

### Issue #35888 - ネットワークアタッチメント timeout
- **エラー型**: `context.DeadlineExceeded`（未ラップ）
- **状況**: API 1.24 でのコンテナネットワーク接続タイムアウト
- **問題**: タイムアウトエラーが適切に errdefs でラップされていなかった
- **リンク**: https://github.com/moby/moby/issues/35888

### PR #49367 - ネットワークエラーハンドリング修正
- **修正内容**: ネットワーク関連エラーを適切な errdefs 型でラップ
  - "No such container" → `errdefs.System()` → 500
  - "cannot join own network" → `errdefs.System()` → 500
  - Exited containers → `errdefs.Conflict()` → 409
- **効果**: FIXME ログの減少、適切な HTTP ステータスコード返却
- **リンク**: https://github.com/moby/moby/pull/49367

## 最近の改善動向

### Commit 99410827c7 (2025-10-25)
"daemon: use errdefs instead of string-matching in some places"

**修正パターン**:
```go
// Before
return errors.Errorf("container not found: %s", id)

// After
return errdefs.NotFound(fmt.Errorf("container not found: %s", id))
```

**修正箇所**:
- `daemon/containerd/image_commit.go`
- `daemon/images/image_commit.go`
- `daemon/pause.go`

## 根本原因

1. **プレーンなエラーの返却**
   - `fmt.Errorf()`, `errors.New()` が errdefs でラップされずに返される
   - 特に OCI runtime、containerd からのエラー

2. **カスタムエラー型の不完全な実装**
   - 一部のカスタムエラー型は errdefs インターフェースを実装している
   - 例: `invalidParam`, `invalidRequestError`, `notImplementedError`
   - しかし、すべてのカスタム型が実装しているわけではない

3. **サードパーティライブラリのエラー**
   - containerd, OCI runtime からのエラーが直接返される
   - これらは moby の errdefs 型システムと統合されていない

## 修正アプローチの提案

### アプローチ1: 不足しているerrdefs型のマッピングを追加

**優先度**: High（`IsAlreadyExists` は使用頻度が高い）

```go
case cerrdefs.IsAlreadyExists(rerr):
    return http.StatusConflict
case cerrdefs.IsResourceExhausted(rerr):
    return http.StatusTooManyRequests  // 429
case cerrdefs.IsFailedPrecondition(rerr):
    return http.StatusBadRequest
case cerrdefs.IsOutOfRange(rerr):
    return http.StatusBadRequest
case cerrdefs.IsAborted(rerr):
    return http.StatusConflict
```

**メリット**:
- 即座に多くのケースをカバー
- 既存コードを変更する必要がない

**デメリット**:
- プレーンなエラー型の問題は解決しない

### アプローチ2: エラー生成元での errdefs ラッピング

**優先度**: Medium（長期的な改善）

エラーが発生する箇所で適切に errdefs でラップする（PR #49367 のアプローチ）

```go
// daemon, router レイヤーでのエラー生成時
return errdefs.NotFound(fmt.Errorf("container %s not found", id))
return errdefs.System(fmt.Errorf("OCI runtime error: %w", err))
```

**メリット**:
- 根本的な解決
- エラーの意味がコード上明確になる

**デメリット**:
- 変更箇所が多い
- 既存のエラーハンドリングの見直しが必要

### アプローチ3: フォールバックロジックの改善

**優先度**: Low

FIXME ログをより詳細にし、どのエラーが多く発生しているかを追跡可能にする。

```go
log.G(context.TODO()).WithFields(log.Fields{
    "module":       "api",
    "error":        err,
    "error_type":   fmt.Sprintf("%T", err),
    "error_string": err.Error(),
    "stack_trace":  fmt.Sprintf("%+v", err), // pkg/errors のスタックトレース
}).Warn("Unhandled API error type - returning 500")
```

**メリット**:
- 問題の追跡が容易
- 将来の改善のためのデータ収集

**デメリット**:
- 根本的な解決ではない

## 推奨する実装順序

1. **Phase 1**: `IsAlreadyExists` のマッピング追加（影響が大きい）
2. **Phase 2**: FIXME ログの改善（データ収集）
3. **Phase 3**: 頻出エラーパターンの errdefs ラッピング（継続的な改善）

## 実装結果 (Phase 1)

### 実装日: 2026-05-02
### ブランチ: toyo/improve-httpstatus-error-handling

すべての未マッピング errdefs 型の HTTP ステータスマッピングを追加しました:

```go
case cerrdefs.IsAlreadyExists(rerr):
    return http.StatusConflict                // 409
case cerrdefs.IsFailedPrecondition(rerr):
    return http.StatusBadRequest              // 400
case cerrdefs.IsOutOfRange(rerr):
    return http.StatusBadRequest              // 400
case cerrdefs.IsAborted(rerr):
    return http.StatusConflict                // 409
case cerrdefs.IsResourceExhausted(rerr):
    return http.StatusTooManyRequests         // 429
```

### 期待される効果

1. **IsAlreadyExists (20箇所)**: 500 → 409 Conflict に改善
2. **gRPC エラーとの完全な整合性**: すべての gRPC エラーコードが errdefs 経由でも同じ HTTP ステータスを返す
3. **FIXME ログの削減**: より多くのエラーが適切に分類される

### 次のステップ

- Phase 2: FIXME ログの改善（どのエラーが未処理かの追跡）
- Phase 3: 頻出する未ラップエラーの特定と修正

## 参考資料

- [Issue #35888](https://github.com/moby/moby/issues/35888) - FIXME: Got an API for which error does not match any expected type!!! attaching container to network
- [Issue #38877](https://github.com/moby/moby/issues/38877) - FIXME: Got an API for which error does not match any expected type!!!
- [Issue #39865](https://github.com/moby/moby/issues/39865) - FIXME: Got an API for which error does not match any expected type
- [Issue #48205](https://github.com/moby/moby/issues/48205) - Cannot pause container - OCI runtime pause failed
- [PR #49367](https://github.com/moby/moby/pull/49367) - daemon: Daemon.getNetworkedContainer: fix errors for invalid network container
- [PR #48359](https://github.com/moby/moby/pull/48359) - FIXME discussion
