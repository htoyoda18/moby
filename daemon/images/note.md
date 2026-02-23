## 概要

daemon/images は Docker のイメージ管理を担当するパッケージ。
イメージの pull、push、削除、ビルド、タグ付けなど、イメージライフサイクル全般を管理する。

## 責務

- イメージの取得（pull）・配布（push）
- イメージの削除・prune
- イメージのメタデータ管理
- レイヤーストアとの連携
- レファレンス（タグ、ダイジェスト）管理
- ビルドキャッシュ管理
- コンテナからのイメージコミット

## 主要コンポーネント

### ImageService (service.go)

イメージ管理の中核となるサービス。

**主要フィールド:**
- `imageStore` - イメージメタデータストア
- `layerStore` - レイヤー管理（graphdriver 経由）
- `referenceStore` - イメージ参照（タグ・ダイジェスト）
- `downloadManager` - レイヤーダウンロード管理（並列ダウンロード制御）
- `uploadManager` - レイヤーアップロード管理（並列アップロード制御）
- `distributionMetadataStore` - レジストリメタデータ
- `registryService` - レジストリ接続
- `eventsService` - イベント通知
- `leases` - コンテンツリース管理（containerd）
- `content` - コンテンツストア（containerd）

**設定項目:**
- `MaxConcurrentDownloads` - 同時ダウンロード数（デフォルト: 3）
- `MaxConcurrentUploads` - 同時アップロード数（デフォルト: 5）
- `MaxDownloadAttempts` - ダウンロード再試行回数

### ストア層 (store.go)

**imageStoreWithLease**
- イメージ削除時にリースも削除するラッパー
- containerd のリース機能を使ってコンテンツのライフサイクル管理

**imageStoreForPull**
- pull 操作専用のイメージストアラッパー
- pull したコンテンツをリースに登録
- GC から保護

**contentStoreForPull**
- pull 時のコンテンツストアラッパー
- commit されたダイジェストを記録
- リース管理と連携

リースの目的:
- pull 途中でのガベージコレクション防止
- 失敗時の自動クリーンアップ
- マルチテナント環境でのリソース分離

## ファイル構成

### コアファイル

- **service.go** - ImageService の定義と初期化
- **store.go** - イメージストアのラッパー、リース管理
- **image.go** - イメージ取得の汎用ロジック
- **cache.go** - ビルドキャッシュ管理

### イメージ操作

- **image_pull.go** - docker pull の実装
- **image_push.go** - docker push の実装
- **image_delete.go** - docker rmi の実装（conflict 検出含む）
- **image_tag.go** - docker tag の実装
- **image_prune.go** - docker image prune の実装
- **image_list.go** - docker images の実装

### イメージ変換・インポート

- **image_commit.go** - docker commit（コンテナからイメージ作成）
- **image_import.go** - docker import（tarball からイメージ作成）
- **image_exporter.go** - docker save の実装
- **image_squash.go** - レイヤー圧縮（実験的機能）

### メタデータ・情報取得

- **image_inspect.go** - docker inspect の実装
- **image_history.go** - docker history の実装
- **image_changes.go** - コンテナの変更検出

### ビルド関連

- **image_builder.go** - ビルド時のレイヤー操作
- **imagespec.go** - OCI イメージ仕様との変換

### イベント

- **image_events.go** - イメージイベント通知

### プラットフォーム固有

- **image_unix.go** - Unix/Linux 固有の処理
- **image_windows.go** - Windows 固有の処理

## 主要な処理フロー

### Pull (image_pull.go)

1. リファレンス解析（タグ/ダイジェスト）
2. レジストリ接続・認証
3. マニフェスト取得
4. プラットフォームチェック（--platform 指定時）
5. リース作成（pull 用）
6. レイヤーダウンロード（並列、downloadManager 経由）
7. イメージ設定の保存
8. リファレンス登録
9. イベント発行

**並列ダウンロード:**
- `MaxConcurrentDownloads` で制御
- レイヤーごとに独立してダウンロード
- 失敗時は `MaxDownloadAttempts` まで再試行

