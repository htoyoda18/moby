# daemon/server TODO/FIXME 優先度分析

## 評価基準

### インパクト
- **High**: セキュリティ、データ損失、重大なバグ、API破損
- **Medium**: パフォーマンス、ユーザー体験、API互換性
- **Low**: コード品質、保守性、ログ改善

### 難易度
- **Hard**: 破壊的変更、アーキテクチャ変更、複数モジュールに影響
- **Medium**: 中規模のリファクタリング、API変更（後方互換性維持）
- **Easy**: 局所的な変更、単純な修正

---

## 優先度順位（High → Low）

### 🔴 **P0: 即座に対応すべき（High Impact × Easy~Medium）**

#### 1. [FIXME] container_routes.go:1107-1110 - fmt.Fprintf/Fprintのエラー無視
- **場所**: [router/container/container_routes.go:1107-1110](router/container/container_routes.go#L1107)
- **インパクト**: **High** - コンテナアタッチ時のHTTPアップグレード失敗を検知できない
- **難易度**: **Easy** - エラーハンドリング追加のみ
- **詳細**: WebSocket接続のアップグレード時にHTTPレスポンス書き込みエラーを無視している。接続確立失敗を見逃す可能性
- **修正方針**: `fmt.Fprintf`のエラーを確認し、失敗時にログ出力と適切なクリーンアップ

```go
// 現在
fmt.Fprintf(conn, "HTTP/1.1 101 UPGRADED\r\n...")

// 修正後
if _, err := fmt.Fprintf(conn, "HTTP/1.1 101 UPGRADED\r\n..."); err != nil {
    log.G(ctx).WithError(err).Error("failed to write upgrade response")
    return err
}
```

#### 2. [FIXME] httpstatus/status.go:69 - 予期しないエラー型の処理
- **場所**: [httpstatus/status.go:65-69](httpstatus/status.go#L65)
- **インパクト**: **High** - 新しいエラー型が適切にマッピングされず500エラーになる
- **難易度**: **Medium** - エラー型の網羅性確認と対応が必要
- **詳細**: 既知のエラー型にマッチしない場合、デバッグログを出力して500を返す。適切なステータスコードが返せない
- **修正方針**: エラー型を網羅的に調査し、適切なマッピングを追加。sentinel errorパターンの見直し

#### 3. [FIXME] distribution_routes.go:55 - repositoriesのオンデマンド構築
- **場所**: [router/distribution/distribution_routes.go:55-65](router/distribution/distribution_routes.go#L55)
- **インパクト**: **Medium** - 不要なレジストリ接続による遅延とリソース浪費
- **難易度**: **Medium** - distribution.Pull()のロジックを参考に実装変更
- **詳細**: GetRepositories()が全エンドポイントに接続を試みるが、最初の1つだけで済む場合が多い
- **修正方針**: pullEndpointsユーティリティを使った遅延評価パターンに変更

---

### 🟠 **P1: 次期バージョンで対応すべき（Medium~High Impact × Medium~Hard）**

#### 4. [FIXME] system_routes.go:181 - API 1.53でレガシーフィールド削除
- **場所**: [router/system/system_routes.go:181](router/system/system_routes.go#L181)
- **インパクト**: **Medium** - API肥大化、保守コスト増加
- **難易度**: **Medium** - バージョンゲートの実装と非推奨化アナウンス
- **詳細**: 1.52以前のクライアント互換性のために残されているレガシーフィールド
- **修正方針**: 1.53で完全削除、リリースノートで明記

#### 5. copy.go:95 - noOverwriteDirNonDirのデフォルト反転
- **場所**: [router/container/copy.go:95](router/container/copy.go#L95)
- **インパクト**: **Medium** - セキュリティとデータ整合性への影響
- **難易度**: **Hard** - 破壊的変更、移行パス設計が必要
- **詳細**: 現在の挙動（ディレクトリをファイルで上書き可能）は直感的でなく危険
- **修正方針**: 新パラメータ`allowOverwriteDirWithFile`を導入し段階的移行

#### 6. backend.go:164 - container.Config → DockerOCIImageConfig
- **場所**: [backend/backend.go:164](backend/backend.go#L164)
- **インパクト**: **Medium** - 型の不整合、OCI標準への非準拠
- **難易度**: **Hard** - 型変更が広範囲に影響
- **詳細**: Docker固有の型を使用しており、OCI標準との互換性に問題
- **修正方針**: 段階的に`dockerspec.DockerOCIImageConfig`へ移行

#### 7. network_routes.go:137 - ネットワーク選択ロジックの移動
- **場所**: [router/network/network_routes.go:137](router/network/network_routes.go#L137)
- **インパクト**: **Medium** - レイヤー違反、テストしづらさ
- **難易度**: **Medium** - バックエンドに新関数追加とリファクタ
- **詳細**: ルーティング層でビジネスロジック（ネットワーク選択）を実装している
- **修正方針**: `backend.GetNetwork(term, scope)`のような関数をバックエンドに追加

---

### 🟡 **P2: 計画的に対応（Low~Medium Impact × Easy~Medium）**

#### 8. debug.go:35 - Server.makeHTTPHandlerとのログ統合
- **場所**: [middleware/debug.go:35](middleware/debug.go#L35)
- **インパクト**: **Low** - ログの重複と不整合
- **難易度**: **Medium** - 2箇所のエラーログロジック統合
- **修正方針**: 共通のエラーログ関数を作成し両方から呼び出す

#### 9. debug.go:67 - JSON形式ログ検出の改善
- **場所**: [middleware/debug.go:67](middleware/debug.go#L67)
- **インパクト**: **Low** - ログ出力の最適化のみ
- **難易度**: **Easy** - ログフォーマッタの型確認
- **修正方針**: `log.GetFormatter()`のような方法で明示的に検出

#### 10. exec.go:70 - stream config重複指定のリファクタ
- **場所**: [router/container/exec.go:70](router/container/exec.go#L70)
- **インパクト**: **Low** - コード重複、保守性
- **難易度**: **Medium** - API変更とリファクタリング
- **詳細**: create時とstart時に同じstream設定を指定する必要がある
- **修正方針**: create時の設定を保持し、start時に再利用

#### 11. container_routes.go:529,534 - CreateRequest直接利用
- **場所**: [router/container/container_routes.go:529-534](router/container/container_routes.go#L529)
- **インパクト**: **Low** - 型変換のオーバーヘッド
- **難易度**: **Medium** - 関数シグネチャ変更
- **修正方針**: バックエンド関数が直接`container.CreateRequest`を受け取る

#### 12. build.go:50 - デフォルト値をdaemon/configへ移動
- **場所**: [router/build/build.go:50](router/build/build.go#L50)
- **インパクト**: **Low** - 設定の一元管理
- **難易度**: **Easy** - 定数の移動
- **修正方針**: daemon/config に定数を定義し参照

---

### 🟢 **P3: 余裕があれば対応（Low Impact × Various）**

#### 13-25. その他のリファクタリング・改善
- network_routes.go:127,295 - バージョンラップ検討
- swarmbackend/swarm.go:33 - パラメータ移動検討
- exec.go:141 - context渡し検討
- grpc_routes.go:42 - conn書き込み済み問題
- plugin_routes.go:179,197 - ログ出力、プログレスバー
- distribution_routes.go:30,98 - パーサーエラー、manifest lookup
- container_routes.go:1179 - 通知クローズ
- image_routes.go:82,104,181 - ユーティリティ追加、エラー改善
- volume_routes_test.go:609,701 - テストのエラー型改善
- build/backend.go:13 - 戻り値型変更

**これらは**: コード品質や保守性の向上が主目的。機能への直接的な影響は小さい

---

## 推奨アクションプラン

### フェーズ1（即座）
1. **container_routes.go:1107** - エラーハンドリング追加（1-2時間）
2. **httpstatus/status.go:69** - エラー型マッピング改善（4-8時間）

### フェーズ2（次期マイナーバージョン）
3. **distribution_routes.go:55** - オンデマンド構築（1-2日）
4. **network_routes.go:137** - ロジック移動（1-2日）

### フェーズ3（次期メジャーバージョン v2.0等）
5. **system_routes.go:181** - レガシーフィールド削除
6. **copy.go:95** - デフォルト反転（破壊的変更）
7. **backend.go:164** - OCI標準型への移行

### フェーズ4（継続的改善）
- その他のリファクタリングをチケット化し、適宜対応

---

## リスク評価

| 項目 | 現状のリスク | 対応遅延のリスク |
|------|-------------|-----------------|
| #1 エラー無視 | 接続失敗の見逃し | 本番障害のデバッグ困難化 |
| #2 エラー型 | 不適切なHTTPステータス | APIクライアントの誤動作 |
| #3 リポジトリ | パフォーマンス劣化 | ユーザー体験低下 |
| #5 上書き挙動 | データ損失の可能性 | セキュリティインシデント |
