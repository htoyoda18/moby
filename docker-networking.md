# Docker ネットワーキング編

## 11. ネットワーキング

### CNM (Container Network Model)

- Docker のネットワークを抽象化するモデル
- 3 つの要素で構成される
  - **Sandbox**: コンテナのネットワーク環境（namespace）そのもの
  - **Endpoint**: Sandbox をネットワークに接続する仮想インターフェース（veth の片割れ）
  - **Network**: Endpoint の集合。同じ Network に属する Endpoint 同士は通信できる
- 実装は `daemon/libnetwork/`
  - `daemon/libnetwork/drivers/` にドライバごとの実装が並ぶ（bridge, host, overlay, macvlan, ipvlan, null, remote, windows）
  - Kubernetes の CNI (Container Network Interface) とは別物。CNM は Docker 独自のモデル

### ネットワークドライバの種類

#### bridge（デフォルト）

- ホスト内に仮想ブリッジ（`docker0`）を作成し、コンテナはそこに接続する
- コンテナごとに veth pair が作られる
  - 片方はコンテナの netns 内（`eth0` として見える）
  - もう片方はホスト側で `docker0` に接続される
- 単一ホスト内の通信に向く。デフォルトブリッジと user-defined bridge で挙動が違う
  - user-defined bridge はコンテナ名での DNS 解決が有効
  - デフォルトブリッジは `--link` を使わないと名前解決できない（レガシー）

#### host

- コンテナがホストの network namespace をそのまま共有する
- 隔離がない代わりにオーバーヘッドがない
- ポートマッピング（`-p`）は無意味になる（ホストのポートをそのまま使うため）

#### overlay

- 複数ホストにまたがるコンテナ同士を 1 つの仮想ネットワークとして扱う
- Swarm モードで使用
- VXLAN でホスト間のトンネリングを行う
- コントロールプレーンは gossip プロトコル（Serf ベース）で構成情報を配布

#### macvlan / ipvlan

- コンテナに物理 NIC 上の仮想 MAC/IP を直接割り当てる
- コンテナが物理ネットワーク上に「独立したホスト」のように見える
- macvlan: コンテナごとに別 MAC アドレスを持つ
- ipvlan: MAC アドレスは共有し、IP のみ分離（スイッチの MAC アドレステーブル上限対策）

#### none

- ネットワークインターフェースを一切持たない（lo のみ）

### ポート公開の実体（`-p` オプション）

```text
docker run -p 8080:80 nginx
```

- ホストの 8080 番ポート宛の通信をコンテナの 80 番へ転送する
- 実体は iptables (nftables) の DNAT ルール
  - `docker-proxy` プロセスまたは iptables の NAT テーブルで実現
  - `DOCKERCHAIN`（`DOCKER` チェーン）に DNAT ルールが追加される
- コンテナからホスト外へ出る通信は MASQUERADE (SNAT) される

### 埋め込み DNS サーバー

- Docker デーモンには組み込みの DNS サーバーがある（`127.0.0.11`）
- user-defined network 内では、コンテナ名やエイリアスで名前解決できる
- 外部ドメインの解決はホストの DNS 設定 or `--dns` オプションへフォワードされる

### 参考実装パス

- `daemon/libnetwork/drivers/bridge/`
- `daemon/libnetwork/drivers/overlay/`
- `daemon/network/` （dockerd 側の API 層）
