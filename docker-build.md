# Docker ビルドシステム編

## 13. ビルドシステム

### `docker build` の裏側

```text
docker build -t myapp .
↓ REST API (/build)
dockerd
↓ ビルドコンテキスト送信 (tar化された ./)
BuildKit (buildkitd, dockerd に統合)
↓ Dockerfile を解析 → 実行グラフ (LLB) を構築
containerd (snapshotter 経由でレイヤーを生成)
```

- 2018 年以降、標準のビルドエンジンは **BuildKit** に置き換わっている（`DOCKER_BUILDKIT=1`、Docker 20.10+ ではデフォルト）
- 従来のビルドエンジン（レガシービルダー）は `daemon/builder/dockerfile/` の evaluator/dispatcher が Dockerfile 命令を1行ずつ逐次実行する方式

### BuildKit のアーキテクチャ

- **Frontend**: Dockerfile などの入力を解析し、実行グラフ（LLB）に変換する
  - Dockerfile 用フロントエンドはコンテナイメージとして配布される（`docker/dockerfile:1` など）
- **LLB (Low-Level Build definition)**: ビルド手順を表す DAG（有向非巡回グラフ）
  - 「何を実行するか」ではなく「何に依存するか」を表現 → 並列実行・キャッシュ判定がしやすい
- **Solver**: LLB を解決し、実際にキャッシュを参照しながらステップを実行するスケジューラ
- **Exporter**: ビルド結果をイメージ・OCI tar・ローカルファイルなど様々な形式で出力する

### レイヤーキャッシュの無効化ルール

- Dockerfile の各命令（`RUN`, `COPY` 等）は 1 レイヤーに対応
- キャッシュが有効な条件
  - 命令の文字列が完全一致
  - `COPY`/`ADD` の場合は対象ファイルの内容（チェックサム）も一致
- 1 つの命令でキャッシュが切れると、それ以降の命令もすべて再実行される
  → 変更頻度が低い命令（依存関係インストール等）を Dockerfile の上に書くのが定石

### マルチステージビルド

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN go build -o /app

FROM alpine
COPY --from=builder /app /app
ENTRYPOINT ["/app"]
```

- 複数の `FROM` でステージを分け、`COPY --from=` で必要な成果物だけ次のステージに持ち越す
- ビルドツールチェーンを最終イメージに含めずに済み、イメージサイズを削減できる

### `.dockerignore`

- ビルドコンテキスト（tar 化して送られるディレクトリ）から除外するファイルを指定
- `.git` や `node_modules` を除外することでコンテキスト送信を高速化し、キャッシュの無駄な破棄も防ぐ

### buildx とマルチプラットフォームビルド

- `docker buildx`: BuildKit を直接操作する CLI プラグイン。複数ビルダーインスタンス管理や `--platform` 指定に対応
- 異なる CPU アーキテクチャ（例: amd64 ホストで arm64 イメージ）をビルドする場合
  - QEMU によるユーザーランドエミュレーション + `binfmt_misc`（カーネルの機能）でエミュレータを自動起動
  - 結果は Manifest List（[docker-advanced.md](docker-advanced.md) の Manifest 参照）としてまとめられる
- `docker buildx bake`: 複数イメージ・複数プラットフォームのビルドを HCL/JSON 定義でまとめて実行（このリポジトリの `docker-bake.hcl` も実例）

### 参考実装パス

- `daemon/builder/dockerfile/`（レガシービルダーの evaluator/dispatcher）
- `daemon/builder/remotecontext/`（ビルドコンテキストの取得・展開、git 経由のビルドなど）
- `daemon/builder/backend/`（ビルド API のバックエンド）