**TODO:**
- 複数プラットフォーム同時 pull 未対応 (image_pull.go:28)

### Delete (image_delete.go)

イメージ削除には2種類の conflict がある:

**Hard Conflict（削除不可）:**
- 実行中のコンテナが使用中
- 子イメージが存在
- pull/build 中

**Soft Conflict（Force で削除可能）:**
- 停止中のコンテナが使用中
- アクティブなタグ・ダイジェスト参照

**削除フロー:**
1. イメージ取得・存在確認
2. 使用中コンテナのチェック
3. 子イメージのチェック
4. conflict 判定
5. タグ削除（リポジトリ参照の場合）
6. イメージ削除（参照がなくなった場合）
7. レイヤー削除（他イメージで使われていない場合）
8. イベント発行

### Commit (image_commit.go)

コンテナから新しいイメージを作成:

1. コンテナの RWLayer を取得
2. レイヤーをマウント
3. 変更の tar アーカイブ作成
4. 新レイヤーとして登録
5. イメージ設定作成
6. タグ付け

**TODO/注意点:**
- Windows ドライバーの Diff() 実装問題で不要なマウントが残っている (image_commit.go:93)

### Prune (image_prune.go)

未使用イメージの削除:

**削除対象:**
- タグなし（dangling）イメージ
- `--all` 指定時: コンテナで使われていない全イメージ
- `--filter` での絞り込み（until, label など）

**Prune の排他制御:**
- `pruneRunning` フラグで同時実行を防止
- 複数の prune コマンドが同時に走らないように保護

## TODO/FIXME コメント

### 機能追加

**image_pull.go:28 - 複数プラットフォーム同時 pull**
```go
// TODO(thaJeztah): add support for pulling multiple platforms
```
- 現状: 1プラットフォームのみ
- 需要: マルチアーキテクチャイメージの一括取得

**image_push.go:22 - 複数プラットフォーム同時 push**
```go
// TODO(thaJeztah): add support for pushing multiple platforms
```
- pull と同様の制限

### Windows 関連の問題

**image_commit.go:93 - 不要なマウント呼び出し**
```go
// TODO: this mount call is not necessary as we assume that TarStream() should
// mount the layer if needed. But the Diff() function for windows requests that
// the layer should be mounted when calling it. So we reserve this mount call
// until windows driver can implement Diff() interface correctly.
```
- 問題: Windows ドライバーの Diff() が正しく実装されていない
- 回避策: 手動でマウントを呼び出している
- 本来: TarStream() が必要に応じてマウントすべき

**image_windows.go:23 - RootFS の破壊的変更**
```go
// FIXME: why does this mutate the RootFS?
img.RootFS.DiffIDs = img.RootFS.DiffIDs[:index]
```
- 問題: GetLayerFolders() がイメージの RootFS を書き換えている
- 副作用: 元のイメージ構造が怖れる可能性
- 改善: コピーを作成して操作すべき

**image_windows.go:14 - サイズ取得未実装**
```go
// TODO Windows
return 0, 0, nil
```
- GetContainerLayerSize() が未実装
- 常に 0 を返す

**image_unix.go:39 - GetSize のエラー処理**
```go
// FIXME: GetSize should return an error. Not changing it now in case
// there are compatibility issues.
```
- 問題: GetSize() がエラーを返せない
- 理由: 互換性の懸念
- 改善: インターフェース変更が必要

### 設計改善

**service.go:30 - containerStore.Get() の削除**
```go
// TODO: remove, only used for CommitBuildStep
Get(string) *container.Container
```
- CommitBuildStep のみで使用
- 依存関係を減らすため削除したい

**service.go:118 - Children() の設計**
```go
// TODO: refactor to expose an ancestry for image.ID?
```
- 現状: 子イメージのリストのみ返す
- 改善案: イメージの祖先関係全体を公開

