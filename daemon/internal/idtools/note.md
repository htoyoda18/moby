# daemon/internal/idtools

ファイル/ディレクトリの所有者情報を表す型を提供する、非常にシンプルなパッケージ。

## 主要な型

### Identity
```go
type Identity struct {
    UID int    // Unix User ID
    GID int    // Unix Group ID
    SID string // Windows Security Identifier
}
```

UIDとGIDのペア（Linux/Unix）またはSID（Windows）のいずれかを保持する。両方同時には使わない。

## 利用箇所（13ファイル）

- `daemon/daemon.go` - デーモン初期化時のルートID設定
- `daemon/create_unix.go` - コンテナ作成時の所有者設定
- `daemon/volumes_unix.go` / `daemon/volumes_windows.go` - ボリュームマウント時の所有者
- `daemon/volume/local/` - ローカルボリュームドライバ
- `daemon/volume/service/` - ボリュームサービス
- `daemon/volume/mounts/mounts.go` - マウント設定

## 備考

型定義のみで、ユーティリティ関数はない。UID/GID操作のロジックは利用側にある。daemon内部の各コンポーネントで共通の型として使われるため `internal` パッケージに置かれている。
