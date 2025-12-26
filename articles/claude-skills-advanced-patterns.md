---
title: "Claude Agent Skills 設計パターン集｜MCP連携・Subagent使い分け・Progressive Disclosure"
emoji: "🏗️"
type: "tech"
topics: ["claudecode", "agentskills", "mcp", "ai", "architecture"]
published: false
---

## はじめに

Agent Skillsの基本的な使い方は理解した。でも、実際にチームで運用し始めると、こんな疑問が出てきませんか？

- 「SkillとSubagent、どっち使うのが正解？」
- 「MCPサーバーとSkillsって、どう組み合わせるの？」
- 「複数のSkillが同時に必要なケースってどう設計する？」

この記事では、Agent Skillsを本番運用する上での設計パターンと、よくあるアーキテクチャの選択について共有します。基本的なSkill作成方法は知っている前提で進めます。

## Progressive Disclosure アーキテクチャを理解する

Agent Skillsの設計思想の核心は **Progressive Disclosure（段階的開示）** です。これを理解すると、なぜSkillsがスケールするのかがわかります。

### 3層構造

Anthropicは3層の情報開示アーキテクチャを設計しました。

```
┌─────────────────────────────────────────────────────────┐
│ Level 1: メタデータ（常時ロード）                        │
│ - スキル名 + description                                │
│ - 約100トークン/スキル                                   │
│ → Claudeが「何が使えるか」を知るためのインデックス        │
└─────────────────────────────────────────────────────────┘
                          ↓ 必要なときだけ
┌─────────────────────────────────────────────────────────┐
│ Level 2: コア指示（オンデマンド）                        │
│ - SKILL.md の本文                                       │
│ - 5,000トークン未満推奨                                  │
│ → タスクに関連するときだけロード                         │
└─────────────────────────────────────────────────────────┘
                          ↓ さらに必要なときだけ
┌─────────────────────────────────────────────────────────┐
│ Level 3+: ネストされたリソース（必要最小限）              │
│ - reference/, templates/, scripts/                      │
│ - 実質無制限                                            │
│ → Claudeがbashで明示的に読み込む                         │
└─────────────────────────────────────────────────────────┘
```

### なぜこれが重要か

10個のSkillをインストールしても、起動時に消費するのは約1,000トークン。実際に使うときだけ詳細がロードされる。

これ、RAGに似てるけど違います。RAGは「検索して関連文書を取得」するのに対し、Skillsは「ファイルシステムを自分でナビゲートして必要なものを読む」。Claudeが能動的に情報を取りに行く設計です。

### 実装例：ドキュメントバンドル型Skill

大量のリファレンスを持つSkillの設計例。

```
.claude/skills/api-integration/
├── SKILL.md
└── reference/
    ├── endpoints.md      # APIエンドポイント一覧
    ├── auth.md           # 認証フロー
    ├── error-codes.md    # エラーコード表
    └── rate-limits.md    # レート制限
```

`SKILL.md`:

```markdown
---
name: api-integration
description: 外部APIとの連携を実装します。API呼び出し、認証、エラーハンドリングのときに使用します。
---

# API連携ガイド

## 概要

このスキルは外部API連携の実装をサポートします。

## 参照ドキュメント

詳細情報は以下のファイルを参照してください：

- `reference/endpoints.md` - 利用可能なエンドポイント一覧
- `reference/auth.md` - 認証フロー（OAuth2, API Key）
- `reference/error-codes.md` - エラーコードと対処法
- `reference/rate-limits.md` - レート制限とリトライ戦略

必要に応じて上記ファイルを読み込んで作業してください。
```

ポイントは「参照してください」と書くこと。Claudeは必要なときだけ`cat reference/endpoints.md`などで読み込みます。全部を最初から読まない。

## Skills + MCP の統合パターン

MCPとSkillsは補完関係にあります。

- **MCP**: 外部システムへの「接続」を提供（何ができるか）
- **Skills**: その接続を「どう使うか」の知識を提供

### パターン1：MCP操作の標準化

GitHubのMCPサーバーを使うとき、チームごとに操作方法がバラバラになりがち。Skillで標準化します。

