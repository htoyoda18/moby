# daemon/pkg TODO/FIXME Analysis

**作成日**: 2026-04-20  
**対象**: daemon/pkg パッケージ内の全TODO/FIXMEコメント

## 概要

daemon/pkgパッケージ内には約20個のTODO/FIXMEコメントが存在する。これらを問題の大きさと解決の難易度で評価し、優先度付けを行った。

---

## 🔴 優先度: High（すぐに対応すべき）

### 1. device cgroup正規表現の検証
**ファイル**: `daemon/pkg/oci/oci.go:11`

```go
// TODO verify if this regex is correct for "a" (all);
```

- **問題の大きさ**: ★★★★★ (セキュリティ/安定性)
  - device cgroupルールの誤解析はコンテナ隔離の脆弱性につながる可能性
  - カーネルドキュメントとの不一致があれば重大なバグ
- **解決の難易度**: ★☆☆☆☆ (簡単)
  - Linuxカーネルソースとドキュメントで正規表現を確認
  - テストケース追加のみで対応可能
- **推奨アクション**: ドキュメント確認とテストケース追加で即座に対応

**詳細**:
- Linuxカーネル仕様（v5.10以降）で"a"（all）の挙動を確認
- `echo a > /sys/fs/cgroup/1/devices.allow` が "a *:* rwm" と等価か検証
- ソースコード: https://github.com/torvalds/linux/blob/v5.10/security/device_cgroup.c#L614-L642

---

### 2. レジストリ設定のメモ化
**ファイル**: `daemon/pkg/registry/service_v2.go:26`

```go
// TODO(thaJeztah); this should all be memoized when loading the config. We're resolving mirrors and loading TLS config every time.
```

- **問題の大きさ**: ★★★★☆ (パフォーマンス)
  - 毎回の名前解決とTLS設定読み込みは無駄なオーバーヘッド
  - イメージpull/push時に頻繁に呼ばれる可能性あり
- **解決の難易度**: ★★★☆☆ (中程度)
  - キャッシュ実装と適切な無効化タイミングの設計が必要
  - 設定変更時のキャッシュクリアロジック
- **推奨アクション**: パフォーマンス改善効果が大きく、優先的に対応

**実装案**:
- 設定読み込み時にミラーとTLS設定を事前解決
- `sync.RWMutex`でキャッシュを保護
- 設定リロード時にキャッシュを無効化

---

### 3. 非推奨API (SetPClient) の使用
**ファイル**: `daemon/pkg/plugin/manager_linux.go:87`

```go
p.SetPClient(client) //nolint:staticcheck // FIXME(thaJeztah): p.SetPClient is deprecated: Hardcoded plugin client is deprecated
```

- **問題の大きさ**: ★★★★☆ (技術的負債)
  - 既に非推奨マークされており、将来削除されるとブレイク
  - プラグインシステムのコア部分
- **解決の難易度**: ★★★☆☆ (中程度)
  - `Addr()`を使った新しいクライアント生成への移行
  - 既存のプラグインとの互換性確認
- **推奨アクション**: 次のメジャーバージョンで削除される前に対応

**関連箇所**:
- `daemon/pkg/plugin/store.go:247` - 同様の非推奨使用あり
- `daemon/pkg/plugin/manager_linux_test.go:240` - テストでも使用

---

## 🟡 優先度: Medium（計画的に対応）

### 4. ランタイム名の検証強化
**ファイル**: `daemon/pkg/opts/runtime.go:37,44`

```go
// TODO(thaJeztah): this should not accept spaces.
// TODO(thaJeztah): this should not be case-insensitive.
```

- **問題の大きさ**: ★★★☆☆ (安定性/セキュリティ)
  - スペースやケース非依存が予期しない動作を引き起こす可能性
- **解決の難易度**: ★★★☆☆ (中程度)
  - 後方互換性への影響調査が必要
  - 既存ユーザーの設定が壊れる可能性
- **推奨アクション**: 次のメジャーバージョンで破壊的変更として実施

