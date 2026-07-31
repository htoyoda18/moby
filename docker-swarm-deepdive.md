# Swarm 深掘り編

[docker-advanced.md](docker-advanced.md) では Swarm のイベント種別（Node/Service/Network/Secret/Config）だけを扱った。
ここではその裏にある合意形成の仕組みと、コンテナがどうスケジューリングされるかを深掘りする。

## 18. Swarm 深掘り

### manager と worker

- Swarm クラスタは **manager ノード**と **worker ノード**で構成される
  - manager: クラスタの状態管理・スケジューリング・API 提供を行う
  - worker: manager の指示を受けてコンテナ（Task）を実行するだけ
  - manager 自身も worker を兼ねられる（デフォルト）
- 実装: `daemon/cluster/`（`noderunner.go`, `nodes.go` がノード管理まわり）

### Raft コンセンサス

- manager 間の状態（サービス定義、ノード一覧など）は **Raft** アルゴリズムで複製・合意される
- なぜ Raft か
  - manager が奇数台（推奨: 3, 5, 7台）構成になるのはこのため（過半数決による分割耐性）
  - 1 台がリーダーとなり、書き込みは必ずリーダー経由。リーダーが落ちると再選挙が走る
  - manager が偶数台や過半数を割ると、クラスタ全体が書き込み不能になる（split-brain 対策）
- worker はこの Raft 合意には参加しない（manager のみの内部合意）

### Service と Task

- **Service**: 「このイメージを N 個のレプリカで動かす」という宣言的な定義（`docker service create`）
- **Task**: Service を実際に実行する単位。1 Task = 1 コンテナに相当
- manager がクラスタ全体の状態を見て、Task をどの worker に配置するか決定する（スケジューリング）
  - 実装: `daemon/cluster/executor/`（各ノード上での実行）, `daemon/cluster/controllers/`（スケジューリング制御）
- Service には 2 種類ある
  - **Replicated**: 指定した数のレプリカを維持する（デフォルト）
  - **Global**: クラスタの全ノードに 1 つずつ配置する（監視エージェント等に向く）

### Overlay ネットワークとの関係

- Swarm の Service 間通信は overlay ネットワーク（[docker-networking.md](docker-networking.md) 参照）を使う
- **Routing Mesh**: Service の公開ポートに、クラスタ内のどのノードからアクセスしても内部ロードバランシングで正しい Task まで到達する仕組み（IPVS ベース）

### Secret / Config

- **Secret**: パスワードや証明書など機密情報を暗号化して Raft ログに保存し、対象コンテナにのみ tmpfs でマウントする
- **Config**: 機密でない設定ファイルの配布に使う、Secret と同様の仕組み
- 実装: `daemon/cluster/secrets.go`

### Swarm vs Kubernetes

- Swarm は Docker 単体で完結する軽量オーケストレーターだが、エコシステム・エクステンションポイントの豊富さで Kubernetes に大きく水をあけられた（[docker-advanced.md](docker-advanced.md) で触れた通り現状はあまり使われていない）
- Kubernetes は CRI 経由で containerd を直接操作する（[docker-runtime-deepdive.md](docker-runtime-deepdive.md) の CRI 参照）ため、Swarm とはアーキテクチャ上の設計思想も異なる
