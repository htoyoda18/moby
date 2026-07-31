# Docker Compose 編

## 16. Docker Compose

### 位置づけ

- Docker Compose はこの `moby` リポジトリには含まれない別プロジェクト（`docker/compose`, 仕様は `compose-spec/compose-spec`）
- 複数コンテナで構成されるアプリケーションを **1 つの YAML ファイル**で定義し、まとめて起動・停止・管理するツール
- 内部的には Compose が Docker Engine の REST API（[docker-fundamentals.md](docker-fundamentals.md) 参照）を呼び出しているだけで、
  dockerd から見れば通常の `docker run` / `docker network create` 等の API 呼び出しの集合に過ぎない

### compose.yaml の基本構造

```yaml
services:
  web:
    build: .
    ports:
      - "8080:80"
    depends_on:
      - db
  db:
    image: postgres:16
    volumes:
      - dbdata:/var/lib/postgresql/data

volumes:
  dbdata:
```

- `services`: 起動するコンテナ（サービス）の定義
- `volumes` / `networks`: トップレベルで定義し、複数サービスから共有
- Compose はデフォルトで **プロジェクト単位の専用ネットワーク**を自動作成し、サービス名で名前解決できるようにする（[docker-networking.md](docker-networking.md) の埋め込み DNS 参照）

### `docker compose up` の裏側

```text
docker compose up
↓ compose.yaml をパース
↓ 依存関係順 (depends_on) に従って
docker network create (プロジェクト用ネットワーク)
docker volume create (名前付きボリューム)
docker build / docker pull (イメージ準備)
docker run 相当の API 呼び出し (コンテナ起動、上記ネットワークに接続)
```

- Compose V2 は Go 製 CLI プラグイン（`docker compose`）として実装されており、旧 Python 版（`docker-compose`）とは別実装
- スケールしても Swarm や Kubernetes のような分散オーケストレーションは行わない（単一ホスト向け）

### Swarm との違い

- Compose: 単一ホスト内でのマルチコンテナ定義・起動が主目的
- Swarm ([docker-swarm-deepdive.md](docker-swarm-deepdive.md) 参照): 複数ホストにまたがるオーケストレーション
- `docker stack deploy` は compose.yaml 相当のファイルを Swarm 向けに読み替えて使うブリッジ的存在
