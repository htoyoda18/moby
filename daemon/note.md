- 概要
  - Docker デーモン（dockerd）のコア実装が含まれている
- 内部構造
  - builder
    - イメージビルド機能
  - cluster
    - Swarm クラスタ機能
  - command
  - config
    - デーモン設定
  - container
    - コンテナのライフサイクル管理
  - containerd
    - containerd との統合レイヤー
  - events
    - イベント発行システム
  - graphdriver
    - ストレージドライバ
  - images
    - イメージ管理(pull, push, build など)
  - initlayer
  - internal
  - libnetwork
    - ネットワーク実装
  - linkss
  - listeners
  - logger
    - ログドライバ
  - names
  - network
    - ネットワーク管理
  - pkg
  - server
  - snapshotter
    - スナップショット管理
  - stats
    - コンテナの統計情報収集
  - testdata
  - volume
    - ボリューム管理
  - apparmor_default_unsupported.go
  - apparmor_default.go
  - archive_tarcopyoptions_unix.go
  - archive_tarcopyoptions.go
  - archive_unix.go
  - archive_windows.go
  - archive.go
    - コンテナとホスト間のファイルコピー
  - attach.go
    - コンテナへのアタッチ
  - auth.go
  - build.go
    - Docker イメージビルド
  - cdi.go
  - changes.go
  - checkpoint.go
    - コンテナのチェックポイント/リストア
  - cluster.go
  - commit.go
    - コンテナからイメージを作成
  - configs.go
  - container_linux.go
  - container_operations_test.go
  - container_operations_unix.go
  - container_operations_windows.go
  - container_operations.go
    - コンテナ操作(start, stop, kill, pause など)
  - container_unix_test.go
  - container_windows.go
  - container.go
  - containerfs_linux_test.go
  - containerfs_linux.go
  - content.go
  - create_unix.go
  - create_windows.go
  - create.go
    - コンテナ作成
  - daemon_linux_test.go
  - daemon_linux.go
  - daemon_test.go
  - daemon_unix_test.go
  - daemon_unix.go
  - daemon_unsupported.go
  - daemon_windows_test.go
  - daemon_windows.go
  - daemon.go
    - デーモンの心臓部
  - daemon.go.md
  - debugtrap_unix.go
  - debugtrap_unsupported.go
  - debugtrap_windows.go
  - delete_test.go
  - delete.go
  - dependency.go
  - devices_amd_linux.go
  - devices_nvidia_linux.go
  - devices.go
  - disk_usage.go
  - errors_test.go
  - errors.go
  - events_test.go
  - events.go
  - exec_linux_test.go
  - exec_linux.go
  - exec_windows.go
  - exec.go
    - コンテナ内でのコマンド実行
  - export.go
  - health_test.go
  - health.go
  - hosts_test.go
  - hosts.go
  - id.go
  - image_service.go
  - image_store_choice_test.go
  - image_store_choice.go
  - info_unix_test.go
  - info_unix.go
  - info_windows.go
  - info.go
  - inspect_linux.go
  - inspect_test.go
  - inspect_windows.go
  - inspect.go
  - keys_unsupported.go
  - keys.go
  - kill.go
  - licensing_test.go
  - licensing.go
  - links.go
  - list_test.go
  - list_unix.go
  - list_windows.go
  - list.go
  - logdrivers_linux.go
  - logdrivers_windows.go
  - logs_test.go
  - logs.go
  - migration_test.go
  - migration.go
  - monitor.go
  - mounts.go
  - names.go
  - network_test.go
  - network_windows.go
  - network.go
  - note.md
  - oci_linux_test.go
  - oci_linux.go
  - oci_opts.go
  - oci_utils.go
  - oci_windows_test.go
  - oci_windows.go
  - pause.go
  - prune.go
  - reload_test.go
  - reload_unix.go
  - reload_windows.go
  - reload.go
  - rename.go
  - resize_test.go
  - resize.go
  - restart.go
  - runtime_unix_test.go
  - runtime_unix.go
  - runtime_windows.go
  - seccomp_linux_test.go
  - seccomp_linux.go
  - seccomp_unsupported.go
  - secrets.go
  - start_linux.go
  - start_notlinux.go
  - start_unix.go
  - start_windows.go
  - start.go
  - stats_collector.go
  - stats_unix_test.go
  - stats_unix.go
  - stats_windows.go
  - stats.go
  - stop.go
  - top_unix_test.go
  - top_unix.go
  - top_windows.go
  - unpause.go
  - update_linux_test.go
  - update_linux.go
  - update_windows.go
  - update.go
  - volumes_linux_test.go
  - volumes_linux.go
  - volumes_unit_test.go
  - volumes_unix.go
  - volumes_windows.go
  - volumes.go
  - wait.go
  - workdir.go

