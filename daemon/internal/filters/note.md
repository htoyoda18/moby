# daemon/internal/filters

Docker APIのリスト/検索系エンドポイントで使うフィルタリング機能を提供するパッケージ。

## 概要

キーから複数の値へのマッピング（`map[string]map[string]bool`）を管理する `Args` 構造体を中心に構成。`docker ps --filter status=running --filter label=app=web` のようなCLIフィルタを内部表現として扱う。

## 主要な型・関数

### Args構造体
```
fields: map[string]map[string]bool
```
- キー（例: "status", "label", "name"）に対して、複数の値をセットとして保持
- JSON シリアライズ/デシリアライズ対応（API経由でフィルタを受け渡し）

### 生成・変換
- `NewArgs(initialArgs ...KeyValuePair)` - 新規作成
- `Arg(key, value string)` - KeyValuePairの生成ヘルパー
- `ToJSON(a Args)` / `FromJSON(p string)` - JSON変換
- `FromJSON` はレガシーの `map[string][]string` 形式もフォールバックでパース

### マッチング（フィルタ適用時に使う）
- `ExactMatch(key, source)` - 完全一致
- `Match(field, source)` - 完全一致 + 正規表現マッチ
- `FuzzyMatch(key, source)` - 完全一致 + プレフィックスマッチ
- `MatchKVList(key, sources)` - `key=value` 形式のペアマッチ（ラベルフィルタ用）
- `UniqueExactMatch(key, source)` - 値が1つのときだけ完全一致
- `GetBoolOrDefault(key, defaultValue)` - "true"/"false"/"1"/"0" のブール値フィルタ

### その他
- `Validate(accepted map[string]bool)` - 許可されたキー以外がないか検証
- `WalkValues(field, op)` - 値のイテレーション
- `Clone()` - ディープコピー

## エラー型

`invalidFilter` - 不正なフィルタキーや値を示すエラー。`InvalidParameter()` メソッドを持ち、APIで400エラーとして返せる。

## 利用箇所

daemon全体で広く使われている（51ファイル）:
- コンテナ一覧 (`daemon/list.go`)
- イメージ一覧/prune (`daemon/containerd/image_list.go`, `daemon/images/image_prune.go`)
- ネットワーク (`daemon/network/filter.go`)
- ボリューム (`daemon/volume/service/`)
- イベント (`daemon/events/`)
- Swarmクラスタ (`daemon/cluster/`)
- APIルーター各種 (`daemon/server/router/`)
