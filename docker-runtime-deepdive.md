# containerd / runc 深掘り編

[docker-fundamentals.md](docker-fundamentals.md) では「dockerd → containerd → runc」を一直線の流れとして扱ったが、
実際には containerd と runc の間に **shim** という重要なプロセスが挟まっている。

## 15. containerd/runc 深掘り

### shim (containerd-shim-runc-v2)

- containerd がコンテナごとに起動する軽量な仲介プロセス
- なぜ shim が必要か
  - **containerd 自体が再起動・クラッシュしてもコンテナを生かし続けるため**
    shim がコンテナプロセスの親（実質的な監視者）になるので、containerd デーモンの再起動はコンテナ本体に影響しない
  - runc はコンテナを起動したらすぐ終了する短命プロセス（[docker-fundamentals.md](docker-fundamentals.md)参照）なので、
    起動後の状態監視・シグナル転送・exit code 回収を継続的に行う「監視役」が別途必要
  - fork/exec のオーバーヘッドを毎回 containerd 本体で負わない、という設計上の分離でもある
- shim v2 はコンテナ 1 つにつき 1 プロセス（旧 shim v1 は runtime ごとに複数プロセスだった）
- shim は ttrpc（軽量な gRPC 代替）で containerd と通信する

### containerd の gRPC API 構成

- containerd は複数のサービス（gRPC）の集合として実装されている
  - **Content**: コンテンツアドレス可能なストア（[docker-advanced.md](docker-advanced.md) の Content Store）
  - **Images**: イメージメタデータ管理
  - **Snapshots**: ファイルシステムスナップショット管理
  - **Tasks**: 実行中のコンテナ（プロセス）のライフサイクル管理。start/pause/kill などはここ経由
  - **Containers**: コンテナのメタデータ（設定）管理。Task とは別物（設定 vs 実行中プロセス）
- Docker 側の連携実装は `daemon/containerd/`（image 関連が中心。イメージの pull/push/展開など）

### CRI (Container Runtime Interface)

- Kubernetes の kubelet が containerd を直接操作するためのプラグイン（containerd に組み込み）
- Docker (dockerd) を経由せず、kubelet → containerd → shim → runc という経路になる
- 「なぜ Kubernetes は dockerd を使わなくなったか」を理解する鍵
  - dockerd は元々 Kubernetes 向けではなく、CRI 経由の方が中間層が少なく効率的
  - dockershim（kubelet 内の互換レイヤー）は Kubernetes 1.24 で削除された

### runc / libcontainer

- runc は OCI Runtime Specification の実装で、内部的に **libcontainer**（旧 Docker 発の低レベルコンテナ生成ライブラリ）を使う
- namespace の作成、cgroup への割り当て、capability の適用、seccomp/AppArmor の適用を行い、最終的に `exec` でユーザープロセスに置き換わる

### OCI Runtime Spec の `config.json`

- runc に渡される実行仕様。主な内容
  - `process`: 実行コマンド、環境変数、作業ディレクトリ、capability
  - `root`: rootfs のパス（読み取り専用かどうか）
  - `mounts`: マウントするファイルシステムのリスト
  - `linux.namespaces`: 使用する namespace の種類
  - `linux.resources`: cgroup によるリソース制限
  - `hooks`: `prestart` / `poststart` / `poststop` などのライフサイクルフック（ネットワーク設定注入などに使われる）

### rootfs の切り替え: pivot_root vs chroot

- runc はデフォルトで `pivot_root` を使ってコンテナの rootfs に切り替える
  - `chroot` と異なり、古い root を完全に見えなくできる（脱出しにくい）
  - `pivot_root` が使えない環境（一部の制約された環境）ではフォールバックとして `chroot` ベースの処理も存在する

### cgroup v1 と v2

- **cgroup v1**: リソースの種類（CPU, memory, blkio 等）ごとに別々の階層ツリーを持つ
- **cgroup v2**: 統一された単一階層ツリー。近年のディストリビューションの標準
  - `cgroup.controllers` によりコントローラを制御。旧 API より一貫性がある
- **cgroup driver** の違い（詰まりやすいポイント）
  - `cgroupfs`: Docker が直接 cgroup ファイルシステムを操作
  - `systemd`: systemd 経由で cgroup を管理（systemd が init のホストではこちらが推奨されることが多い、Kubernetes でも同様の議論がある）
  - 実装は `daemon/daemon_unix.go`, `daemon/oci_linux.go` あたりでランタイムに渡す設定を組み立てている

### 参考実装パス

- `daemon/containerd/`（Docker からの containerd 呼び出し）
- `daemon/oci_linux.go`（OCI spec 組み立て）
- `daemon/runtime_unix.go`（ランタイム設定）