**移行計画**:
1. 現在のバージョン: 警告ログを出力
2. 次のマイナーバージョン: Deprecation警告
3. 次のメジャーバージョン: エラーとして拒否

---

### 5. エラーハンドリング改善
**ファイル**: `daemon/pkg/registry/search.go:193-194`

```go
// TODO(thaJeztah): return upstream response body for errors (see https://github.com/moby/moby/issues/27286).
// TODO(thaJeztah): handle other status-codes to return correct error-type
```

- **問題の大きさ**: ★★★☆☆ (UX/デバッグ性)
  - 詳細なエラー情報がないとトラブルシューティングが困難
  - レジストリエラーの詳細がユーザーに伝わらない
- **解決の難易度**: ★★☆☆☆ (低〜中)
  - レスポンスボディの保持と適切なエラー型への変換
- **推奨アクション**: ユーザー体験向上のため中優先度で対応

**参考**: https://github.com/moby/moby/issues/27286

---

### 6. プラグイン再接続の検討
**ファイル**: `daemon/pkg/plugin/manager_linux.go:141`

```go
// TODO(@cpuguy83): Should we always just re-attach to the running plugin instead of doing this?
```

- **問題の大きさ**: ★★★☆☆ (リソース管理)
  - 既存プラグインプロセスへの再接続vs再起動の判断
  - 不必要なプラグイン再起動によるダウンタイム
- **解決の難易度**: ★★★★☆ (高)
  - プラグインライフサイクルの設計変更が必要
  - 複雑な状態管理とエッジケース
- **推奨アクション**: 十分な調査と設計レビュー後に対応

**検討事項**:
- プラグインプロセスの状態確認方法
- 再接続失敗時のフォールバック
- LiveRestore時の挙動

---

### 7. BuildKit/containerdとのPATH統一
**ファイル**: `daemon/pkg/oci/defaults.go:25-26`

```go
// TODO(thaJeztah) align Windows default with BuildKit; see https://github.com/moby/buildkit/pull/1747
// TODO(thaJeztah) use defaults from containerd (but align it with BuildKit; see https://github.com/moby/buildkit/pull/1747)
```

- **問題の大きさ**: ★★★☆☆ (一貫性/UX)
  - 異なるツール間でのデフォルト動作の不一致
  - ユーザーの混乱を招く
- **解決の難易度**: ★★★☆☆ (中程度)
  - 外部プロジェクトとの調整が必要
  - Windows互換性の確認
- **推奨アクション**: BuildKitチームと連携して統一

**参考**: https://github.com/moby/buildkit/pull/1747

---

### 8. レジストリオプションの正規化
**ファイル**: `daemon/pkg/registry/config.go:305-306`

```go
// TODO(thaJeztah): consider normalizing other known options, such as "(https://)registry-1.docker.io", "https://index.docker.io/v1/".
// TODO: upstream this to check to reference package
```

- **問題の大きさ**: ★★★☆☆ (一貫性)
  - 同じレジストリへの異なる表記方法の統一
  - 設定の重複や混乱
- **解決の難易度**: ★★★☆☆ (中程度)
  - 正規化ロジックとテストケースが必要
  - 既知のレジストリエイリアスのリスト化
- **推奨アクション**: ユーザー体験向上のため対応

**対象**:
- `registry-1.docker.io` ↔ `index.docker.io`
- `https://`プレフィックスの有無

---

### 9. アドレスプールのケース依存化
**ファイル**: `daemon/pkg/opts/address_pools.go:35`

```go
// TODO(thaJeztah): this should not be case-insensitive.
```

- **問題の大きさ**: ★★☆☆☆ (一貫性)
  - ケース非依存は予期しない動作を引き起こす可能性
- **解決の難易度**: ★★★☆☆ (中程度)
  - 後方互換性への影響
  - 既存設定への影響調査
- **推奨アクション**: 破壊的変更として次期バージョンで対応

---

## 🟢 優先度: Low（余裕があれば対応）

### 10. レイヤープルの並列化検討
**ファイル**: `daemon/pkg/plugin/fetch_linux.go:117`

```go
// TODO(@cpuguy83) This gets run sequentially after layer pull (makes sense), however
```

