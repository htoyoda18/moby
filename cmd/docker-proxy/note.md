- 概要
  - ホスト ↔ コンテナ間の通信を仲介するプロセス
    - Docker コンテナとホストマシン間のネットワークトラフィックを転送するための独立したプロセス
    - TCP、UDP、SCTP の 3 つのプロトコルに対応
    - これにより、Docker のポートマッピングが実現される
  - 本来は iptables (DNAT) だけで転送出来る
    - localhost 絡みの通信
    - IPv6 / IPv4 の混在
    - 古いカーネルの挙動差
    - hairpin NAT
- docker-proxy の動作イメージ
  1. Docker がホスト側ポートをバインド
  2. docker-proxy が listen()
  3. 接続を受ける
  4. コンテナ IP:PORT へ connect()
  5. 双方向で read/write を中継
- 内部構造
  - main_linux.go
    - エントリポイント
  - network_proxy_linux_test.go
  - proxy_linux.go
    - プロキシの基本インターフェースを定義
  - sctp_proxy_linux.go
    - SCTP 接続のプロキシ実装
  - tcp_proxy_linux.go
    - TCP 接続のプロキシ実装
  - udp_proxy_linux_test.go
  - udp_proxy_linux.go
    - UDP 用のステートフルプロキシ
