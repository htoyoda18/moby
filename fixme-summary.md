# このセッションで確認した FIXME 一覧

### `api/pkg/authconfig/authconfig.go:22` / `authconfig_test.go:135`

空の `AuthConfig{}` を渡すと古いデーモンが "io.EOF" を返す可能性があるため、空チェックの早期リターンがコメントアウトされたまま。どのコードパスが壊れるか特定できていない。

### `api/types/swarm/service_create_response.go:20`

`Warnings []string` に `omitempty` がないため、警告がない場合も `"Warnings":null` が JSON に含まれる。API の後方互換性への影響で保留されている。

### `client/client_options.go:302`

`FromEnv` オプションが `http.Client` ごと丸ごと置き換えており、既存のタイムアウト等の設定が失われる。`WithTLSClientConfig` はトランスポートのみ更新するのと不整合。

### `client/container_exec_test.go:53` ✅ 修正済み

`TestExecCreate` が `User` フィールドしか検証しておらず、他の 10 フィールドのマッピングが正しいかテストされていなかった。全フィールドを `is.DeepEqual` で検証するよう修正済み。

### `client/container_resize.go:28` / `:53`

`ContainerResizeOptions.Height/Width` が `uint`（64-bit OS では 64-bit）だが、サーバー側は `uint32` で受け取る。`uint32` 上限を超えた値を渡すとサーバーがエラーを返す。

### `client/image_pull.go:32`

`refStr` の使われ方が「フル参照を渡す場合」と「trusted content で参照名とダイジェストを分けて渡す場合」の 2 パターン混在しており、統一されていない。

### `client/request.go:186`

`net.Error` のうちタイムアウトと特定文字列マッチのみ接続エラーとして扱われており、他の `net.Error`（例: `broken pipe`）が素通りしてしまう。

### `client/pkg/progress/progress.go:42`

クローズ済みチャネルへの書き込みパニックを `recover()` で握りつぶすワークアラウンド。根本修正はチャネルのライフサイクル管理の改善。

### `client/pkg/streamformatter/streamformatter.go:146` 🔧 修正予定

`p.Start`（開始時刻）を使った「残り時間」表示ロジックが実装済みだが、`progress.Progress` 構造体に `Start` フィールドがなく値が渡ってこないためデッドコードになっている。実装計画: `progress-start-impl-plan.md` 参照。

### `daemon/archive_windows.go:151`

`copyUIDGID` パラメータを受け取るが Windows では UID/GID の chown が OS レベルで非サポートのため黙って無視される。エラーも警告も返さない。

### `daemon/container_operations.go:973`

`--network container:<自分自身>` の自己参照チェックが `docker create` 時に行われておらず、`docker start` まで検出されない。エラー種別も本来 `InvalidParameter` のはずが `System` になってしまっている。

### `daemon/container.go:103`

`containers.Add`（メモリ登録）と `CheckpointTo`（ディスク書き込み）がアトミックでなく、後者が失敗するとメモリとディスクの状態が不整合になる。重複 ID の上書きも防がれていない。

### `daemon/container.go:144`

`Container.Args` は `Config.Entrypoint` と `Config.Cmd` から導出される派生データなのに独立フィールドとして重複保持されており、設計上の負債になっている。

### `daemon/create_windows.go:45`

Linux版には「コンテナ起動前にコンテナFSへVOLUMEディレクトリの内容をコピーする」処理があるが、Windows版では利用している `FollowSymLinkInScope` がWindowsのボリューム形式パス（例: `c:\myvol`）に対応していないためスキップされている。`@swernli` による別途対応が予定されており暫定的に除外。TP5時点ではHCSがコンテンツ入りマップドディレクトリを非サポートなため実害は限定的。

### `daemon/daemon_linux.go:151`

`setupResolvConf` 内で `resolvconf.Path()` を呼ぶ際、libnetwork 内の `internal/resolvconf.Path` が使えず（`internal` パッケージのため）、外部パッケージ `github.com/moby/moby/v2/vendor/...` の同名関数に頼っている。libnetwork の `internal` を外部公開するか、パッケージ構造を整理することで解消できる。

### `daemon/devices_nvidia_linux.go:147`

NVIDIAコンテナランタイムフックを `Prestart` フックとして登録しているが、`Prestart` はOCI仕様で非推奨。`CreateRuntime` フックが最も近い代替だが、フックの具体的な処理内容によっては `CreateContainer` や `StartContainer` が適切な可能性もあるため、調査・移行が保留されている。

### `daemon/exec_linux_test.go:43`

`execSetPlatformOpts` において、コンテナに `--privileged` が設定されていてもカスタムAppArmorプロファイルが指定されていると後者が優先される挙動がある。`--privileged` はAppArmor・seccomp・SELinuxをすべて無効化すべきであり、これはバグの可能性が高い。問題箇所: `daemon/exec_linux.go:32-40`。テスト内の `expectedProfile: unconfinedAppArmorProfile` はこのバグが修正されるまでコメントアウトされたまま。
修正自体は簡単だが、合意形成が難しい。

