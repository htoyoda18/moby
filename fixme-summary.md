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
