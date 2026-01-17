---
title: "Claude Code Skillsで実装からレビューまで全部自動化してみた"
emoji: "🔧"
type: "tech"
topics: ["claudecode", "github", "githubactions", "ai", "codereview"]
published: false
---

## はじめに

「実装したらそのままレビューもAIに任せたい」

開発してると、コード書く→PR作る→レビュー待ち→修正→また待ち...のサイクルが地味にストレス。特に一人開発やスタートアップだと、レビュアーがいない or 忙しいことも多い。

そこで試したのが**Claude Code Skills + GitHub Actions**の組み合わせ。結論から言うと、実装からレビュー、修正までほぼ自動化できました。

この記事では、実際にセットアップして使ってみた体験を共有します。

## 使うもの

今回使うのは3つ。

### 1. Claude Code Skills

2025年10月にAnthropicから正式発表された機能。`.claude/skills/` にマークダウンファイルを置くだけで、Claudeに「こういう時はこうして」というルールを教えられます。

コードレビュー用のSkillを作れば、レビュー時に必ずチェックしてほしいポイントを指定できる。

### 2. Claude Code Action（GitHub Actions）

GitHubのPRで`@claude`とメンションすると、Claude Codeがレビューしてくれる。PR作成時に自動でレビューを走らせることも可能。

公式リポジトリ: https://github.com/anthropics/claude-code-action

### 3. /review コマンド

Claude Codeのスラッシュコマンド。ローカルで `gh` CLI経由でPRの差分を取得して、その場でレビューしてくれる。

## Skills のセットアップ

まずはコードレビュー用のSkillを作ります。

### ディレクトリ構造

```
.claude/skills/
└── code-review/
    ├── SKILL.md
    └── examples.md
```

### SKILL.md

```yaml
---
name: code-review
description: コードレビューを行う時に使用するスキル。
  「レビューして」「このコード見て」「PRチェックして」などの
  リクエストがあった場合に自動で適用される。
---

## レビュー観点

以下の観点でコードをレビューしてください：

### 1. セキュリティ
- SQLインジェクション、XSSの可能性
- 認証・認可の漏れ
- 機密情報のハードコード

### 2. パフォーマンス
- N+1クエリ
- 不要なループ
- メモリリーク

### 3. 可読性
- 変数名・関数名は意図が伝わるか
- 複雑すぎるロジックはないか
- コメントは適切か

### 4. テスト
- テストは書かれているか
- エッジケースはカバーされているか

## 出力フォーマット

レビュー結果は以下の形式で出力してください：

**Good**
- 良い点を箇条書きで

**Needs Improvement**
- 改善点を箇条書きで
- 具体的な修正案も添えて

**Questions**
- 確認したい点があれば
```

### ポイント：description が超重要

Skillsが呼ばれるかどうかは、**descriptionの書き方で9割決まる**。

Claudeはdescriptionを見て「このスキルを使うべきか」を判断するので、曖昧な記述だと呼び出されません。

```yaml
# ダメな例
description: コードレビュー用

# 良い例
description: コードレビューを行う時に使用するスキル。
  「レビューして」「このコード見て」「PRチェックして」などの
  リクエストがあった場合に自動で適用される。
```

トリガーワードを具体的に書くのがコツ。

## GitHub Actions の設定

次にClaude Code Actionを設定します。

### 1. GitHub Appのインストール

一番簡単なのは、Claude Codeのターミナルで以下を実行：

```bash
/install-github-app
```

手動でやる場合は https://github.com/apps/claude にアクセスしてインストール。

### 2. シークレットの設定

リポジトリの Settings → Secrets and variables → Actions から `ANTHROPIC_API_KEY` を追加。

### 3. ワークフローファイルの作成

`.github/workflows/claude.yml` を作成：

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  pull_request:
    types: [opened, synchronize]

jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request')
    runs-on: ubuntu-latest

    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write

    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 1

      - name: Run Claude Code
        uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

これで、PR作成時に自動レビュー + `@claude`メンションで追加対応ができるようになります。

## 実際のワークフロー

セットアップが終わったら、実際の開発フローはこうなります。

### Step 1: Claude Codeで実装

```bash
claude
> 新しいAPIエンドポイント POST /api/tasks を実装して。
> バリデーションとエラーハンドリングも含めて。
```

Skillsを設定しておくと、実装時から品質が担保される。

### Step 2: PR作成 → 自動レビュー

