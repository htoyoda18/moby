# プロジェクトの背景・歴史編

## 19. プロジェクトの背景・歴史

「なぜ moby というリポジトリ名で、Docker Engine の実体がここにあるのか」を理解するための背景知識。

### dotCloud から Docker へ

- 2013 年、PaaS 企業 dotCloud が社内で使っていたコンテナ管理ツールを OSS 化したのが Docker の始まり
- 反響を受けて会社自体も Docker Inc. に改称し、コンテナ技術に事業を集中させた
- 当時は LXC をベースにしていたが、後に自社の libcontainer に置き換えた

### OCI (Open Container Initiative) の設立（2015年）

- Docker の急成長に伴い、「コンテナフォーマット・ランタイムを一企業の実装に依存させない」という機運が高まった
- 2015 年、Docker が中心となり CoreOS 等と共に Linux Foundation 傘下で OCI を設立
- Docker は自社の **libcontainer を runc として OCI に寄贈**し、OCI Runtime Spec の実装のリファレンスとなった
- これにより「コンテナランタイム」という概念が Docker という一企業の実装から標準仕様（[docker-fundamentals.md](docker-fundamentals.md), [docker-advanced.md](docker-advanced.md) の OCI 参照）へと切り離された

### containerd の切り出し（2016〜2017年）

- Docker Engine の中からコンテナライフサイクル管理の部分を **containerd** として独立コンポーネント化
- 2017 年に CNCF（Cloud Native Computing Foundation）へ寄贈、2019 年に CNCF のグラデュエートプロジェクトとなった
- これにより containerd は Docker だけでなく Kubernetes からも直接使われる共通基盤になった（[docker-runtime-deepdive.md](docker-runtime-deepdive.md) の CRI 参照）

### Moby Project の発足（2017年）

- 2017 年の DockerCon で、Docker Engine を構成するコンポーネント群を再編し、**Moby Project** としてオープンソースのアップストリームプロジェクトに分離
- 位置づけ
  - **Moby**: 誰でも自由に組み合わせてコンテナシステムを構築できる、モジュール化されたコンポーネント集合（このリポジトリ自体）
  - **Docker (製品)**: Moby のコンポーネントを特定の構成で組み立てた、Docker Inc. のブランド製品
  - 例えるなら Chromium（オープンソース）と Google Chrome（製品）の関係に近い
- 名前の由来は Docker のマスコットである鯨「Moby Dock」から

### Docker Enterprise の売却（2019年）とその後

- 2019 年、Docker Inc. はエンタープライズ向け事業（Docker Enterprise）を Mirantis に売却
- Docker Inc. は Docker Desktop / Docker Hub / 開発者向けツールに事業を集中する方向へ転換
- 一方 Kubernetes 業界では CRI 経由の containerd 直接利用が主流になり、kubelet 内の dockershim は Kubernetes 1.24（2022年）で削除された

### まとめ: 現在の構図

```text
OCI (仕様策定)          ─ Runtime Spec / Image Spec / Distribution Spec
  ↑ 準拠
Moby Project (OSS)      ─ dockerd, containerd 等のコンポーネント群（このリポジトリ）
  ↑ 組み立て
Docker (製品)           ─ Docker Desktop, Docker Engine (商用配布)
```

- 「なぜ moby というリポジトリで Docker の実体を追っているのか」の答えは、
  Docker という製品の中身が、この Moby Project のコンポーネントを組み立てたものだから
