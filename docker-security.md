# Docker セキュリティ編

## 14. セキュリティ

namespace と cgroup（[docker-internals.md](docker-internals.md)参照）は「隔離」と「リソース制御」を担うが、
それだけではコンテナ内のプロセスがホストに対して危険な操作（デバイス直叩き、カーネルモジュールロード等）をできてしまう。
ここで扱うのは「隔離された中で何を許可するか」を制御する仕組み。

### Linux Capabilities

- root 権限を細かい単位（約40種）に分割したもの
- 例: `CAP_NET_ADMIN`（ネットワーク設定変更）、`CAP_SYS_ADMIN`（広範なシステム管理操作）
- Docker はデフォルトで一部の capability のみ付与し、危険なものは落として起動する
- `--cap-add` / `--cap-drop` で個別に調整可能
  - 例: `docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE`

### seccomp（Secure Computing Mode）

- プロセスが呼び出せるシステムコールをホワイトリスト/ブラックリストで制限する Linux カーネル機能
- Docker はデフォルトの seccomp プロファイルを適用し、`reboot()` や `mount()` など危険な syscall をブロックする
- `--security-opt seccomp=profile.json` でカスタムプロファイルを指定可能
- 実装: `daemon/seccomp_linux.go`

### AppArmor / SELinux

- MAC (Mandatory Access Control)。ファイルパスやリソース単位でアクセス制御する仕組み
- **AppArmor**: Ubuntu/Debian 系で使われる。Docker はデフォルトプロファイル（`docker-default`）を自動生成・適用する
  - 実装: `daemon/apparmor_linux.go`, `daemon/apparmor_default.go`
- **SELinux**: RHEL/Fedora 系で使われる。ラベルベースのアクセス制御（`--security-opt label=...`）
- カーネルが対応していない場合（`_unsupported.go`）は無効化される

### User Namespace Remap（`--userns-remap`）

- コンテナ内の root (UID 0) を、ホスト上の非特権 UID にマッピングする
- コンテナ内で root 権限を奪取されても、ホスト上では一般ユーザー権限にしかならない
- User namespace（[docker-internals.md](docker-internals.md) 参照）を利用した多層防御

### rootless モード

- dockerd 自体を非 root ユーザーで起動する仕組み
- setuid/root 権限なしで network namespace や overlay マウントを扱うため、`slirp4netns`（ユーザーランドネットワーク）などの補助ツールを利用
- セットアップスクリプト: `contrib/dockerd-rootless-setuptool.sh`, `contrib/dockerd-rootless.sh`
- 実装: `daemon/internal/rootless`
- ホストに root 権限を一切要求しない代わりに、ネットワーク性能などにトレードオフがある

### `no-new-privileges`

- `--security-opt no-new-privileges` で setuid バイナリ等による権限昇格を禁止する
- コンテナ内で予期せず root 権限を得ることを防ぐ最終防衛ライン

### イメージの署名・検証

- **Docker Content Trust (DCT)**: Notary を使ったイメージの署名・検証（`DOCKER_CONTENT_TRUST=1`）
- 近年は Sigstore/cosign によるキーレス署名や、SBOM/Provenance などの Attestation（[docker-advanced.md](docker-advanced.md) 参照）と組み合わせるのが主流
