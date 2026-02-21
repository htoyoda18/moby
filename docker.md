# Docker アーキテクチャと動作原理

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

## 4. イメージとファイルシステム

### Image (イメージ)

- Dockerfile からビルドされる
- コンテナ実行に必要なファイル群とメタデータ
- 読み取り専用のレイヤー構造
- 実行時に差分レイヤーが追加される

### Layer (レイヤー)

- イメージを構成する差分単位
- 各 Dockerfile 命令ごとに作られる
- 読み取り専用で再利用可能

### ファイルシステムの仕組み

- イメージレイヤー
  - 読み取り専用
- コンテナ実行時
  - 差分レイヤー（書き込み可能）が追加される

### OverlayFS / overlay2

- Linux の Union Filesystem
- 複数ディレクトリを 1 つに合成して見せる
- Docker の標準ストレージドライバー

### RootFS

- コンテナから見えるルートファイルシステム
- イメージレイヤーを合成した結果
- / としてマウントされる
- runc がマウント処理を行う

## 5. Linux カーネル機能

### 隔離の仕組み

Docker の隔離は Linux カーネルの 2 大機能で実現

#### namespace (名前空間)

- プロセスから見える世界を分離する仕組み
- 各コンテナは独自の namespace を持つ

- namespace の種類
  - **PID namespace**
    - プロセス ID 空間を分離する
    - コンテナ内では PID 1 から始まる
    - ホストの PID とは別に見える
  - **NET namespace**
    - ネットワークスタックを分離する namespace
    - 各コンテナは独自の NIC・IP を持つ
  - **MNT namespace**
    - マウントポイントを分離
  - **UTS namespace**
    - ホスト名・ドメイン名を分離
  - **IPC namespace**
    - プロセス間通信を分離
  - **User namespace**
    - ユーザー ID・グループ ID を分離

#### cgroup (コントロールグループ)

- プロセスが使えるリソース量を制御する仕組み
- どれだけ使っていいかを制御
- Docker の --memory や --cpus の実体

- cgroup の種類
  - **CPU cgroup**
    - CPU 使用率・時間の制限
  - **Memory cgroup**
    - メモリ使用量の制限
  - **IO cgroup**
    - ディスク I/O の制限
  - **PIDs cgroup**
    - プロセス数の制限

### プロセス管理

- コンテナは単なる Linux プロセス
- **fork**
  - 親プロセスを複製
- **clone**
  - namespace 等を指定して生成
- **exec**
  - プロセスを別プログラムに置換

## 6. コンテナライフサイクル

### 状態遷移

- container lifecycle は「状態遷移の塊」
  - moby はイベント駆動の state machine
  - created → running → paused → stopped → dead

### ロック戦略

- API → backend → container のロック階層

## 7. 関連技術・標準

### OCI (Open Container Initiative)

- コンテナ技術の標準仕様を策定する団体
- 主な仕様
  - Image Spec
    - イメージフォーマット
  - Runtime Spec
    - コンテナ実行環境
  - Distribution Spec
    - イメージ配布

### ネットワークプロトコル

#### TCP

- インターネット上でデータを確実に届けるための通信プロトコル
- 送信順序の保証、再送制御、誤り検出を行う
- Web 通信やメールなど、信頼性が重要な通信で使われる

#### UDP

- 高速でシンプルな通信を行うためのプロトコル
- 到達保証や順序制御、再送は行わず、送ったデータはそのまま届ける
- 音声通話、動画配信、オンラインゲームなど低遅延が重要な用途で利用
- **双方向通信**
  - 技術的には常に「片方向のデータ送信」を行い、双方向は双方が送り合うことで成立
- **片方向通信**
  - syslog とかはクライアントがログをサーバへ一方的に送信
  - 応答を待たず、成功・失敗の確認もしない

#### SCTP

- TCP や UDP の特徴を併せ持つ信頼性の高い通信プロトコル
- 1 つの接続で複数ストリームを扱え、順序遅延を防ぐ
- 通信経路の切り替えにも対応

### Swarm

- Docker 公式の軽量オーケストレーター
- 複数の Docker ホストを 1 つのクラスタとして扱う仕組み
- 現状ではあまり使われてない
  - Kubernetes に負けた

### containerd

#### Lease

- containerd のガベージコレクションから一時的にリソースを保護する
- リースを取得している間は、そのコンテンツが削除されない
- 作業が終わったらリースを解放して、GC 対象に戻す
- プル/ビルド中に GC が走ると、途中のデータが消える可能性がある

#### Content Store

- コンテンツをダイジェスト(ハッシュ値)でアドレッシングするキーバリューストア
- 圧縮された「メタデータ」

  例:

```md
/var/lib/containerd/io.containerd.content.v1.content/
└── blobs/
└── sha256/
├── abc123... (マニフェスト)
├── def456... (Config)
└── 789xyz... (レイヤー)
```

#### Snapshotter

- 実際のファイルシステムレイヤーを管理するコンポーネント
- 展開された「実ファイルシステム」
- Snapshot の種類
  - Active
    - 読み書き可能
  - Committed
    - 読み取り専用

#### Image Store

- イメージのメタデータを管理するデータベース
- この名前はこのダイジェストを指す

#### Manifest

- 1 つのプラットフォーム用イメージの「設計図」
- 内容
  - Config のダイジェスト
  - Layers のダイジェストリスト
  - Platform 情報
- Manifest List / Index
  - 複数プラットフォームのマニフェストをまとめたもの

#### Descriptor

- コンテンツを参照するためのメタデータ

#### Digest

- コンテンツの SHA256 ハッシュ値
- 特性
  - 一意性: 同じ内容なら必ず同じダイジェスト
  - 整合性検証: データ改ざん検出
  - 重複排除: 同じダイジェストなら再利用

#### Attestation

- イメージの「証明書」や「来歴情報」
- 種類
  - SBOM
    - Software Bill of Materials
  - Provenance
    - ビルドの来歴
  - Signature
    - 署名

```md
ubuntu:latest
├── Manifest (実際のイメージ)
└── Attestation Manifest (証明書)
└── SBOM: このイメージには curl 7.68.0 が含まれる
```

#### Platform Matcher

- 実行環境に合ったプラットフォームを選択するロジック

### docker event

- Docker デーモンで発生するすべての操作(コンテナの起動、停止、イメージの pull)がイベントとして記録され、ストリーミング配信される
- Dockerデーモン内で起きている出来事をリアルタイムで監視する
- 特徴
  - リアルタイムストリーミング
  - 過去 256 件のイベントをバッファ保持
  - ナノ秒精度のタイムスタンプ
  - 強力なフィルタリング機能
