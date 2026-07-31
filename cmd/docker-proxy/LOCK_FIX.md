# UDP Proxy ロック順序の修正

## 問題の概要

`udp_proxy_linux.go` の `replyLoop` メソッドにおいて、ロックの取得順序と解放順序が不一貫であり、潜在的なデッドロックとレースコンディションのリスクがありました。

## 問題の詳細

### コードの設計意図（コメントより）

[udp_proxy_linux.go:58](udp_proxy_linux.go#L58) のコメント:
> "Never lock mu without locking UDPProxy.connTrackLock first."

**ロック規約**: 必ず `proxy.connTrackLock` → `cte.mu` の順でロックを取得する

### 問題のあったコード

```go
func (proxy *UDPProxy) replyLoop(...) {
    defer func() {
        proxy.connTrackLock.Lock()      // 1. ロック
        delete(proxy.connTrackTable, *clientKey)
        cte.mu.Lock()                   // 2. ロック
        proxy.connTrackLock.Unlock()    // 3. 先にアンロック ← 問題！
        cte.conn.Close()
    }()  // 4. defer終了時に暗黙的に cte.mu.Unlock()
    ...
}
```

**問題点**:
1. **LIFO違反**: ロック解放順序が LIFO (Last In First Out) になっていない
   - 取得順序: `connTrackLock` → `cte.mu`
   - 解放順序: `connTrackLock` → `cte.mu` (暗黙的)
   - 正しくは: `cte.mu` → `connTrackLock` の順で解放すべき

2. **暗黙的アンロック**: defer終了時の暗黙的アンロックに依存しており、意図が不明瞭

3. **Close() とのレースコンディション**: `Close()` メソッド ([udp_proxy_linux.go:242-252](udp_proxy_linux.go#L242-L252)) が `cte.mu` をロックせずに `cte.conn.Close()` を呼ぶため、`Run()` メソッドと競合する可能性がある

## 修正内容

### 修正後のコード

```go
func (proxy *UDPProxy) replyLoop(...) {
    defer func() {
        proxy.connTrackLock.Lock()
        delete(proxy.connTrackTable, *clientKey)
        cte.mu.Lock()
        cte.conn.Close()
        cte.mu.Unlock()                 // LIFO順序で明示的にアンロック
        proxy.connTrackLock.Unlock()
    }()
    ...
}
```

**改善点**:
1. ✅ LIFO順序の遵守: ロック取得順序と逆順で解放
2. ✅ 明示的なアンロック: 暗黙的な動作に依存しない
3. ✅ `Close()` 実行の保護: `cte.mu` でロックされている間に `Close()` を実行

## 影響範囲

- 修正箇所: [udp_proxy_linux.go:100-108](udp_proxy_linux.go#L100-L108)
- テスト結果: すべてのテストがパス（通常実行・レース検出器ともに）
- 互換性: 既存の動作に影響なし

## 残存する潜在的な問題

`Close()` メソッド ([udp_proxy_linux.go:246-250](udp_proxy_linux.go#L246-L250)) のコメント:
```go
// Unlike the GC logic in replyLoop, we want to close the connections
// immediately, even if there are pending and in-progress writes. So no
// need to lock cte.mu here.
cte.conn.Close()
```

このコメントは誤解を招く可能性があります。`Run()` メソッドが別のgoroutineで `cte.mu` をロックして `Write()` を実行している最中に、`Close()` が `cte.conn.Close()` を呼ぶとレースコンディションが発生する可能性があります。ただし、これは既存の設計上の意図的な動作である可能性もあります。

## 検証方法

```bash
# 通常のテスト実行
go test -v ./cmd/docker-proxy

# レース検出器を使用
go test -race ./cmd/docker-proxy
```

## 関連ファイル

- [udp_proxy_linux.go](udp_proxy_linux.go) - 修正対象ファイル
- [udp_proxy_linux_test.go](udp_proxy_linux_test.go) - 既存のテスト
- [network_proxy_linux_test.go](network_proxy_linux_test.go) - 統合テスト