```bash
git checkout -b feature/add-tasks-api
git add .
git commit -m "feat: add tasks API endpoint"
git push -u origin feature/add-tasks-api
gh pr create --fill
```

PR作成後、3〜4分でClaude Code Actionがレビューコメントを投稿してくれます。

### Step 3: @claude で修正依頼

レビューで指摘があったら、PRのコメント欄で：

```
@claude セキュリティの指摘を修正して
```

Claudeが修正コミットを作成してプッシュしてくれます。

### Step 4: /review でローカル確認

マージ前に手元で最終確認したい場合：

```bash
claude
> /review 123
```

PRナンバーを指定すると、`gh pr diff` で差分を取得してレビューしてくれます。

## 辛かったポイント

正直に書きます。ハマりどころはありました。

### 1. description地獄

最初、Skillsが全然呼ばれなくて困った。原因はdescriptionが曖昧だったから。

「コードレビュー」とだけ書いてもダメで、「レビューして」「このコード見て」などトリガーワードを明示的に書く必要がある。

### 2. 認証ループ

GitHub Appの認証で無限ループにハマることがある。解決策は：

1. ChatGPTの設定 → GitHub接続を解除
2. もう一度接続し直す

地味に時間を溶かした。

### 3. レビュー待ち時間

Claude Code Actionのレビューは3〜4分かかる。すぐ結果が欲しい時はちょっともどかしい。

ローカルで `/review` を使えば即座に結果が返ってくるので、急ぎの時はそっちを使うようになった。

### 4. 必要なプラン

Claude Code SkillsはPro/Max/Team/Enterpriseプランが必要。月$20〜かかる。

GitHub Actions側はAnthropic APIの従量課金。レビュー1回あたり数セント程度だけど、PR数が多いと積もる。

## よかったポイント

辛いところを乗り越えたら、かなり快適になりました。

### レビュー時間が激減

人間のレビュアーを待つ時間がほぼゼロに。PR作成から3分でフィードバックが来る。

### 一貫した品質

Skillsでレビュー観点を定義しておけば、毎回同じ基準でチェックされる。人によるバラつきがない。

### 修正まで自動

`@claude 直して` で修正コミットまで作ってくれるのが地味に便利。指摘を見て手動で直す手間が省ける。

### ローカル確認が楽

`/review` コマンドでPRを作る前にセルフレビューできる。「これPRに出していいかな？」の判断が楽になった。

## 実際のコード

### カスタム /review コマンド

`.claude/commands/review.md` を作成：

```markdown
PR #$ARGUMENTS をレビューしてください。

以下の手順で実行してください：
1. `gh pr view $ARGUMENTS` でPRの概要を確認
2. `gh pr diff $ARGUMENTS` で差分を取得
3. code-review スキルを使ってレビュー
4. 結果を出力

特に以下の観点を重視してください：
- セキュリティリスク
- パフォーマンス問題
- テストの網羅性
```

これで `/review 123` のように使えます。

## 学び・Tips

### Skillsは小さく始める

最初から完璧なSkillを作ろうとしない。まず最低限の観点だけ書いて、使いながら育てていくのがおすすめ。

### SKILL.mdは500行以下に

長すぎるとコンテキストを圧迫する。詳細は別ファイル（examples.md など）に分けて、SKILL.mdはメニュー的な役割にする。

### /compact を活用

長いセッションになったら `/compact` でコンテキストを圧縮。レビューの品質が安定する。

### 公式ドキュメント

- [Claude Code Skills 公式ドキュメント](https://code.claude.com/docs/ja/skills)
- [Claude Code Action リポジトリ](https://github.com/anthropics/claude-code-action)
- [スラッシュコマンド 公式ドキュメント](https://code.claude.com/docs/ja/slash-commands)

## まとめ

Claude Code Skills + GitHub Actionsの組み合わせで、実装からレビューまでほぼ自動化できました。

### 向いている人

- 一人開発 or 小規模チームでレビュアーが足りない
- レビュー待ちの時間を削減したい
- 一貫したレビュー品質を担保したい

### 向いていない人

- 無料で使いたい（Pro/Max以上が必要）
- 即座にレビュー結果が欲しい（3-4分かかる）
- 人間のレビューを重視したい

個人的には、一度セットアップしてしまえばあとは楽。レビュー待ちのストレスがなくなったのが一番大きい。

これからAIにレビューを任せたい人は、まずSkillsから試してみるのがおすすめです。
