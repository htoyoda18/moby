# daemon/internal/filedescriptors

プロセスが使用中のファイルディスクリプタ(FD)数を取得するパッケージ。

## ファイルディスクリプタとは

- OS が「開いているリソース」を管理するための**整数のハンドル（ID）**
- ファイルやソケットなどを操作するための番号

## 公開 API

- `GetTotalUsedFds(ctx context.Context) int` - 使用中の FD 数を返す

## 仕組み（Linux）

2 段階の戦略で取得する:

1. **Fast-path（Linux 6.2 以降）**: `/proc/<pid>/fd` を `stat()` して `size` フィールドから取得。Linux 6.2 のコミット `f1f1f2569901` で、`/proc/<pid>/fd` の `stat.Size` にオープンファイル数が格納されるようになった
2. **Slow-path（フォールバック）**: `/proc/<pid>/fd` ディレクトリを開いて `Readdirnames(100)` で 100 件ずつ読み取り、エントリ数をカウント。この場合、ディレクトリ自体を開くため FD が 1 つ余分にカウントされる

非 Linux 環境では常に `-1` を返す（未サポート）。

## 利用箇所

- `daemon/info.go` - `docker info` コマンドのレスポンスに含まれるシステム情報として使用

## ファイル名の typo

`filiedescriptors_linux.go` - "fili" は "file" の typo と思われるが、そのまま残っている。