## FIXME(thaJeztah) - ネットワーク名"container"の曖昧性バグ

### 問題の概要 (daemon/internal/runconfig/hostconfig.go:8)

`validateNetContainerMode()` において、**"container"という名前のユーザー定義ネットワーク**と**コンテナモードネットワーク（`container:<id>`形式）**を正しく区別できない問題。

### 現在の実装

**daemon/internal/runconfig/hostconfig.go:7-14**
```go
func validateNetContainerMode(c *container.Config, hc *container.HostConfig) error {
    // FIXME(thaJeztah): a network named "container" (without colon) is not seen as "container-mode" network.
    if string(hc.NetworkMode) != "container" && !hc.NetworkMode.IsContainer() {
        return nil  // バリデーションをスキップ
    }

    if hc.NetworkMode.ConnectedContainer() == "" {
        return validationError("invalid network mode: invalid container format container:<name|id>")
    }
    // 以下、コンテナモード専用のバリデーション（Hostname, DNS, PortBindings等の禁止）
}
```

### 関連する実装

**api/types/container/hostconfig.go:482-488**
```go
func containerID(val string) (idOrName string, ok bool) {
    k, v, hasSep := strings.Cut(val, ":")
    if !hasSep || k != "container" {
        return "", false  // コロンがない、または "container:" で始まらない
    }
    return v, true
}

func (n NetworkMode) IsContainer() bool {
    _, ok := containerID(string(n))
    return ok  // "container:xxx" 形式の場合のみ true
}

func (n NetworkMode) ConnectedContainer() string {
    idOrName, _ := containerID(string(n))
    return idOrName  // "container" の場合は空文字列
}
```

### バグの詳細分析

#### 各ケースの動作

| NetworkMode | IsContainer() | 条件判定 | バリデーション | 結果 |
|-------------|---------------|----------|----------------|------|
| `"mynetwork"` | false | `true && true` = true | スキップ | ✅ 正常（カスタムネットワーク） |
| `"container:web"` | true | `true && false` = false | 実行 | ✅ 正常（コンテナモード） |
| `"container:"` | true | `true && false` = false | 実行 → エラー | ✅ 正常（空ID検出） |
| `"container"` | false | `false && true` = false | 実行 → エラー | ❌ **バグ！** |

#### 問題のシナリオ

**シナリオ1: "container"という名前のカスタムネットワークが存在する場合**

```bash
# 1. "container"という名前のネットワークを作成（Dockerは許可する）
$ docker network create container
container

# 2. このネットワークを使ってコンテナを起動しようとする
$ docker run --network=container nginx
Error: invalid network mode: invalid container format container:<name|id>
```

**問題**:
- ユーザーは "container" という名前のネットワークを作成できる
- しかし、そのネットワークを使ったコンテナ起動は失敗する
- エラーメッセージは「container:<name|id> 形式が必要」と誤解を招く

**シナリオ2: コンテナモードの誤入力**

```bash
# コロンを忘れた場合
$ docker run --network=container nginx
Error: invalid network mode: invalid container format container:<name|id>
```

**現在の動作**: エラーが返る（これは正しい）
**問題点**: "container"という名前のカスタムネットワークとの区別がつかない

### 根本原因

条件式 `string(hc.NetworkMode) != "container" && !hc.NetworkMode.IsContainer()` の意図が曖昧：

