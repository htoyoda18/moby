# Moby プロジェクト コントリビューションガイド（日本語版）

このドキュメントは、Mobyプロジェクトへのコントリビューション方法をまとめたガイドです。

## 目次

1. [Mobyプロジェクトについて](#mobyプロジェクトについて)
2. [必要な事前知識](#必要な事前知識)
3. [開発環境のセットアップ](#開発環境のセットアップ)
4. [コントリビューションの流れ](#コントリビューションの流れ)
5. [Issue報告](#issue報告)
6. [Pull Requestの作成](#pull-requestの作成)
7. [コーディング規約](#コーディング規約)
8. [テスト](#テスト)
9. [コミュニティガイドライン](#コミュニティガイドライン)

---

## Mobyプロジェクトについて

Mobyは、Dockerによって作成されたオープンソースプロジェクトで、ソフトウェアのコンテナ化を実現・加速するためのツールキットです。

### プロジェクトの原則

- **モジュラー**: 明確な機能とAPIを持つコンポーネントで構成
- **バッテリー同梱だが交換可能**: 完全なシステムを構築できるコンポーネントを含むが、ほとんどは交換可能
- **使いやすいセキュリティ**: 使いやすさを損なわずにセキュアなデフォルト設定を提供
- **開発者重視**: APIは強力なツールを構築するために機能的で有用

### 対象者

Mobyプロジェクトは、コンテナベースのシステムを修正、実験、発明、構築したいエンジニア、インテグレーター、愛好家を対象としています。

---

## 必要な事前知識

コントリビューションを始める前に、以下の知識・経験があると望ましいです：

- GitHubの基本的な使い方
- Gitコマンドラインの操作
- Go言語の基礎（Goのインストールは不要、開発環境が提供します）
- Dockerの基本的な知識

---

## 開発環境のセットアップ

### 1. 必要なソフトウェア

以下のソフトウェアが必要です：

- **GitHubアカウント**: [https://github.com](https://github.com)で無料アカウントを作成
- **git**: バージョン管理システム
  ```bash
  git --version  # インストール確認
  ```
- **make**: ビルドツール
  ```bash
  make -v  # インストール確認
  ```
- **Docker**: コンテナランタイム
  ```bash
  docker --version  # インストール確認
  ```

> **注意**: Go言語のインストールは不要です。Mobyの開発環境が提供します。

### 2. リポジトリのフォークとクローン

#### 2.1 GitHubでリポジトリをフォーク

1. ブラウザで [https://github.com/moby/moby](https://github.com/moby/moby) にアクセス
2. 右上の「Fork」ボタンをクリック
3. あなたのアカウントに `YOUR_ACCOUNT/moby` がフォークされます

#### 2.2 ローカルにクローン

```bash
# ホームディレクトリに移動
cd ~

# reposディレクトリを作成
mkdir repos
cd repos

# フォークをクローン（YOUR_ACCOUNTを自分のユーザー名に置き換える）
git clone https://github.com/YOUR_ACCOUNT/moby.git moby-fork
cd moby-fork
```

### 3. Gitの設定

#### 3.1 ユーザー情報の設定

```bash
# リポジトリのルートに移動
cd moby-fork

# 名前を設定
git config --local user.name "FirstName LastName"

# メールアドレスを設定
git config --local user.email "your.email@example.com"
```

#### 3.2 upstreamリモートの追加

```bash
# 元のmoby/mobyリポジトリをupstreamとして追加
git remote add upstream https://github.com/moby/moby.git

# 設定確認
git remote -v
# 以下のように表示されるはずです：
# origin    https://github.com/YOUR_ACCOUNT/moby.git (fetch)
# origin    https://github.com/YOUR_ACCOUNT/moby.git (push)
# upstream  https://github.com/moby/moby.git (fetch)
# upstream  https://github.com/moby/moby.git (push)
```

---

## コントリビューションの流れ

### 1. Issue の確認

変更を始める前に、[Issue一覧](https://github.com/moby/moby/issues)で同じ問題や提案が既に報告されていないか確認してください。

### 2. ブランチの作成

作業内容に応じてブランチ名を決定します：

- **バグ修正**: `XXXX-bug-description` （XXXXはIssue番号）
- **新機能**: `XXXX-feature-description` （XXXXはIssue番号）

```bash
# 新しいブランチを作成して切り替え
git checkout -b 1234-fix-authentication-bug

# ブランチの確認
git branch
```

### 3. コードの変更

- `vendor`ディレクトリ以外の任意のGoパッケージを変更可能
- 新しいパッケージを追加する場合は、まず`internal`ディレクトリに配置することを検討

#### ディレクトリ構造ガイドライン

- `api` - クライアントとデーモンで共有される型とSwagger定義
- `client` - Dockerクライアントの全Goファイル
- `contrib` - 外部ツールやライブラリ関連のファイル
- `daemon` - デーモン構築用の全Goファイルとパッケージ
- `docs` - Markdownを使用した全技術ドキュメント
- `hack` - テスト、開発、CI用のスクリプト
- `integration` - API、クライアント、デーモンの統合テスト
- `pkg` - 外部で使用されるレガシーGoパッケージ（新規追加不可）
- `project` - Mobyプロジェクトのガバナンス関連ファイル

### 4. 変更のコミット

#### コミットメッセージの規約

- 最初の行: 大文字で始まる50文字以内の要約（命令形）
- 空行
- 詳細な説明（オプション）

```bash
# 変更をステージング
git add path/to/changed/file.go

# サインオフ付きでコミット（-sオプションが自動でサインオフを追加）
git commit -s -m "Fix authentication timeout issue

This commit resolves the timeout issue in the authentication
flow by increasing the default timeout value from 5s to 30s.

Fixes #1234"
```

**重要**: すべてのコミットには `Signed-off-by` が必要です。これは[Developer Certificate of Origin](http://developercertificate.org/)に同意することを示します。

### 5. フォークへのプッシュ

```bash
# 初回プッシュ時
git push --set-upstream origin 1234-fix-authentication-bug

# 2回目以降
git push
```

---

## Issue報告

### セキュリティ問題の報告

セキュリティに関する問題を発見した場合：

- **公開Issueは作成しないでください**
- [security@docker.com](mailto:security@docker.com) に直接メールで報告

### 一般的な問題の報告

バグや提案を報告する際は、以下の情報を含めてください：

```bash
# Dockerのバージョン
docker version

# Dockerの情報
docker info
```

また、以下も含めると良いです：

- 問題を再現する手順
- 期待される動作
- 実際の動作
- ログファイル（機密情報は削除）

---

## Pull Requestの作成

### 1. Pull Requestの準備

#### テストの実行

変更後は必ずテストを実行してください：

```bash
# テストの実行（詳細はTESTING.mdを参照）
make test
```

#### コードのフォーマット

```bash
# gofmtでコードをフォーマット
gofmt -s -w path/to/changed/file.go
```

#### コミットの整理

複数のコミットがある場合、論理的な単位にまとめます：

```bash
# インタラクティブリベースで複数コミットをまとめる
git rebase -i master

# 強制プッシュ（注意して実行）
git push -f
```

### 2. Pull Requestの作成

1. GitHubのフォークページにアクセス
2. 「Pull request」ボタンをクリック
3. ベースブランチ: `moby/moby:master`
4. 比較ブランチ: `YOUR_ACCOUNT/moby:your-branch-name`
5. タイトルと説明を記入
   - タイトル: 変更内容の簡潔な要約
   - 説明: 変更の詳細、理由、関連Issue（`Fixes #1234`など）

### 3. レビュープロセス

- メンテナーからコードレビューコメントが付きます
- 修正が必要な場合は、同じブランチに追加コミットしてプッシュ
- コメント後に通知されるため、コメントを投稿してください
- 承認されると「LGTM（Looks Good To Me）」とコメントされます

### 4. マージの承認基準

- テストがすべてパス
- コーディング規約に準拠
- ドキュメントが更新されている（該当する場合）
- コミットメッセージが適切
- Developer Certificate of Originにサインオフ

---

## コーディング規約

Mobyプロジェクトは、Goコミュニティのコーディングガイドラインに従います。

### 基本ルール

1. **フォーマット**: すべてのコードは `gofmt -s` でフォーマット
2. **Lint**: `golint` のデフォルトレベルをパス
3. **スタイル**: [Effective Go](https://go.dev/doc/effective_go)と[Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)に従う
4. **コメント**: コードにコメントを追加（理由、履歴、コンテキストを説明）
5. **ドキュメント**: プライベートなものも含めて、すべての宣言とメソッドをドキュメント化
6. **変数名**: コンテキストに応じた長さ（短いメソッドは短い変数名、グローバルは長い名前）
7. **パッケージ名**: アンダースコアを使用しない
8. **utilsパッケージ禁止**: 一般的でない関数はエクスポートせず、ドキュメント化する
9. **テスト**: すべてのテストは `go test` で実行可能

### 例外

これらは「ルール」ですが、実際にはガイドラインです。状況に応じて適切に判断してください。

---

## テスト

### テストの実行

```bash
# 全テストの実行
make test

# 特定のパッケージのテスト
go test ./path/to/package

# 統合テストの実行
make test-integration
```

### テストの作成

- 変更には必ずテストを追加
- 統合テストが必要な場合は、APIに対して記述
- テストは `go test` で実行可能であること

詳細は [TESTING.md](https://github.com/moby/moby/blob/master/TESTING.md) を参照してください。

---

## コミュニティガイドライン

### 基本原則

- **親切に**: 礼儀正しく、敬意を持って、丁寧に接する
- **多様性を奨励**: 全員を歓迎し、参加を奨励
- **合法的に**: 自分が所有するコンテンツのみを共有
- **トピックに沿う**: 適切なチャンネルに投稿
- **メンテナーへの直接メール禁止**: GitHubのメンションを使用

### コミュニティリソース

#### フォーラム

[https://forums.mobyproject.org](https://forums.mobyproject.org)
- GitHubアカウントでログイン可能
- 質問やベストプラクティスの議論

#### Slack

[https://dockr.ly/comm-slack](https://dockr.ly/comm-slack)
- `#moby-project` チャンネルで一般的な議論
- 他のMobyプロジェクト用の個別チャンネルあり（例: `#containerd`）

#### Twitter

[@moby](https://twitter.com/moby/)
- 製品のアップデートを取得
- 質問やストーリーの共有

### ガイドライン違反（3ストライク制）

1. **初回**: 公開で友好的な注意
2. **2回目**: プライベートメッセージで警告
3. **3回目**: アカウントの削除またはバン

**注意点**:
- 明らかなスパムは初回でバン
- 6ヶ月間の良好な行動で違反は許される
- 軽微な違反は教育的対応
- 脅迫的、虐待的、破壊的、違法な行為は即座に対処

---

## 大規模な変更の提案

### デザイン提案

大規模な機能追加やリファクタリングの場合：

1. まず問題を述べるIssueを作成
2. 解決策を提案し、代替案をリストアップ
3. デザインドキュメントを作成（大規模変更の場合）
4. メンテナーと調整してからPull Requestを作成

**重要**: 大規模なPRを事前の連絡なしに提出すると、受け入れられない可能性が高いです。

---

## メンテナーになるには

メンテナーになるための手順は [/project/GOVERNANCE.md](https://github.com/moby/moby/blob/master/project/GOVERNANCE.md) に記載されています。

メンテナーは時間の投資が必要です。メンテナーにならなくても、プロジェクトに大きな違いをもたらすことができます。

---

## 参考リンク

- **メインドキュメント**: [CONTRIBUTING.md](https://github.com/moby/moby/blob/master/CONTRIBUTING.md)
- **開発環境セットアップ**: [docs/contributing/](https://github.com/moby/moby/tree/master/docs/contributing)
- **テストガイド**: [TESTING.md](https://github.com/moby/moby/blob/master/TESTING.md)
- **ガバナンス**: [project/GOVERNANCE.md](https://github.com/moby/moby/blob/master/project/GOVERNANCE.md)
- **レビュープロセス**: [project/REVIEWING.md](https://github.com/moby/moby/blob/master/project/REVIEWING.md)
- **Developer Certificate of Origin**: [developercertificate.org](http://developercertificate.org/)

---

## クイックスタートチェックリスト

コントリビューションを始める前に、以下を確認してください：

- [ ] GitHubアカウントを作成済み
- [ ] git, make, Dockerをインストール済み
- [ ] moby/mobyリポジトリをフォーク済み
- [ ] ローカルにクローン済み
- [ ] Gitのユーザー名とメールを設定済み
- [ ] upstreamリモートを追加済み
- [ ] 既存のIssueを確認済み
- [ ] 作業用ブランチを作成済み
- [ ] Developer Certificate of Originを理解済み
- [ ] コミットメッセージにサインオフを含める準備ができている

---

このガイドが、Mobyプロジェクトへのコントリビューションの助けになれば幸いです！

質問やサポートが必要な場合は、[Moby Project Forums](https://forums.mobyproject.org)や[Slack](https://dockr.ly/comm-slack)で気軽に尋ねてください。

Happy Contributing! 🎉
