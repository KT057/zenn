---
title: "Claude Code 2026年1月アップデートまとめ｜PRリンクとタスク管理が地味に便利だった"
emoji: "🚀"
type: "tech"
topics: ["claudecode", "github", "ai", "cli", "開発効率化"]
published: false
---

## はじめに

Claude Code使ってる人、アップデートちゃんと追ってますか？

正直、自分も「動いてるからいいか」と放置しがちだったんですが、久しぶりに`claude update`を実行したら、地味に便利な機能が増えていて驚きました。特に**PRとセッションのリンク機能**と**タスク管理の改善**は、日々の開発で「あ、これ欲しかった」と思える機能でした。

この記事では、2026年1月のアップデート内容をまとめつつ、実際に使ってみて「これは便利」「ここは微妙」と感じたポイントを共有します。

まずはアップデートから。ターミナルで以下を実行するだけです。

```bash
claude update
```

## 1月アップデート全体像

主要なアップデートを時系列で整理しました。

| 日付 | バージョン | 主要変更 |
|------|-----------|----------|
| 1/31 | [v2.1.29](https://github.com/anthropics/claude-code/releases/tag/v2.1.29) | セッション再開時のパフォーマンス改善 |
| 1/30 | [v2.1.27](https://github.com/anthropics/claude-code/releases/tag/v2.1.27) | PRセッションリンク機能（`--from-pr`） |
| 1/28 | [v2.1.21](https://github.com/anthropics/claude-code/releases/tag/v2.1.21) | 日本語IME（全角数字）対応 |
| 1/27 | [v2.1.20](https://github.com/anthropics/claude-code/releases/tag/v2.1.20) | `--add-dir`対応、PRレビュー状態表示 |
| 1/23 | [v2.1.19](https://github.com/anthropics/claude-code/releases/tag/v2.1.19) | タスク管理システム（依存関係対応） |
| 1/14 | [v2.1.7](https://github.com/anthropics/claude-code/releases/tag/v2.1.7) | 権限プロンプト改善、Windows修正 |

派手な新機能はないものの、開発ワークフローの細かい部分が改善されています。個人的には「痒いところに手が届く」系のアップデートが多かった印象です。

## 注目機能1: PRセッションリンク（--from-pr）

### 概要

PRで作業したセッションを、後から再開できる機能です。[v2.1.27](https://github.com/anthropics/claude-code/releases/tag/v2.1.27)で追加されました。

これまでは「3日前にこのPR作ったけど、どんな文脈で実装したっけ...」という状況で、一から説明し直す必要がありました。`--from-pr`を使えば、当時のセッションを引き継いで作業を再開できます。

### 使い方

```bash
# PR番号を指定して作業再開
claude --from-pr 123

# URLでも指定可能
claude --from-pr https://github.com/owner/repo/pull/123

# PR作成時に自動でセッションをリンク（gh CLI経由）
gh pr create
```

gh CLIでPRを作成すると、自動的に現在のセッションがリンクされます。後から`--from-pr`で再開すれば、「このPRで何をやったか」という文脈が保持された状態で作業できます。

### ユースケース

**レビュー指摘への対応**
PRにレビューコメントがついた時、当時の実装意図を思い出しながら修正できます。「なぜこの実装にしたんだっけ」と考える時間が減ります。

**長期間放置したPRの再開**
数週間前に作ったPRを再開する時、セッション履歴があれば「ここまでやった」「次はこれをやる予定だった」がすぐ分かります。

**PRレビュー状態の確認**
[v2.1.20](https://github.com/anthropics/claude-code/releases/tag/v2.1.20)で追加されたPRレビュー状態インジケーターにより、セッション再開時に以下のステータスが表示されます。
- ✅ Approved（承認）
- 📝 Changes requested（変更リクエスト）
- ⏳ Pending review（レビュー保留中）
- 📄 Draft（ドラフト）

## 注目機能2: タスク管理システム改善

### 概要

Claude Code内でタスクを管理できる機能が強化され、**タスク間の依存関係**を設定できるようになりました。[v2.1.19](https://github.com/anthropics/claude-code/releases/tag/v2.1.19)で追加されています。

複数のファイルを変更する大きなPRで、「どのタスクから手をつけるべきか」「並列で進められるタスクはどれか」が可視化されます。

### 使い方

Claude Codeのセッション内で、タスクをリストアップするよう依頼します。

```
> このPRで必要なタスクをリストアップして

# Claude Codeの出力例:
タスクリスト:
1. [x] 型定義ファイルの追加
2. [ ] APIエンドポイントの実装（1に依存）
3. [ ] フロントエンドの接続（2に依存）
4. [ ] テストの追加（2, 3に依存）

次に取り組むべき: タスク2（型定義は完了済み）
```

依存関係が可視化されるので、「タスク2が終わらないとタスク3,4は始められない」といった状況が一目で分かります。

[v2.1.20](https://github.com/anthropics/claude-code/releases/tag/v2.1.20)ではタスク削除機能も追加され、不要になったタスクを整理できるようになりました。

### ユースケース

**複数ファイルにまたがるリファクタリング**
「まず型を変更」→「次に関数を更新」→「最後にテストを修正」という順序を明確にできます。

**大きな機能実装の分解**
「認証機能を追加して」という大きなタスクを、実装可能な単位に分解して進捗管理できます。

**並列作業の特定**
依存関係がないタスクは並列で進められます。一人で作業する場合も、コンテキストスイッチのタイミングを計画できます。

## 注目機能3: CLAUDE.md対応拡張（--add-dir）

### 概要

別ディレクトリの`CLAUDE.md`を追加で読み込めるようになりました。[v2.1.20](https://github.com/anthropics/claude-code/releases/tag/v2.1.20)で追加された機能です。

モノレポ構成や、複数プロジェクトで共通のルールを使いたい場合に便利です。

### 使い方

```bash
# 共通設定を持つ別ディレクトリを追加
claude --add-dir ../shared-config

# モノレポで共通ルールを参照
claude --add-dir packages/common

# 複数ディレクトリを追加することも可能
claude --add-dir ../design-system --add-dir ../api-specs
```

### ユースケース

**モノレポでの共通コーディング規約**
`packages/common/CLAUDE.md`に共通ルールを書いておき、各パッケージで参照できます。

**複数プロジェクト間での設定共有**
チーム共通のルールを別リポジトリで管理し、各プロジェクトから参照するパターン。

**外部リソースの参照**
API仕様書やデザインシステムのドキュメントがある別ディレクトリを参照して、一貫性のある実装ができます。

## その他の改善点

### UI/UX

- **日本語IME対応**: 全角数字の入力がサポートされました（[v2.1.21](https://github.com/anthropics/claude-code/releases/tag/v2.1.21)）
- **スピナーカスタマイズ**: `spinnerVerbs`設定で処理中メッセージをカスタマイズ可能に（[v2.1.23](https://github.com/anthropics/claude-code/releases/tag/v2.1.23)）
- **ワイド文字レンダリング**: 絵文字やCJK文字の表示が改善（[v2.1.20](https://github.com/anthropics/claude-code/releases/tag/v2.1.20)）
- **showTurnDuration設定**: 「Cooked for 1m 6s」メッセージを非表示に（[v2.1.7](https://github.com/anthropics/claude-code/releases/tag/v2.1.7)）

### パフォーマンス

- **セッション再開改善**: `saved_hook_context`関連のスタートアップ問題を修正（[v2.1.29](https://github.com/anthropics/claude-code/releases/tag/v2.1.29)）
- **Bashコマンド実行改善**: Windows環境での`.bashrc`対応（[v2.1.27](https://github.com/anthropics/claude-code/releases/tag/v2.1.27)）

### 企業利用向け

- **mTLS対応**: 相互TLS認証を必要とするプロキシ環境での接続をサポート（[v2.1.23](https://github.com/anthropics/claude-code/releases/tag/v2.1.23)）
- **Bedrock/Vertex修正**: ゲートウェイユーザー向けの検証エラー修正（[v2.1.25](https://github.com/anthropics/claude-code/releases/tag/v2.1.25)）

### 安定性

- **プロンプトキャッシング修正**: スコープ有効時の400エラー修正（[v2.1.23](https://github.com/anthropics/claude-code/releases/tag/v2.1.23)）
- **メモリ不足クラッシュ修正**: セッション再開時の問題対応（[v2.1.16](https://github.com/anthropics/claude-code/releases/tag/v2.1.16)）
- **AVXなしプロセッサ対応**: 古いCPUでのクラッシュ修正（[v2.1.17](https://github.com/anthropics/claude-code/releases/tag/v2.1.17)）

## 辛かったポイント

### --from-prでセッションが見つからない

`--from-pr`を使おうとして「リンクされたセッションが見つかりません」と言われることがあります。

**原因**: GitHub Web UIでPRを作成した場合、セッションは自動リンクされません。**gh CLIで作成したPRのみ自動リンク対象**です。

**対策**: PRを作る時は`gh pr create`を使う習慣をつける。既存のPRについては、Claude Codeでそのブランチにチェックアウトして作業すれば、以降のセッションはリンクされます。

### タスクの依存関係が複雑になりすぎる

大きなPRでタスクをリストアップすると、依存関係が複雑になりすぎて逆に把握しづらくなることがあります。

**対策**: 結局、PRを小さく分割する方が楽です。「1PR = 1つの明確な目的」を意識すると、タスク管理もシンプルになります。

## よかったポイント

### PRの文脈保持が快適

一番良かったのは、PRの文脈を保ったまま作業再開できること。「このPRで何をやったか」を毎回説明し直す手間がなくなりました。

特にレビュー対応で効果を感じます。「なぜこの実装にしたか」という経緯がセッションに残っているので、レビュアーの指摘に対して「あ、それは〇〇の理由でこうしたんだった」とすぐ思い出せます。

### 「次何やる？」が明確に

タスクの依存関係が可視化されることで、「次に何をすべきか」が明確になりました。

大きめの実装で「どこから手をつけよう...」と迷う時間が減り、作業の見通しが立てやすくなっています。

### モノレポでの開発が楽に

`--add-dir`は、モノレポで開発している人には嬉しい機能です。

これまでは各パッケージの`CLAUDE.md`に共通ルールをコピペしていましたが、共通部分を一箇所で管理できるようになりました。

### 日本語ユーザーへの配慮

[v2.1.21](https://github.com/anthropics/claude-code/releases/tag/v2.1.21)での日本語IME対応は地味ですが嬉しい。全角数字を入力してしまっても正しく処理されます。

## まとめ

### 向いている人

- GitHub PRベースで開発している
- 複数タスクを並行して進めることが多い
- モノレポや複数プロジェクトを扱っている
- レビュー対応で「当時の文脈」を忘れがち

### 向いていない人

- GitHubを使っていない（GitLabなどは未対応）
- 小規模な単発タスクが多い
- すでに別のタスク管理ツールでワークフローが確立している

### 最後に

派手な機能追加ではないですが、日々の開発で「地味に便利」と感じるアップデートでした。特にPRセッションリンクは、使い始めると手放せなくなります。

まずは`claude update`で最新版にして、`--from-pr`を試してみてください。

## 参考リンク

- [Claude Code Releases - GitHub](https://github.com/anthropics/claude-code/releases)
- [Claude Code公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [Claude Code CHANGELOG](https://docs.claude.com/ja/release-notes/claude-code)