```go
// 現在のロジック
if (モード名 != "container") AND (container:xxx形式ではない) {
    return nil  // コンテナモードじゃないのでスキップ
}
// それ以外 → コンテナモードとして検証
```

**問題**: `"container"` (コロンなし) は以下の2つの意味を持つ可能性がある：
1. ユーザーが作った "container" という名前のカスタムネットワーク
2. ユーザーのタイプミス（`container:xxx` と入力すべきところ）

### 影響範囲

#### 呼び出し元
- `daemon/internal/runconfig/hostconfig_unix.go:16` - Unix系OSでのバリデーション
- `daemon/internal/runconfig/hostconfig_windows.go:13` - Windowsでのバリデーション

#### 影響レベル
- **ユーザビリティ**: 🔴 High
  - "container" という名前のネットワークが使えない
  - エラーメッセージが誤解を招く

- **セキュリティ**: 🟢 Low
  - セキュリティ上の問題はない（誤動作ではなくエラーになる）

- **発生頻度**: 🟡 Medium
  - "container" という名前のネットワークを作るケースは稀
  - しかし、作成は可能なので潜在的な問題

### 解決策の比較

#### 案1: 条件を明確化（推奨）

```go
func validateNetContainerMode(c *container.Config, hc *container.HostConfig) error {
    // "container:xxx" 形式の場合のみバリデーション実行
    if !hc.NetworkMode.IsContainer() {
        return nil  // コンテナモードではないのでスキップ
    }

    // "container:" (空ID) のチェック
    if hc.NetworkMode.ConnectedContainer() == "" {
        return validationError("invalid network mode: invalid container format container:<name|id>")
    }

    // 以下、既存のバリデーション...
}
```

**メリット**:
- シンプルで明確
- "container" という名前のネットワークを正しく扱える
- コロン忘れのエラーは別の場所で検出される（フォーマットエラー）

**デメリット**:
- `--network=container` （コロンなし）のエラーメッセージが変わる可能性

#### 案2: 明示的にエラーを返す

```go
func validateNetContainerMode(c *container.Config, hc *container.HostConfig) error {
    // 正確に "container" の場合は明示的にエラー
    if string(hc.NetworkMode) == "container" {
        return validationError("invalid network mode: did you mean 'container:<name|id>'? (Note: a network literally named 'container' cannot be used due to ambiguity)")
    }

    if !hc.NetworkMode.IsContainer() {
        return nil
    }

    // 以下、既存のバリデーション...
}
```

**メリット**:
- より分かりやすいエラーメッセージ
- 意図を明確に伝えられる

**デメリット**:
- "container" という名前のネットワークを完全に禁止してしまう

#### 案3: ネットワーク名の予約語化（根本的解決）

ネットワーク作成時に "container" を予約語として禁止する。

**メリット**:
- 根本的に問題を解決
- 将来の混乱を防ぐ

**デメリット**:
- 後方互換性を壊す（既存の "container" ネットワークが使えなくなる）
- 影響範囲が広い（ネットワーク作成ロジックの変更）

### 推奨アプローチ

**短期（Quick Fix）**: 案1（条件の明確化）
- 最小限の変更で問題を解決
- 1ファ���ル、数行の修正
- "container" という名前のネットワークを使用可能にする
- 既存の動作への影響が最小

**長期（根本解決）**: 案3（予約語化） + マイグレーション期間
- 次のメジャーバージョンで "container" を予約語として追加
- 既存の "container" ネットワークには警告を表示
- ドキュメントで推奨されないネットワーク名として明記

### 技術的負債の評価

| 項目 | 評価 |
|------|------|
| 難易度 | 🟢 Very Low（条件式の修正のみ） |
| インパクト | 🔴 High（ユーザビリティ改善、バグ修正） |
| 優先度 | **High（Quick Win）** |
| 推定工数 | 短期案: 30分～1時間、長期案: 1-2日（テスト・ドキュメント含む） |
| テストの必要性 | Medium（エッジケースのテスト追加） |