**service.go:125 - CreateLayer() の引数**
```go
// TODO: accept an opt struct instead of container?
```
- 現状: container 全体を受け取る
- 改善: オプション構造体で必要な情報のみ受け取る

**service.go:170 - GetLayerMountID() の廃止**
```go
// TODO: needs to be refactored to Unmount (see callers), or removed and
// replaced with GetLayerByID
```
- 現状: MountID を返すだけ
- 改善: Unmount メソッドに変更、または GetLayerByID に統合

**image_builder.go:113 - 空レイヤーの最適化**
```go
// TODO: An optimization would be to handle empty layers before returning
```
- 空レイヤーの事前チェックでパフォーマンス改善可能

**image_builder.go:153 - PullImage の共通化**
```go
// TODO: could this use the regular daemon PullImage ?
```
- ビルダー専用の pull ロジックがある
- 通常の PullImage と統合できる可能性

### その他

**context.TODO() の使用**
- service.go:199, image_builder.go:50, 131 など
- ログ出力時に context.TODO() を使用
- 適切な context を伝播すべき

## 設計パターン

### レイヤードアーキテクチャ

```
ImageService (API層)
    ↓
imageStore / layerStore (ストレージ層)
    ↓
graphdriver (ドライバー層)
    ↓
filesystem (OS層)
```

### マネージャーパターン

- `downloadManager` - レイヤーダウンロードの並列制御
- `uploadManager` - レイヤーアップロードの並列制御
- セマフォで同時実行数を制限

### リースパターン（containerd）

```
pull 開始
  → リース作成
  → コンテンツダウンロード
  → リースにリソース追加
  → pull 完了（リース維持）
  → GC 実行（リースで保護されたコンテンツは削除されない）
```

### イベント駆動

全操作でイベントを発行:
- `pull`, `push`, `delete`, `tag`, `untag`, `prune`
- イベントリスナー（ログ、メトリクス、通知など）で処理

## TODO/FIXME 優先度分析

### インパクト・難易度マトリクス

| 問題 | インパクト | 難易度 | 優先度 | カテゴリ |
|------|-----------|--------|--------|----------|
| 1. RootFS の破壊的変更 (Windows) | 高 | 低 | 🔴 最高 | バグ修正 |
| 2. 複数プラットフォーム pull/push | 中 | 高 | 🟡 中 | 機能追加 |
| 3. CreateLayer() の引数改善 | 中 | 中 | 🟡 中 | 設計改善 |
| 4. GetLayerMountID() の廃止 | 中 | 中 | 🟡 中 | 設計改善 |
| 5. PullImage の共通化 | 中 | 中 | 🟡 中 | 設計改善 |
| 6. containerStore.Get() の削除 | 低 | 中 | 🟢 低 | 設計改善 |
| 7. Windows Diff() 実装修正 | 中 | 高 | 🟡 中 | Windows修正 |
| 8. GetSize のエラー処理 | 低 | 中 | 🟢 低 | API改善 |
| 9. Windows サイズ取得実装 | 低 | 中 | 🟢 低 | Windows機能 |
| 10. 空レイヤー最適化 | 低 | 低 | 🟢 低 | 最適化 |
| 11. Children() の設計改善 | 低 | 中 | 🟢 低 | 設計改善 |
| 12. context.TODO() 置き換え | 中 | 高 | 🟡 中 | アーキテクチャ |

### 詳細分析

#### 🔴 最高優先度（すぐ修正すべき）

**1. image_windows.go:23 - RootFS の破壊的変更**
- **インパクト: 高**
  - データ破損の潜在的リスク
  - 同じイメージを複数回使うと壊れる可能性
  - Windows コンテナユーザーに直接影響
  - 予期しない動作・バグの原因
- **難易度: 低**
  - RootFS のコピーを作成するだけ
  - 1箇所の修正で済む
  - テストも容易
- **対策案**
  ```go
  // 修正前
  img.RootFS.DiffIDs = img.RootFS.DiffIDs[:index]

  // 修正後
  rootFS := *img.RootFS  // コピーを作成
  rootFS.DiffIDs = rootFS.DiffIDs[:index]
  // 以降は rootFS を使用
  ```