```markdown
---
name: github-workflow
description: GitHubでのPR作成、レビュー、マージのワークフローを実行します。PRを作る、レビューする、マージするときに使用します。
---

# GitHub ワークフロー

## 前提

このスキルはGitHub MCPサーバーが接続されていることを前提とします。

## PR作成手順

1. ブランチ名は `feature/`, `fix/`, `chore/` で始める
2. コミットメッセージは Conventional Commits 形式
3. PR説明には以下を含める：
   - ## 概要
   - ## 変更内容
   - ## テスト方法
   - ## レビュー観点

## コードレビュー手順

1. CIが通っていることを確認
2. 変更ファイルを一つずつ確認
3. コメントは以下の接頭辞を使用：
   - `[must]` - 修正必須
   - `[should]` - 修正推奨
   - `[nit]` - 軽微、任意

## マージポリシー

- Squash merge を使用
- マージ後はブランチを削除
```

### パターン2：複数MCPの協調

1つのSkillで複数のMCPサーバーを協調させる例。

```markdown
---
name: competitive-analysis
description: 競合分析レポートを作成します。競合調査、市場分析、ベンチマーキングのときに使用します。
---

# 競合分析

## 使用するMCPサーバー

- Google Drive: 社内の過去調査資料
- GitHub: 競合のオープンソースプロジェクト
- Web Search: 最新のプレスリリースや記事

## 分析フロー

1. **社内資料の確認**（Google Drive）
   - 過去の競合分析レポートを検索
   - 既存の知見を把握

2. **技術分析**（GitHub）
   - 競合のパブリックリポジトリを調査
   - スター数、コミット頻度、技術スタックを確認

3. **最新動向**（Web Search）
   - 直近3ヶ月のプレスリリースを検索
   - 資金調達、製品発表、パートナーシップを確認

4. **レポート作成**
   - 上記の情報を統合
   - SWOT分析形式でまとめる
```

## Skills vs Subagents：判断フロー

これ、最初かなり迷いました。結論から言うと：

- **Skills** = レシピ（手順書）
- **Subagents** = 専門家（独立したワーカー）

### 判断フローチャート

```
タスクを依頼されたとき
    │
    ├─ 手順を知っていれば自分でできる？
    │     │
    │     ├─ Yes → Skill
    │     │
    │     └─ No（探索・試行錯誤が必要）→ Subagent
    │
    └─ コンテキストを共有したい？
          │
          ├─ Yes（対話しながら進めたい）→ Skill
          │
          └─ No（結果だけ欲しい）→ Subagent
```

### 具体例

| タスク | 選択 | 理由 |
|--------|------|------|
| コードレビュー | Skill | 手順が決まっている、結果を見ながら議論したい |
| バグ調査 | Subagent | 試行錯誤が必要、詳細なログを本体に持ち込みたくない |
| テスト作成 | Skill | TDDサイクルで文脈を保持したい |
| 大規模リファクタリング | 両方 | Skillで方針を定義、Subagentで各ファイルを処理 |
| ドキュメント検索 | Subagent | 大量のファイルを読む、結果だけ欲しい |

### ハイブリッドパターン

Subagentの中でSkillを使う設計。

```markdown
---
name: refactoring-guide
description: リファクタリングの方針とパターンを定義します。リファクタ、コード改善、技術的負債解消のときに使用します。
---

# リファクタリングガイド

## 原則

- 動作を変えずに構造を改善する
- テストが通る状態を維持する
- 小さな変更を積み重ねる

## パターン

### Extract Method
長いメソッドを分割するとき...

### Replace Conditional with Polymorphism
複雑な条件分岐を整理するとき...
```

このSkillを定義しておくと、リファクタリング用のSubagentを起動したときに、Subagent内でも同じ方針が適用されます。Skillsはグローバルに利用可能なので、Subagentにも継承される。

## チーム運用のベストプラクティス

### ディレクトリ構成

大規模チーム向けの構成例。

```
.claude/
├── skills/
│   ├── shared/              # 全チーム共通
│   │   ├── code-review/
│   │   ├── testing/
│   │   └── documentation/
│   │
│   ├── frontend/            # フロントエンドチーム
│   │   ├── react-patterns/
│   │   └── a11y-check/
│   │
│   ├── backend/             # バックエンドチーム
│   │   ├── api-design/
│   │   └── database/
│   │
│   └── devops/              # DevOpsチーム
│       ├── ci-cd/
│       └── monitoring/
```

### 命名規則

```
{domain}-{action}

例：
- frontend-component-generator
- backend-api-validator
- devops-deploy-checker
```

### description の書き方テンプレート

発動条件を明確にするテンプレート。

```markdown
description: {何をするか}。{どんなときに使うか}。「{トリガーとなるキーワード}」と言われたときに起動します。
```

