# Moby / Docker コードリーディング学習ロードマップ

## 【第 1 段階】全体像の把握

### 目的

プロジェクト全体の目的・思想・構成を把握する。

### 読むべきもの

- README.md

  - プロジェクトの目的と原則

- ROADMAP.md

  - 今後の方向性

- docs/ 配下の主要ドキュメント

  - アーキテクチャ概要
  - 主要コンポーネントの説明

- api/swagger.yaml
  - API の全体像
  - どんな操作ができるかを理解する

---

## 【第 2 段階】エントリポイントから追う

### 目的

デーモンが **どこから起動し、どう初期化されるか** を理解する。

### 読む流れ

- cmd/dockerd/

  - デーモンの起動処理

- cmd/dockerd/docker.go

  - main 関数
  - 起動時の処理フロー

- 初期化の流れを追う  
  main()
  ↓
  各種初期化
  ↓
  daemon パッケージへ

- daemon/daemon.go
- デーモンの中核
- Daemon 構造体の定義
- 初期化プロセス
- 主要メソッド

---

## 【第 3 段階】機能ごとに深掘り

### 目的

関心のある機能を軸に、実装と責務の分担を理解する。

### 興味のある機能から選んで読む

#### 🐳 コンテナ管理を理解したい

- daemon/container/
- コンテナ構造体

- daemon/start.go
- コンテナ起動処理

- daemon/containerd/
- containerd との統合

- integration/container/
- テストで動作確認

---

#### 🖼️ イメージ管理を理解したい

- daemon/images/
- イメージサービス

- daemon/builder/
- ビルド機能

- integration/image/
- テスト

---

#### 🌐 ネットワークを理解したい

- daemon/network/
- ネットワーク API

- daemon/libnetwork/
- ネットワーク実装

- integration/network/
- テスト

---

#### 💾 ストレージを理解したい

- daemon/graphdriver/
- ストレージドライバ

- daemon/graphdriver/overlay2/
- overlay2 実装

- daemon/volume/
- ボリューム管理