- **推奨**: 即座に修正すべき

#### 🟡 中優先度（計画的に実施）

**2. 複数プラットフォーム同時 pull/push**
- **インパクト: 中**
  - マルチアーキテクチャ環境での利便性向上
  - CI/CD パイプラインの高速化
  - buildx などとの整合性
  - ユーザーからの需要あり
- **難易度: 高**
  - pull/push ロジック全体の見直し
  - レイヤー管理の複雑化
  - 並列ダウンロードとの兼ね合い
  - エラーハンドリングが複雑
  - プラットフォームごとのマニフェスト処理
- **対策案**
  1. まず pull の複数プラットフォーム対応
  2. 次に push の対応
  3. プラットフォームごとに独立したダウンロードフロー
  4. 統合的なエラーレポート
- **推奨**: 段階的に実装（2-3リリース）

**3. service.go:125 - CreateLayer() の引数改善**
- **インパクト: 中**
  - コードの可読性向上
  - 依存関係の明確化
  - テストのしやすさ
  - 将来の拡張性
- **難易度: 中**
  - インターフェース変更（破壊的）
  - 全呼び出し元の修正
  - オプション構造体の設計
  - 後方互換性の考慮
- **対策案**
  ```go
  // 新インターフェース
  type CreateLayerOpts struct {
      ImageID    string
      ContainerID string
      MountLabel  string
      StorageOpt  map[string]string
      InitFunc    layer.MountInit
  }

  func (i *ImageService) CreateLayerV2(opts *CreateLayerOpts) (container.RWLayer, error)
  ```
- **推奨**: メジャーバージョンアップ時に実施

**4. service.go:170 - GetLayerMountID() の廃止**
- **インパクト: 中**
  - API の簡素化
  - 責務の明確化
  - 呼び出し側のコード改善
- **難易度: 中**
  - 全呼び出し元の調査
  - Unmount メソッドへの置き換え
  - または GetLayerByID への統合
  - テストの更新
- **対策案**
  1. 呼び出し元を調査（daemon.go）
  2. Unmount に置き換えられるか検証
  3. デプリケーション警告を追加
  4. 1-2バージョン後に削除
- **推奨**: 段階的廃止

**5. image_builder.go:153 - PullImage の共通化**
- **インパクト: 中**
  - コードの重複削除
  - 保守性の向上
  - バグ修正が1箇所で済む
  - 一貫性の向上
- **難易度: 中**
  - ビルダー専用ロジックの分析
  - 通常の PullImage との差異確認
  - オプションでの切り分け
  - リグレッションテスト
- **対策案**
  1. ビルダー専用の部分を特定
  2. PullImage にオプション追加
  3. ビルダーから通常の PullImage を呼び出す
- **推奨**: リファクタリング時に実施

**7. image_commit.go:93 - Windows Diff() 実装修正**
- **インパクト: 中**
  - パフォーマンス改善（不要なマウント削除）
  - コードの簡素化
  - Windows ドライバーの正しい実装
- **難易度: 高**
  - Windows ドライバー（daemon/graphdriver/windows）の修正
  - Diff() インターフェースの理解
  - TarStream() との連携
  - Windows 環境でのテスト必須
- **対策案**
  1. Windows ドライバーの Diff() を修正
  2. TarStream() が自動マウントするように変更
  3. image_commit.go から手動マウントを削除
- **推奨**: Windows 開発者が対応すべき

**12. context.TODO() の置き換え**
- **インパクト: 中**
  - タイムアウト・キャンセルの伝播
  - 分散トレーシング対応
  - モダンな Go プラクティス
- **難易度: 高**
  - ImageService インターフェース全体の変更
  - 全メソッドに context 追加
  - 呼び出し元の修正
  - graphdriver とも連携
- **対策案**
  - daemon/graphdriver の context 対応と合わせて実施
  - 段階的移行
- **推奨**: アーキテクチャ改善の一環として

