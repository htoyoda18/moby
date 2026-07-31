# プラグインとロギングドライバ編

## 17. プラグインとロギングドライバ

### プラグインシステムの全体像

- Docker はいくつかの拡張点を、外部プロセス（プラグイン）に委譲できる仕組みを持つ
- 実装: `pkg/plugins`（プラグインの検出・通信の共通基盤）, `pkg/plugingetter`（プラグイン取得インターフェース）
- プラグインは Unix ソケットまたは HTTP 経由の独自 API で dockerd と通信する（[docker-networking.md](docker-networking.md) の Volume driver で触れた仕組みと同系統）
- 主な拡張点
  - **Volume プラグイン**: [docker-storage.md](docker-storage.md) 参照。リモートストレージ対応など
  - **Network プラグイン**: [docker-networking.md](docker-networking.md) の CNM における `remote` ドライバがこれに相当
  - **Authorization プラグイン**: API リクエストごとに許可/拒否を判定するフック（例: 特定ユーザーに `docker exec` を禁止する等）

### ロギングドライバ

- コンテナの標準出力/標準エラーをどこに、どういう形式で送るかを切り替える仕組み
- 実装: `daemon/logger/`
  - `jsonfilelog/`: デフォルト。JSON 行形式でホストのファイルに保存（`docker logs` が読むのはこれ）
  - `journald/`: systemd-journald に送る
  - `syslog/`: syslog プロトコルで外部へ送信
  - `fluentd/`: Fluentd に転送（集約基盤との連携に使われる）
  - `awslogs/`, `gcplogs/`, `splunk/`, `gelf/`: 各種クラウド/ログ収集基盤向け
- 共通の仕組み
  - `copier.go`: コンテナプロセスの stdout/stderr を各ドライバへコピーする中心的なロジック
  - `ring.go`: メモリ上のリングバッファ（`--log-opt max-size` 等でログ肥大化を防ぐ仕組みの土台）
- ロギングドライバ自体もプラグインとして外部実装できる（`plugin.go`）

### 設定方法

```bash
# デーモン全体のデフォルトを変更 (daemon.json)
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}

# コンテナ単位で上書き
docker run --log-driver=journald nginx
```

- `jsonfilelog` 以外のドライバを使うと `docker logs` コマンドが使えなくなる場合がある点に注意
  （ドライバがローカル読み出しに対応していないため）
