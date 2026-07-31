# Docker ストレージとボリューム編

## 12. ストレージとボリューム

イメージレイヤー（[docker-internals.md](docker-internals.md)参照）は読み取り専用で、コンテナを消せば書き込みレイヤーも消える。
**データを永続化したい／ホストと共有したい場合**に使うのがここで扱う仕組み。

### 3 種類のマウント

#### Volume（推奨）

```bash
docker volume create mydata
docker run -v mydata:/var/lib/mysql mysql
```

- Docker が管理する領域（デフォルトは `/var/lib/docker/volumes/`）
- コンテナのライフサイクルと独立して存在する
- バックアップ・移行がしやすく、複数コンテナで共有しやすい
- 実装: `daemon/volume/local/`（デフォルトの local driver）

#### Bind mount

```bash
docker run -v /host/path:/container/path nginx
```

- ホスト上の任意のパスをそのままコンテナにマウントする
- Docker の管理外（存在確認・権限管理はホスト任せ）
- 開発時にソースコードを直接マウントするのに向く

#### tmpfs mount

```bash
docker run --tmpfs /app/cache nginx
```

- メモリ上にのみ存在するマウント（コンテナ停止で消える）
- 機密データを一時的に扱う場合や、ディスク I/O を避けたい場合に使う
- Linux のみ（tmpfs は Linux カーネルの機能）

### Volume driver

- Volume の実体（作成・マウント・削除）を抽象化するプラグイン機構
- デフォルトは `local` ドライバ（ホストのローカルディスクを使用）
- サードパーティドライバでリモートストレージ（NFS, EBS, Ceph 等）にも対応可能
- 実装: `daemon/volume/drivers/`（プラグインへの委譲）、`pkg/plugins` と連携（[docker-plugins.md](docker-plugins.md)参照）

### マウント解析

- Dockerfile 内の `-v` / `--mount` の文字列パースは `daemon/volume/mounts/` が担当
- Linux と Windows でパスの扱いが異なるため `linux_parser.go` / `windows_parser.go` に分かれている

### `-v` と `--mount` の違い

- `-v`（`--volume`）: 短縮記法。存在しないホストパスは自動作成される
- `--mount`: 明示的なキー・バリュー記法（`type=volume,source=...,target=...`）。誤り防止のため公式は `--mount` を推奨