#### 🟢 低優先度（できれば改善）

**6. service.go:30 - containerStore.Get() の削除**
- **インパクト: 低**
  - 依存関係の削減
  - インターフェースの簡素化
  - CommitBuildStep のみの影響
- **難易度: 中**
  - CommitBuildStep の代替実装
  - containerStore インターフェースの変更
  - 後方互換性
- **対策案**
  - CommitBuildStep で別の方法を使う
  - または Get() を別の場所に移動
- **推奨**: リファクタリング時に検討

**8. image_unix.go:39 - GetSize のエラー処理**
- **インパクト: 低**
  - エラーハンドリングの改善
  - より正確な動作
  - 互換性の問題のみ
- **難易度: 中**
  - インターフェース変更（破壊的）
  - 全呼び出し元の修正
  - デプリケーション期間
- **対策案**
  - GetSizeWithError() など新メソッド追加
  - 段階的移行
- **推奨**: メジャーバージョンアップ時

**9. image_windows.go:14 - Windows サイズ取得実装**
- **インパクト: 低**
  - Windows での情報精度向上
  - docker ps のサイズ表示
  - 実用上は大きな問題なし
- **難易度: 中**
  - Windows レイヤーサイズの計算
  - graphdriver からの情報取得
  - Windows 環境でのテスト
- **対策案**
  - layerStore.DiffSize() を使う
  - または graphdriver 経由で取得
- **推奨**: Windows ユーザーからの要望があれば

**10. image_builder.go:113 - 空レイヤー最適化**
- **インパクト: 低**
  - ビルド時の微小な高速化
  - リソース節約
  - 実用上の差は小さい
- **難易度: 低**
  - 空レイヤーチェックの追加
  - 早期リターン
  - テスト追加
- **対策案**
  ```go
  if isEmpty(layer) {
      return nil, nil  // 早期リターン
  }
  ```
- **推奨**: パフォーマンス改善の一環で

**11. service.go:118 - Children() の設計改善**
- **インパクト: 低**
  - イメージツリーの可視化
  - API の一貫性
  - 現状でも機能は足りている
- **難易度: 中**
  - 祖先関係の構造体設計
  - インターフェース追加
  - 呼び出し元の更新
- **対策案**
  ```go
  type ImageAncestry struct {
      ID       image.ID
      Parents  []image.ID
      Children []image.ID
  }

  func (i *ImageService) GetAncestry(id image.ID) (*ImageAncestry, error)
  ```
- **推奨**: 需要があれば実装

### 推奨実装順序

**Phase 1 - 緊急バグ修正（即座）:**
1. RootFS の破壊的変更修正 ← Windows コンテナの重大バグ

**Phase 2 - Quick Wins（1-2リリース）:**
1. 空レイヤー最適化 ← 低難易度・低リスク

**Phase 3 - 設計改善（メジャーバージョン）:**
1. CreateLayer() の引数改善
2. GetLayerMountID() の廃止
3. GetSize のエラー処理

**Phase 4 - 大規模改善（複数リリース）:**
1. context.TODO() 置き換え（graphdriver と連携）
2. 複数プラットフォーム pull/push
3. PullImage の共通化

**Phase 5 - Windows 専用改善（Windows チーム）:**
1. Diff() 実装修正
2. サイズ取得実装

**Phase 6 - 必要に応じて:**
1. containerStore.Get() の削除
2. Children() の設計改善

### リスク評価

**高リスク:**
- RootFS の破壊的変更 ← データ破損の可能性

**中リスク:**
- 複数プラットフォーム対応 ← 複雑な変更
- Windows Diff() 修正 ← プラットフォーム依存

**低リスク:**
- 空レイヤー最適化 ← 影響範囲が小さい
- 各種設計改善 ← 段階的移行可能

## RootFS 破壊的変更の詳細分析

### 問題の所在

