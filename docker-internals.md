# Docker 内部実装編

## 4. イメージとファイルシステム

### Image (イメージ)

#### イメージとは

- **Dockerfile をビルドした結果**が Image
- コンテナを起動するための「テンプレート」や「設計図」
- 読み取り専用で、何度でも再利用できる
- 複数のレイヤーが積み重なった構造

#### イメージの正体

イメージは以下の 2 つで構成される:

**1. ファイルシステムのスナップショット**

- アプリケーション実行に必要なすべてのファイル
  - OS の基本ファイル（/bin、/lib など）
  - アプリケーションコード
  - ライブラリや依存パッケージ
  - 設定ファイル

**2. メタデータ**

- 実行時の設定情報
  - デフォルトで実行するコマンド（CMD, ENTRYPOINT）
  - 環境変数（ENV）
  - 公開するポート（EXPOSE）
  - 作業ディレクトリ（WORKDIR）
  - など

#### イメージの識別

- Docker イメージは「リポジトリ名・タグ・ダイジェスト」で識別する
  - リポジトリ名: イメージの名前
    - nginx, myapp/backend
  - タグ: バージョンや用途を表す
    - latest, 1.25, prod
  - ダイジェスト: イメージ内容のハッシュ
    - sha256:...

#### イメージとコンテナの関係

```text
Image (設計図)              Container (実行中のインスタンス)
┌─────────────┐            ┌─────────────┐
│  nginx      │  docker    │  Container  │
│  Image      │  ─run───>  │  #1         │
│             │            └─────────────┘
│ 読み取り専用  │            ┌─────────────┐
│ テンプレート  │  docker    │  Container  │
│             │  ─run───>  │  #2         │
└─────────────┘            └─────────────┘
                           ↑ 同じImageから何個でも起動可能
```

- **Image**: 1 つ（静的な設計図）
- **Container**: 何個でも起動できる（動的なインスタンス）
- 例: クラスとインスタンスの関係に似ている

#### 具体例: nginx イメージ

```bash
# イメージをビルド
docker build -t mynginx .

# このイメージから複数のコンテナを起動できる
docker run -d --name web1 mynginx
docker run -d --name web2 mynginx
docker run -d --name web3 mynginx
```

- `mynginx` というイメージは 1 つ
- そのイメージから `web1`, `web2`, `web3` という 3 つのコンテナを起動
- 各コンテナは独立して動作するが、元は同じイメージ

#### イメージの作成方法

**1. Dockerfile からビルド（最も一般的）**

```bash
docker build -t myapp:v1 .
```

**2. 実行中のコンテナから作成**

```bash
docker commit <container-id> myapp:v2
```

**3. tar ファイルからインポート**

```bash
docker import myapp.tar myapp:v3
```

**4. レジストリから取得**

```bash
docker pull nginx:latest
```

#### イメージの特性

- **不変性（Immutable）**

  - 一度作成されたイメージは変更されない
  - 変更したい場合は新しいイメージを作る

- **レイヤー構造**

  - 複数のレイヤーが積み重なってできている
  - 下層レイヤーは他のイメージと共有可能

- **タグ管理**
  - 同じイメージに複数のタグを付けられる
  - 例: `nginx:1.21`, `nginx:latest`

### Layer (レイヤー)

#### レイヤーとは

- イメージを構成する「差分」の単位
- Dockerfile の各命令（RUN, COPY など）ごとに 1 つのレイヤーが作られる
- 読み取り専用で、一度作られたら変更されない
- 複数のイメージで共有・再利用される

#### 具体例: Dockerfile とレイヤーの対応

```dockerfile
FROM ubuntu:22.04          # ← レイヤー1: Ubuntu ベースイメージ
RUN apt-get update         # ← レイヤー2: apt-get update の結果
RUN apt-get install -y nginx  # ← レイヤー3: nginx のインストール結果
COPY index.html /var/www/  # ← レイヤー4: index.html のコピー
```

このイメージは **4 つのレイヤー** で構成される:

