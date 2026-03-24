# daemon/internal/fstype

パスからファイルシステムの種類を検出するパッケージ。

## 概要

Linuxの `statfs` システムコールを使い、ファイルシステムの「マジックナンバー」を取得する。各ファイルシステムにはカーネルが定義した一意のID（マジックナンバー）があり、これでファイルシステムの種類を判別できる。

## 主要な型・定数

### FsMagic (uint32)
ファイルシステムIDを表す型。以下のような定数が定義されている:
- `FsMagicOverlay` (0x794C7630) - OverlayFS
- `FsMagicBtrfs` (0x9123683E) - Btrfs
- `FsMagicZfs` (0x2fc12fc1) - ZFS
- `FsMagicAufs` (0x61756673) - AUFS
- `FsMagicExtfs` (0x0000EF53) - ext2/3/4
- `FsMagicXfs` (0x58465342) - XFS
- その他多数（tmpfs, nfs, fuse, squashfs, etc.）
- `FsMagicUnsupported` (0x00000000) - 未サポート/検出不可

### FsNames
`map[FsMagic]string` - マジックナンバーからファイルシステム名への変換マップ。

## 公開API

- `GetFSMagic(rootpath string) (FsMagic, error)` - パスのファイルシステムIDを返す
  - Linux: `unix.Statfs()` で `Statfs_t.Type` を取得
  - 非Linux: `FsMagicUnsupported` を返す（エラーなし）

## 利用箇所

ストレージドライバの初期化時に、バッキングファイルシステムの種類を確認するために使用:
- `daemon/graphdriver/overlay2/overlay.go` - OverlayFS用
- `daemon/graphdriver/btrfs/btrfs.go` - Btrfs用
- `daemon/graphdriver/zfs/zfs_linux.go` - ZFS用
- `daemon/graphdriver/fuse-overlayfs/fuseoverlayfs.go` - FUSE-OverlayFS用

例えば overlay2 ドライバは、バッキングFSが ext4/xfs/btrfs/tmpfs などサポート対象かどうかをこのパッケージで確認する。