[image_windows.go:19-43](daemon/images/image_windows.go#L19-L43) の `GetLayerFolders()` メソッド:

```go
func (i *ImageService) GetLayerFolders(img *image.Image, rwLayer container.RWLayer, containerID string) ([]string, error) {
    folders := []string{}
    rd := len(img.RootFS.DiffIDs)
    for index := 1; index <= rd; index++ {
        // FIXME: why does this mutate the RootFS?
        img.RootFS.DiffIDs = img.RootFS.DiffIDs[:index]  // ← 破壊的変更！
        // ...
        layerPath, err := layer.GetLayerPath(i.layerStore, img.RootFS.ChainID())
        // ...
        folders = append([]string{layerPath}, folders...)
    }
    // ...
}
```

### 何が起こっているか

**ループごとの RootFS.DiffIDs の変化:**

```
元のイメージ: DiffIDs = [A, B, C, D]

index=1: img.RootFS.DiffIDs = [A]           ← [A, B, C, D][:1]
index=2: img.RootFS.DiffIDs = [A, B]        ← [A][:2] ではなく、元の配列の [:2]
index=3: img.RootFS.DiffIDs = [A, B, C]     ← [A, B][:3] ではなく、元の配列の [:3]
index=4: img.RootFS.DiffIDs = [A, B, C, D]  ← 最終的に元に戻る

関数終了後: img.RootFS.DiffIDs = [A, B, C, D]  ← 偶然元に戻る
```

**スライスの仕組み:**
Go のスライスは内部的に配列への参照を保持しているため、`[:index]` は元の配列を参照し続けます。
そのため、各ループで元の配列に対して操作が行われています。

### 実際の影響

#### シナリオ 1: 正常終了の場合
```go
// コンテナ起動時
img := imageService.GetImage(ctx, imageID)  // 新しいオブジェクト取得
// img.RootFS.DiffIDs = [layer1, layer2, layer3, layer4]

GetLayerFolders(img, ...)
// ループ後: img.RootFS.DiffIDs = [layer1, layer2, layer3, layer4] (元に戻る)
// → 問題なし（ただし、途中の状態は壊れている）
```

#### シナリオ 2: 途中でエラーが発生した場合
```go
img := imageService.GetImage(ctx, imageID)
// img.RootFS.DiffIDs = [layer1, layer2, layer3, layer4]

GetLayerFolders(img, ...)
// index=2 で layer.GetLayerPath() がエラー
// この時点で img.RootFS.DiffIDs = [layer1, layer2]  ← 壊れたまま！

// もし img がどこかでキャッシュされていたら？
// または同じ関数内で img を再利用していたら？
// → データ破損
```

#### シナリオ 3: 並行処理の場合
```go
// goroutine 1
img1 := imageService.GetImage(ctx, imageID)
go GetLayerFolders(img1, ...)  // img1.RootFS.DiffIDs を変更中

// goroutine 2 (同じイメージ)
img2 := imageService.GetImage(ctx, imageID)  // 別のオブジェクト
go GetLayerFolders(img2, ...)  // 問題なし（別オブジェクトなので）
```

### 幸運な点

**imageStore.Get() は毎回新しいオブジェクトを作成:**
- [daemon/internal/image/store.go:210-229](daemon/internal/image/store.go#L210-L229)
- `NewFromJSON()` で新しい Image オブジェクトを生成
- ストレージ内のデータは破壊されない
- 各コンテナ起動時に新しい img を取得するので、前のコンテナの影響を受けない

```go
func (is *store) Get(id ID) (*Image, error) {
    config, err := is.fs.Get(id.Digest())  // ストレージから読み込み
    img, err := NewFromJSON(config)        // 新しいオブジェクト作成
    return img, nil                        // 毎回新規作成
}
```

### 実際のリスク

#### 低リスク（現状）:
- imageStore.Get() が毎回新規オブジェクトを作成
- GetLayerFolders() が正常終了すれば元に戻る
- 各コンテナ起動は独立している

#### 潜在的な高リスク:
1. **コードの可読性**: なぜデータを壊しているのか不明瞭
2. **保守性**: 将来的な変更でバグを埋め込みやすい
3. **将来の実装変更**:
   - imageStore.Get() がキャッシュを返すように変更されたら？
   - GetLayerFolders() の中で img を複数回参照したら？
   - 途中でエラーが発生して img を再利用したら？
4. **デバッグの困難さ**: 途中でエラーが出たときの状態が予測不能

### なぜこのコードが書かれたか

**ChainID の計算のため:**
```go
// ChainID は累積的なハッシュ
// DiffIDs = [A, B, C, D] の場合:
// ChainID(1) = Hash(A)
// ChainID(2) = Hash(Hash(A) + B)
// ChainID(3) = Hash(Hash(Hash(A) + B) + C)
// ChainID(4) = Hash(Hash(Hash(Hash(A) + B) + C) + D)
```

各レイヤーの ChainID を計算するために、DiffIDs を切り詰めて `img.RootFS.ChainID()` を呼んでいます。

### 正しい修正方法

**方法1: RootFS のコピーを作成（推奨）**
```go
func (i *ImageService) GetLayerFolders(img *image.Image, rwLayer container.RWLayer, containerID string) ([]string, error) {
    folders := []string{}
    rd := len(img.RootFS.DiffIDs)

    // RootFS のコピーを作成
    rootFS := image.RootFS{
        Type:    img.RootFS.Type,
        DiffIDs: make([]layer.DiffID, rd),
    }
    copy(rootFS.DiffIDs, img.RootFS.DiffIDs)

    for index := 1; index <= rd; index++ {
        // コピーを変更（元のデータは無傷）
        rootFS.DiffIDs = rootFS.DiffIDs[:index]

        if err := image.CheckOS(img.OperatingSystem()); err != nil {
            return nil, errors.Wrapf(err, "cannot get layerpath for ImageID %s", rootFS.ChainID())
        }
        layerPath, err := layer.GetLayerPath(i.layerStore, rootFS.ChainID())
        if err != nil {
            return nil, errors.Wrapf(err, "failed to get layer path from graphdriver %s for ImageID %s", i.layerStore, rootFS.ChainID())
        }
        folders = append([]string{layerPath}, folders...)
    }
    // ...
}
```

**方法2: 一時的な RootFS を毎回作成**
```go
for index := 1; index <= rd; index++ {
    // 毎回新しい RootFS を作成
    tempRootFS := image.RootFS{
        Type:    img.RootFS.Type,
        DiffIDs: img.RootFS.DiffIDs[:index],  // 元は変更しない
    }

    layerPath, err := layer.GetLayerPath(i.layerStore, tempRootFS.ChainID())
    // ...
}
```

**方法3: ChainID を直接計算**
```go
// RootFS を変更せずに累積 ChainID を計算
var chainID layer.ChainID
for index := 1; index <= rd; index++ {
    diffID := img.RootFS.DiffIDs[index-1]
    if chainID == "" {
        chainID = layer.CreateChainID([]layer.DiffID{diffID})
    } else {
        chainID = layer.CreateChainID(append(chainID.DiffIDs(), diffID))
    }

    layerPath, err := layer.GetLayerPath(i.layerStore, chainID)
    // ...
}
```

### 影響範囲

**呼び出し元:**
- [daemon/oci_windows.go:143](daemon/oci_windows.go#L143) - Windows コンテナ起動時のみ
- Windows Server Core / Nano Server コンテナが対象
- Linux コンテナには影響なし

**頻度:**
- コンテナ起動ごとに1回実行
- 高頻度で実行される可能性あり

### 修正の緊急度

**即座に修正すべき理由:**
1. データ破損のリスク（低確率だが高影響）
2. コードの意図が不明瞭（FIXME コメント付き）
3. 修正が簡単（数行の変更）
4. テストも容易
5. 将来の実装変更で深刻なバグになる可能性

**修正の難易度: 低**
- コード変更: 10行程度
- テスト: Windows コンテナ起動テスト
- リスク: ほぼゼロ（動作は同じ）
- 所要時間: 1-2時間