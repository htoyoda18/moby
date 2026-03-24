# daemon/internal/filedescriptors

プロセスが使用中のファイルディスクリプタ(FD)数を取得するパッケージ。

## 公開API

- `GetTotalUsedFds(ctx context.Context) int` - 使用中のFD数を返す

## 仕組み（Linux）

2段階の戦略で取得する:

1. **Fast-path（Linux 6.2以降）**: `/proc/<pid>/fd` を `stat()` して `size` フィールドから取得。Linux 6.2のコミット `f1f1f2569901` で、`/proc/<pid>/fd` の `stat.Size` にオープンファイル数が格納されるようになった
2. **Slow-path（フォールバック）**: `/proc/<pid>/fd` ディレクトリを開いて `Readdirnames(100)` で100件ずつ読み取り、エントリ数をカウント。この場合、ディレクトリ自体を開くためFDが1つ余分にカウントされる

非Linux環境では常に `-1` を返す（未サポート）。

## 利用箇所

- `daemon/info.go` - `docker info` コマンドのレスポンスに含まれるシステム情報として使用

## ファイル名のtypo

`filiedescriptors_linux.go` - "fili" は "file" のtypoと思われるが、そのまま残っている。