例：

```markdown
description: TypeScriptの型定義を生成します。インターフェース作成、型安全性向上のときに使用します。「型を作って」「インターフェースを定義して」と言われたときに起動します。
```

### セキュリティ考慮

Skillsはフルユーザー権限で実行されます。チームで共有する際の注意点：

1. **コードレビュー必須**: Skillの追加・変更はPRで管理
2. **外部通信の制限**: `scripts/`内で外部APIを叩く場合は監査
3. **機密情報の除外**: API KeyやパスワードをSkillに含めない
4. **信頼できるソースのみ**: 第三者のSkillは内容を確認してから使用

## 辛かったポイント

### 1. Skill同士の競合

複数のSkillが同じタスクに反応してしまう問題。

```yaml
# skill-a
description: コードを書きます。実装、開発のときに使用。

# skill-b
description: TypeScriptを書きます。コード作成のときに使用。
```

これだと両方起動してしまう。解決策は**descriptionの粒度を揃える**こと。

```yaml
# skill-a（汎用）
description: 一般的なコード実装を行います。特定の言語やフレームワークの指定がないときに使用。

# skill-b（特化）
description: TypeScript/Reactのコード実装を行います。フロントエンド、React、TypeScriptの実装のときに使用。
```

### 2. Subagentへの継承が不完全

Subagentを起動したとき、SkillsのLevel 1（メタデータ）は継承されますが、コンテキストは共有されません。

「さっきのレビュー方針で進めて」とSubagentに言っても、「さっきの」がわからない。

対策：Subagent起動時に必要なコンテキストを明示的に渡す。

### 3. デバッグが難しい

「どのSkillが発動したか」「Level何まで読み込まれたか」を追うのが大変。

```bash
# デバッグモードで起動
claude --debug
```

これでSkillのロードログが見えます。本番運用前にデバッグモードで動作確認するのがおすすめ。

## 学び・Tips

### 1. 「1 Skill = 1 目的」の原則

大きなSkillを作りたくなりますが、分割した方がいい。

```
# NG: 大きすぎる
code-quality/
└── SKILL.md  # レビュー、テスト、リファクタ全部入り

# OK: 分割
code-review/
└── SKILL.md
test-generator/
└── SKILL.md
refactoring-guide/
└── SKILL.md
```

### 2. SKILL.md は 5,000 トークン以下

これを超えると、Claudeがコンテキストを消費しすぎて、本来のタスクに集中できなくなる。

長くなったら：
- `reference/`に分離
- 別Skillに分割
- CLAUDE.mdに移動（全タスク共通なら）

### 3. バージョン管理と段階的ロールアウト

新しいSkillは、まず一部メンバーでテスト。

```
.claude/skills/
├── experimental/          # 実験中（一部メンバーのみ）
│   └── new-feature/
└── stable/                # 安定版（全員）
    └── code-review/
```

### 参考リソース

- [Equipping agents for the real world with Agent Skills - Anthropic Engineering](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Skill authoring best practices - Claude Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Skills explained - Claude Blog](https://claude.com/blog/skills-explained)
- [Subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [awesome-claude-skills - GitHub](https://github.com/travisvn/awesome-claude-skills)

## まとめ

Agent Skillsを本番運用するための設計パターンをまとめました。

### 覚えておきたいこと

1. **Progressive Disclosure** を理解する
   - Level 1（メタデータ）→ Level 2（本文）→ Level 3+（リソース）
   - 使わないものはトークンを消費しない

2. **Skills + MCP** の組み合わせ
   - MCP = 接続（何ができるか）
   - Skills = 手順（どう使うか）
   - 両方使うと効果的

3. **Skills vs Subagents** の使い分け
   - 手順書 → Skill
   - 独立ワーカー → Subagent
   - コンテキスト共有したい → Skill

4. **チーム運用**
   - ディレクトリ構成を統一
   - descriptionは具体的に
   - セキュリティに注意

### 最後に

Agent Skillsは「Claudeにドメイン知識を教える」仕組み。Progressive Disclosureのおかげで、大量の知識をバンドルしてもスケールします。

最初は「CLAUDE.mdに書けばいいじゃん」と思っていましたが、チームで運用し始めると、モジュール化・再利用・段階的ロードのメリットが実感できます。

Simon Willison氏の「MCPより大きなインパクトがあるかも」という言葉、使い込むほど納得できるようになりました。ぜひ、チームでの運用を試してみてください。
