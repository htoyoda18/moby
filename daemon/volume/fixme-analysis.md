# daemon/volume の FIXME/TODO 分析

## 重大な問題（実際のバグ）

### 1. **マウントカウントがゼロの状態でデクリメントされるバグ** ⚠️ 🔴
**Issue**: https://github.com/moby/moby/issues/46508

**場所**: 
- `daemon/volume/mounts/mounts.go:110-115`
- `daemon/volume/local/local.go:369-377`

**問題**: 
特定の競合状態において、マウントカウントがゼロに達した後もデクリメントされることがある。これは本来発生してはならず、mount/unmount操作における深刻な並行性バグを示している。

**現在の回避策**:
```go
// TODO: Remove once the real bug is fixed: https://github.com/moby/moby/issues/46508
if m.active == 0 {
    logger.Error("An attempt to decrement a zero mount count")
    logger.Error(string(debug.Stack()))
    return nil  // 負の値になることを防ぐ
}
```

**影響**: 
- ガードがない場合、ボリュームが削除不可能になる（マウントカウントが負になる）
- 根本原因は未修正 - おそらく並行mount/unmountにおける競合状態
- スタックトレースのログはデバッグに役立つが、根本的な問題は解決していない

**推奨事項**: マウント参照カウントロジックの徹底的な調査が必要

---

## パフォーマンスとアーキテクチャの TODO

### 2. **ByReferenced フィルターの最適化** 📊
**場所**: `daemon/volume/service/store.go:267-269`

```go
// TODO(@cpuguy83): It would be nice to optimize this by looking at the list
// of referenced volumes, however the locking strategy makes this difficult
// without either providing inconsistent data or deadlocks.
```

**問題**: 
参照状態でボリュームをフィルタリングする際、別途 `refs` マップを保持しているにもかかわらず、すべてのボリュームをリストアップしてからフィルタリングする必要がある。

**現在の動作**:
```go
refs map[string]map[string]struct{}  // ボリューム名 -> 参照のセット
```
ロックの複雑さのため、このマップを直接使用できない。

**影響**: 参照済み/未参照のボリュームのみが必要な場合でも O(n) のパフォーマンス

**潜在的な解決策**: refs マップへの安全なアクセスを可能にするための読み取り/書き込みロックのリファクタリング

---

### 3. **V2 プラグインのパフォーマンス** 🐢
**場所**: `daemon/volume/service/store.go:517-519`

```go
// TODO(cpuguy83): With v2 plugins this shouldn't be a problem. Could also potentially
// use a connect timeout for this kind of check to ensure we aren't blocking for a
// long time.
```

**背景**: V1プラグインが遅いため、名前の競合チェックですべてのドライバーを調査していない。

**問題**: 
- V1プラグインがダウンしている場合、ストア全体がブロックされる
- プラグイン通信にタイムアウトがない
- デッドロックを引き起こす可能性がある

**影響**: 異なるボリュームドライバー間の名前の衝突を見逃す可能性がある

**推奨事項**: 接続タイムアウトまたは非同期プロービングの実装

---

### 4. **Context の伝播が欠落** 🔌
**場所**: 
- `daemon/volume/service/store.go:399`
- `daemon/volume/service/store.go:761`

```go
// TODO(@cpuguy83): plumb context through
```

**問題**: 複数のボリュームストア操作がcontextを受け取らないため、以下が不可能:
- リクエストのキャンセル
- タイムアウトの伝播
- トレーシング/可観測性

**影響**: 長時間実行されるボリューム操作をキャンセルできない

---

## 軽微な問題と疑問

### 5. **EvalSymlinks パスに関する疑問** 🤔
**場所**: `daemon/volume/local/local.go:233-241`

```go
// TODO(thaJeztah) is there a reason we're evaluating the data-path here, and not the volume's rootPath?
realPath, err := filepath.EvalSymlinks(lv.path)
```

**疑問**: ボリュームルートではなく `_data` パスでシンボリックリンクを評価するのはなぜか？

**現在の動作**: _data が存在しない場合は rootPath にフォールバックする（コミット 8d27417 参照）

**影響**: 不明 - 調査が必要

---

### 6. **Windows マウントモードの検証** 🪟
**場所**: `daemon/volume/mounts/windows_parser.go:140`

```go
// TODO should windows mounts produce an error if any mode was provided (they're a no-op on windows)
```

**問題**: マウントモード（rw/ro）がWindowsでは現在エラーを出さずに無視される

**影響**: ユーザーの混乱 - モードを指定しても効果がない

---

### 7. **網羅的な Switch 文** 🔀
**場所**: 
- `daemon/volume/mounts/linux_parser.go:363`
- `daemon/volume/mounts/windows_parser.go:403`

```go
// TODO(thaJeztah): make switch exhaustive: anything to do for mount.TypeNamedPipe, mount.TypeCluster ?
```

**問題**: マウントタイプに対するswitch文がすべての可能なタイプを処理していない

**処理されていないタイプ**:
- `mount.TypeNamedPipe` (Windows)
- `mount.TypeCluster` 
- `mount.TypeTmpfs` (Windows)
- `mount.TypeImage` (Windows)

**影響**: 黙ってフォールスルー - 意図的なものか実装漏れかが不明

---

## テスト関連の FIXME

### 8. **空のモード文字列の動作** ❓
**場所**: 複数のテストファイル

```go
Mode: "", // FIXME(thaJeztah): why is this different than an explicit "rw" ?
```

**ファイル**:
- `daemon/volume/mounts/linux_parser_test.go:171,191`
- `daemon/volume/mounts/lcow_parser_test.go:198`
- `daemon/volume/mounts/windows_parser_test.go:215`

**疑問**: 空のモード文字列 `""` は明示的な `"rw"` とは異なる扱いにすべきか？

**現在の動作**: テストでは両者が異なることを示しているが、理由は不明

---

### 9. **Windows CI の失敗** 🧪
**場所**: `daemon/volume/local/local_test.go:51`

```go
skip.If(t, runtime.GOOS == "windows", "FIXME: investigate why this test fails on CI")
```

**問題**: テストはローカルでは成功するが、Windows CI では失敗する

**影響**: Windowsでのテストカバレッジが低下

---

## 統計サマリー

| カテゴリ | 件数 |
|----------|-------|
| 重大なバグ | 1 |
| パフォーマンス問題 | 2 |
| アーキテクチャ改善 | 2 |
| 不明確な動作 | 3 |
| テストの問題 | 2 |
| **合計** | **10** |

## 優先度別推奨事項

1. **HIGH（高）**: issue #46508（ゼロマウントカウントバグ）の修正 - データ損失の可能性
2. **MEDIUM（中）**: キャンセルサポートのためのcontext伝播の追加
3. **MEDIUM（中）**: プラグイン接続タイムアウトの実装
4. **LOW（低）**: 適切なエラーハンドリングを伴う網羅的なswitch文の作成
5. **LOW（低）**: 空のモード文字列の動作を調査
