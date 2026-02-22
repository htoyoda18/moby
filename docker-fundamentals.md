# Docker 基礎編

## 1. Docker の本質

### Docker とは何か

- Docker は「OS を持たない」

  - Docker は独自の OS カーネルを持たない
  - Linux カーネル機能の集合体

  ```text
    [ App ]
    [ Container ]
    [ Linux Kernel ]  ← これを全コンテナで共有
    [ Hardware ]
  ```

- macOS/Windows の場合
  - macOS / Windows は Linux カーネルを持たない
  - 軽量 VM の中で Linux を動かしている
  - Docker Desktop は Linux を裏で起動

### コンテナの正体

- コンテナは「ただのプロセス」
  - コンテナ = 特殊な設定で起動されたプロセス
  - Linux カーネルの namespace と cgroup で隔離・制御されている

## 2. アーキテクチャ全体像

### moby (Docker Engine)

- dockerd

  - 「巨大なオーケストレーター」であって、実体は持たない
  - moby は OCI Runtime Spec のクライアント (gRPC client)
  - dockerd 自体は「実行エンジン」ではない
  - 実体はすべて外部コンポーネントに委譲
    - containerd
    - runc (namespace / cgroup のロジックは runc 側)
    - networking
    - snapshotter

- Docker Engine の役割
  - Docker 全体を動かす実行基盤の総称
  - 主に dockerd + containerd で構成される
  - API 処理、状態管理、実行委譲を担当
  - CLI や外部ツールから操作される対象

## 3. 実行フロー

### docker run の裏側

```text
docker run nginx
↓
docker CLI
↓ REST API
dockerd
↓ gRPC
containerd
↓ exec
runc
↓
Linux kernel
```

### 各コンポーネントの役割

#### docker CLI

- 人間向けインターフェース
- docker run / build / ps などを提供
- 内部では REST API を呼び出すだけ

#### REST API

- Docker の安定した契約
- Docker Engine が公開する操作インターフェース
- CLI や SDK、外部ツールが利用

#### dockerd

- Docker Engine の中心となるデーモン
- 全体オーケストレーター
- REST API を提供し状態を管理
- containerd に実行を委譲
- 「判断と管理」が役割で実行はしない

#### containerd

- コンテナのライフサイクルを管理するデーモン
  - コンテナ作成、起動、停止、削除を担当
- コンテナの実行管理者
- イメージ管理や snapshot 管理も行う
- Docker や Kubernetes から利用される
- Snapshot
  - 実行時のファイルシステム状態
  - containerd が管理する概念
  - レイヤーを実体として展開したもの
  - 起動・停止時に作成／破棄される

#### runc

- OCI Runtime Specification の実装
- Linux カーネルと直接対話するツール
- namespace / cgroup 設定を行い exec する
- コンテナ起動後は即終了する短命プロセス
- 「コンテナを実際に起動する最後の役者」

#### Linux kernel

- すべての実体
- プロセス管理、メモリ管理、ファイルシステム、ネットワークを担当