### `daemon/health_test.go:47`

`TestHealthStates` の実行に約3秒かかっており、ユニットテストとして許容できない時間がかかっている。主因は `CommitInMemory` が JSON encode/decode で `Container` 全体をディープコピーしていること（10回呼ばれる）。詳細: `fixme-detail-daemon-health-exec.md`。

### `daemon/mounts.go:32`

`prepareMountPoints` では `config.Volume == nil`（ボリュームマウントでない）の場合に処理をスキップしている。ただしバインドマウントや tmpfs も `Volume == nil` になるため、`config.Type` を追加確認してバインドマウント等を明示的に除外すべきかが問われている。現状でも `LiveRestore` が呼ばれないだけで実害は限定的だが、意図が不明瞭なまま。

### `daemon/oci_linux.go:258` / `:301` / `:337`

同一の問題が net / IPC / PID 名前空間の3箇所に存在する。`--network container:A` かつ `--ipc container:B` のように複数コンテナの名前空間を同時に共有する場合、ユーザー名前空間のパスが後から処理されるコンテナの PID で**上書き**される。例: ネットNS共有でコンテナAのユーザーNSパスを設定後、IPCのNS共有でコンテナBのユーザーNSパスに上書きされ、最終的な名前空間構成が不整合になる可能性がある。Issue [#46210](https://github.com/moby/moby/issues/46210) で追跡中。

### `daemon/oci_windows.go:368`

Windows の `credentialspec` セキュリティオプション処理でキー名の比較に `strings.EqualFold` を使っており、大文字小文字を区別しない（`CREDENTIALSPEC`・`CredentialSpec`・`credentialspec` がすべて受け入れられる）。他のセキュリティオプションは大文字小文字を区別するため一貫性がなく、意図しないオプション名の受け入れにつながる。

### `daemon/runtime_unix.go:273`

`isPermissibleC8dRuntimeName` 内でランタイム名の検証ロジック（`.` を含むか、絶対パスでないかなど）を手書きで実装しているが、これは containerd の内部実装を複製したもの。本来は containerd モジュール側のユーティリティを使いたいが、該当の `shim.BinaryName` は `shim` パッケージに属しており依存関係が大量についてくるため直接利用できない。containerd 側で検証ロジックを独立したパッケージに切り出してもらう必要がある。

### `daemon/start.go:286`

`conditionalUnmountOnCleanup` が失敗した場合のフォールバックとして、graphdriver のマウントを ID で直接クリーンアップしている。コメントによれば「graphdriver の参照カウントがリファクタリングされたら削除する」とのこと。containerd ベースのストレージへの移行が進めば不要になる暫定的なワークアラウンド。

### `daemon/stats_collector.go:16`

`newStatsCollector` の中で Linux の `machineMemory`（物理メモリ量）の初期化を行っているが、本来この処理は統計コレクターの生成とは無関係なため別の場所（デーモン初期化時など）に移すべき。関心の分離が不十分な設計上の問題。

### `daemon/stats.go:121`

`GetContainerStats` 内でシステム全体の CPU 使用量を `getSystemCPUUsage()` で取得しているが、Linux では containerd 側に移管すべきとのコメントがある。Windows は HCS から直接ネットワーク統計を取得するため対象外。containerd との統合が進んだ段階での改善項目。

### `daemon/top_unix_test.go:21`

`validatePSArgs` は `ps` の引数中に `=PID...` というパターン（値が "PID" で始まるカラム指定）を禁止している。これは `pid=PID` のような PID カラムの直接指定を弾くための正規表現だが、`uid=PIDX` のような無害な指定（値がたまたま "PID" で始まるだけ）も巻き込んでエラーにしてしまう。テストケース `"ae -o pid=PID -o uid=PIDX": true` にコメントで「本来は禁止しなくてよい」と記されている。正規表現 `psArgsRegexp` の精度が不十分。

### `daemon/command/daemon.go:661`

Windows では設定ファイルのデフォルトパスが `--data-root` に依存しており、`data-root` が変更されると設定ファイルのパスも変わってしまう。`"daemon.json"` という固定ファイル名と可変の `--data-root` に依存しない、より安定したデフォルトパスが必要とされているが、Windows 固有のパス規約上の代替が見つかっていない。

### `daemon/command/daemon.go:1100`

`--raw-logs` オプションによる ANSI カラー無効化の実装が、containerd のログパッケージ内部の `*logrus.TextFormatter` に型アサートして直接 `DisableColors` フィールドを書き換えるという脆い方法に依存している。containerd のログパッケージがカラー制御 API を公開していないため、内部実装に依存せざるを得ない状態。

### `daemon/command/docker.go:19`

`newDaemonCommand` が呼ばれると必ず `config.New()` が実行され、バイナリパスの探索など重い初期化が走る。`dockerd --version` だけを実行したい場合でも同様で、バージョン表示に不要な処理が含まれている。バージョン出力とデーモン設定の初期化を分離する必要がある。
