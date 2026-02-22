# Docker 内部実装編

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