```text
┌─────────────────────────┐
│ Layer 4: index.html     │ ← COPY 命令
├─────────────────────────┤
│ Layer 3: nginx installed│ ← RUN apt-get install
├─────────────────────────┤
│ Layer 2: apt updated    │ ← RUN apt-get update
├─────────────────────────┤
│ Layer 1: Ubuntu 22.04   │ ← FROM ubuntu
└─────────────────────────┘
```

#### なぜレイヤーが必要なのか

**1. ディスク容量の節約**

同じベースイメージを使う複数のイメージは、下層レイヤーを共有できる:

```text
Image A (nginx)          Image B (apache)
┌──────────────┐         ┌──────────────┐
│ nginx層      │         │ apache層     │
├──────────────┤         ├──────────────┤
│ Ubuntu 22.04 │ ←─────→ │ Ubuntu 22.04 │ (同じレイヤーを共有)
└──────────────┘         └──────────────┘
```

**2. ビルドの高速化**

Dockerfile を修正しても、変更された行より前のレイヤーは再利用される:

```dockerfile
FROM ubuntu:22.04          # キャッシュ利用 ✓
RUN apt-get update         # キャッシュ利用 ✓
RUN apt-get install -y nginx  # キャッシュ利用 ✓
COPY index.html /var/www/  # ← ここだけ再ビルド (index.html を変更した場合)
```

**3. イメージ配布の効率化**

Docker レジストリから pull する際、すでに持っているレイヤーはスキップされる

#### コンテナレイヤー（書き込み可能レイヤー）

コンテナを起動すると、イメージレイヤーの上に **薄い書き込み可能レイヤー** が追加される:

```text
コンテナ実行時:
┌─────────────────────────┐
│ Container Layer (R/W)   │ ← コンテナ内での変更はここに保存
├─────────────────────────┤
│ Layer 4: index.html     │ (読み取り専用)
├─────────────────────────┤
│ Layer 3: nginx installed│ (読み取り専用)
├─────────────────────────┤
│ Layer 2: apt updated    │ (読み取り専用)
├─────────────────────────┤
│ Layer 1: Ubuntu 22.04   │ (読み取り専用)
└─────────────────────────┘
```

- コンテナ内でファイルを作成/変更 → コンテナレイヤーに保存
- コンテナを削除 → コンテナレイヤーも削除（イメージレイヤーは残る）
- 同じイメージから複数コンテナを起動 → それぞれ独自のコンテナレイヤーを持つ

### ファイルシステムの仕組み

- イメージレイヤー
  - 読み取り専用
- コンテナ実行時
  - 差分レイヤー（書き込み可能）が追加される

### OverlayFS / overlay2

- Linux の Union Filesystem
- 複数ディレクトリを 1 つに合成して見せる
- Docker の標準ストレージドライバー

### Storage Driver / Graph Driver

- Docker がイメージレイヤーを管理するための抽象化層
- dockerd がファイルシステムレイヤーを扱うためのインターフェース
- レイヤーの作成、マウント、削除などを担当

#### 主なドライバーの種類

- **overlay2** (推奨・標準)
  - 現在の標準ストレージドライバー
  - Linux カーネル 4.0 以降で利用可能
  - 高速で効率的
- **aufs**
  - 古い Ubuntu で使われていた
  - 現在は非推奨
- **devicemapper**
  - RHEL/CentOS 7 などで使われていた
  - 現在は overlay2 が推奨
- **btrfs**
  - Btrfs ファイルシステムを使用
  - スナップショット機能を活用
- **zfs**
  - ZFS ファイルシステムを使用
  - エンタープライズ向け

#### Graph Driver の役割

- レイヤーの管理
  - 作成 (Create)
  - 削除 (Remove)
  - マウント (Get)
- メタデータの保持
- ディスク使用量の計算
- ストレージ最適化

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

### コンテナ再起動ポリシー

- コンテナが停止したときに自動で再起動するかを制御する
- 主に --restart オプションで指定する
  - no: 自動再起動しない（デフォルト）
  - always: 停止理由に関係なく常に再起動する
  - unless-stopped: 手動停止しない限り再起動する（再起動後も継続）
  - on-failure: 異常終了（非 0 終了コード）のときだけ再起動する

### ロック戦略

- API → backend → container のロック階層
