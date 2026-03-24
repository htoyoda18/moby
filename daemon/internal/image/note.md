# daemon/internal/image

Dockerイメージの内部表現・永続化・ビルドキャッシュ・tar形式でのimport/exportを担うパッケージ群。

## 全体構成

```
image/
├── image.go        -- Image型の定義（中核）
├── rootfs.go       -- RootFS型（レイヤーのdiffID一覧）
├── store.go        -- イメージストア（メモリ + ファイルシステム）
├── fs.go           -- ストアのFSバックエンド
├── image_os.go     -- OS互換性チェック
├── cache/          -- ビルドキャッシュ
│   ├── cache.go    -- キャッシュロジック
│   └── compare.go  -- Config/Platform比較
├── tarexport/      -- docker save/load
│   ├── tarexport.go -- Exporter初期化
│   ├── save.go     -- docker save (イメージ→tar)
│   ├── load.go     -- docker load (tar→イメージ)
│   └── os_path.go  -- mkdirAllWithChtimes ヘルパー
└── v1/
    └── imagev1.go  -- V1互換ID生成
```

---

## image.go - Image型

### ID
`type ID digest.Digest` - イメージ設定JSONのcontent-addressable hash（sha256）。

### V1Image（埋め込み）
Docker V1形式の設定情報:
- Parent, Comment, Created, Container, ContainerConfig
- DockerVersion, Author, Config（コンテナ設定）
- Architecture, Variant, OS

### Image（本体）
V1Imageを埋め込み、さらに以下を持つ:
- `RootFS` - レイヤー構成
- `History` - ビルドヒストリー（OCI History型のエイリアス）
- `OSVersion`, `OSFeatures` - Windows向け
- `rawJSON` - 不変のJSON（キャッシュ）
- `computedID` - コンテンツハッシュから計算されたID
- `Details` - ManifestDescriptor等の追加情報

### 主要関数
- `NewFromJSON(src []byte)` - JSONからImage生成、`rawJSON`をキャッシュ
- `NewChildImage(img, child, os)` - 親イメージから子イメージを作成（ビルド時に使用）
- `Clone(base, id)` - イメージの複製

### Exporter インターフェース
```go
type Exporter interface {
    Load(ctx, io.ReadCloser, io.Writer, bool) error
    Save(ctx, []string, io.Writer) error
}
```
`docker save` / `docker load` の抽象化。

---

## rootfs.go - RootFS

```go
type RootFS struct {
    Type    string         // 常に "layers"
    DiffIDs []layer.DiffID // 各レイヤーのdiffID（順序あり）
}
```
- `ChainID()` - DiffIDの列からChainIDを計算（レイヤーの一意識別に使用）
- `Append(id)` / `Clone()` - レイヤー追加・複製

---

## store.go - イメージストア

### Store インターフェース
```go
type Store interface {
    Create(config []byte) (ID, error)
    Get(id ID) (*Image, error)
    Delete(id ID) ([]layer.Metadata, error)
    Search(partialID string) (ID, error)
    SetParent/GetParent       // 親子関係
    SetLastUpdated/GetLastUpdated  // 更新日時
    SetBuiltLocally/IsBuiltLocally // ローカルビルドフラグ
    Children(id ID) []ID
    Map() / Heads() / Len()
}
```

### 内部構造 (store)
- `images: map[ID]*imageMeta` - メモリ上のイメージメタデータ
- `fs: StoreBackend` - ファイルシステムバックエンド
- `lss: LayerGetReleaser` - レイヤーストアへの参照
- `digestSet` - 部分ID検索用のdigestセット

### 起動時の復元 (restore)
1. FSバックエンドをWalkして全イメージを読み込み
2. 各イメージのRootFS.ChainIDでレイヤーを取得・保持
3. 2パス目で親子関係を復元

### Create時のバリデーション
- History内の非emptyレイヤー数がRootFS.DiffIDs数を超えないことを確認
- レイヤーが存在することを確認

---

## fs.go - StoreBackend（ファイルシステム永続化）

### ディレクトリ構造
```
<root>/
├── content/sha256/<encoded-digest>     -- イメージ設定JSON
└── metadata/sha256/<encoded-digest>/
    ├── parent                          -- 親イメージID
    ├── lastUpdated                     -- 更新日時
    └── builtLocally                    -- ローカルビルドフラグ
```

- `Set(data)` - データのsha256を計算し、content-addressableに保存（atomicwriter使用）
- `Get(dgst)` - 読み取り後にdigest検証
- メタデータはdigestごとのディレクトリにkey=ファイル名で保存

---

## cache/ - ビルドキャッシュ

Dockerfileのビルド時に、既存イメージをキャッシュとして再利用する仕組み。

### LocalImageCache
親イメージのchildrenから、同じConfig（コンテナ設定）を持つ最新の子イメージを探す。
- `IsBuiltLocally` がtrueのイメージのみ対象
- プラットフォームの一致も確認

### ImageCache
`--cache-from` 指定時に使用。外部イメージ（レジストリからpullしたもの等）をキャッシュソースとして使う。
- Historyの一致で親子関係を推定
- 必要に応じて中間イメージを復元（`restoreCachedImage`）

### compare.go
- `compare(a, b *container.Config)` - 2つのConfig比較。Hostname, Domainname, MacAddress, Imageは除外
- `comparePlatform` - Windowsの場合、OSVersionのMajor.Minorのみ比較

---

## tarexport/ - docker save / docker load

### Save（docker save）
1. 指定された名前/タグからイメージIDを解決
2. 各イメージのレイヤーをtarストリームとして書き出し
3. V1互換のlayer configも生成（後方互換性）
4. manifest.json + OCI layout（index.json, oci-layout）を生成
5. 全体をtarアーカイブにまとめて出力

出力形式はOCI Image Layout準拠:
- `blobs/sha256/<digest>` にレイヤーtar・設定JSON・マニフェストを配置
- `index.json` にマニフェストディスクリプタ一覧
- `manifest.json` はDocker独自のレガシー形式（後方互換）
- タイムスタンプを全てUnix epoch(0)に統一（再現性のため）

### Load（docker load）
1. tarを一時ディレクトリに展開
2. manifest.jsonを読み取り
3. 各マニフェストエントリについて:
   - 設定JSONを読み込み→OS/プラットフォームチェック
   - レイヤーを順に登録（既存なら再利用）
   - イメージをストアに作成
   - タグがあればrefstoreに登録
4. 親子関係を設定

---

## v1/imagev1.go - V1互換ID

`CreateID(v1Image, layerID, parent)` - V1形式のイメージID生成。V1Image設定 + layer_id + parent からsha256ハッシュを計算。`docker save` でレガシーレイヤーconfigを生成する際に使用される。