- **問題の大きさ**: ★★☆☆☆ (パフォーマンス最適化)
  - 現在の逐次処理でも問題ない可能性
- **解決の難易度**: ★★★★☆ (高)
  - 並列化の設計は複雑、慎重な実装が必要
- **推奨アクション**: パフォーマンスボトルネックが確認されてから対応

---

### 11. コード重複のリファクタリング
**ファイル**: `daemon/pkg/plugin/v2/plugin.go:129`

```go
// TODO(vieux): lots of code duplication here, needs to be refactored.
```

- **問題の大きさ**: ★★☆☆☆ (保守性)
  - 重複コードは保守性を下げるが、直接的な機能障害ではない
- **解決の難易度**: ★★★☆☆ (中程度)
  - リファクタリングと回帰テスト
- **推奨アクション**: コードクリーンアップとして対応

---

### 12. go-unitsからの関数移動
**ファイル**: `daemon/pkg/opts/ulimit.go:26`

```go
// FIXME(thaJeztah): these functions also need to be moved over from go-units.
```

- **問題の大きさ**: ★☆☆☆☆ (依存性管理)
  - 外部依存を減らす意図だが緊急性は低い
- **解決の難易度**: ★★☆☆☆ (低〜中)
  - 単純な関数移動とテスト
- **推奨アクション**: 依存性整理時に対応

---

### 13-20. その他の低優先度項目

| ファイル | 内容 | 分類 |
|---------|------|------|
| `plugin/manager.go:66` | LiveRestoreEnabledの削除 | コードクリーンアップ |
| `plugin/v2/plugin.go:22,25` | 構造体の埋め込み/private化 | API設計改善（breaking change） |
| `plugin/backend_linux.go:216,264` | reference packageの置き換え | コード簡潔化 |
| `registry/search_endpoint_v1.go:35` | 削除検討 | V1 APIは既にレガシー |
| `registry/search.go:113` | ヘッダー追加検討 | 機能追加の可能性 |
| `opts/ulimit.go:17` | map with pointersの理由 | 設計の見直し |
| `opts/hosts_windows.go:4` | 古いGo/Windowsバグ | 詳細不明、調査困難 |
| `registry/service.go:60` | ctx使用検討 | リファクタリング |

---

## 📊 推奨アクションプラン

### 第1フェーズ (即座に対応)
1. ✅ **device cgroup正規表現の検証とテスト追加**
   - 工数: 1-2日
   - 担当: セキュリティレビュー必須
   
2. ✅ **レジストリ設定のメモ化実装**
   - 工数: 3-5日
   - パフォーマンステスト必要

### 第2フェーズ (次のマイナーバージョン)
3. ✅ **非推奨API (SetPClient) からの移行**
   - 工数: 5-7日
   - プラグイン互換性テスト必須
   
4. ✅ **エラーハンドリングの改善**
   - 工数: 2-3日
   - レジストリエラーケースの網羅的テスト

### 第3フェーズ (次のメジャーバージョン)
5. ✅ **ランタイム名検証の強化（破壊的変更）**
   - 工数: 3-4日
   - マイグレーションガイド作成
   
6. ✅ **アドレスプール/レジストリオプションの正規化**
   - 工数: 5-7日
   - 既存設定の移行パス提供

### 継続的改善
- コードクリーンアップ（低優先度項目）
- 技術的負債の解消
- ドキュメント整備

---

## メトリクス

- **総TODO/FIXME数**: 約50件（コメントのみ）
- **High優先度**: 3件
- **Medium優先度**: 6件
- **Low優先度**: 11件以上
- **推定総工数**: 25-40人日

---

## 参考リンク

- [Moby Issue #27286](https://github.com/moby/moby/issues/27286) - Registry error handling
- [BuildKit PR #1747](https://github.com/moby/buildkit/pull/1747) - Windows PATH defaults
- [Linux Device Cgroup Docs](https://github.com/torvalds/linux/blob/v5.10/Documentation/admin-guide/cgroup-v1/devices.rst)

---

**最終更新**: 2026-04-20
