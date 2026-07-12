# Docker アーキテクチャと動作原理

Docker の内部実装や動作原理について学んだ内容をまとめたドキュメント集です。
https://chatgpt.com/c/6a53dd90-0004-83e8-a738-74a92a3b9287

## ドキュメント構成

### [基礎編](docker-fundamentals.md)

Docker の基本概念、アーキテクチャ、実行フローを理解する

- **1. Docker の本質**
  - Docker とは何か
  - コンテナの正体
- **2. アーキテクチャ全体像**
  - moby (Docker Engine)
  - 各コンポーネントの役割
- **3. 実行フロー**
  - docker run の裏側
  - docker CLI → dockerd → containerd → runc → Linux kernel

### [内部実装編](docker-internals.md)

ファイルシステム、Linux カーネル機能、ライフサイクルの深掘り

- **4. イメージとファイルシステム**
  - Image, Layer の仕組み
  - OverlayFS / RootFS
- **5. Linux カーネル機能**
  - namespace による隔離
  - cgroup によるリソース制御
  - プロセス管理
- **6. コンテナライフサイクル**
  - 状態遷移
  - ロック戦略

### [詳細トピック編](docker-advanced.md)

イベントシステム、containerd の詳細、標準仕様、関連技術

- **7. イベントシステム**
  - docker event の仕組み
- **8. containerd 深掘り**
  - Lease, Content Store, Snapshotter
  - Image Store, Manifest, Descriptor
  - Digest, Attestation, Platform Matcher
- **9. 標準仕様**
  - OCI (Open Container Initiative)
- **10. 付録: その他の関連技術**
  - ネットワークプロトコル (TCP/UDP/SCTP)
  - Swarm

## 学習の進め方

1. **基礎編** から順に読むことを推奨
2. Docker の全体像を理解したい → 基礎編のみでも十分
3. 内部実装を深く知りたい → 内部実装編へ
4. containerd や OCI の詳細を知りたい → 詳細トピック編へ
