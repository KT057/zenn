---
title: "毎回.claudeの構成を考えるのが面倒だったので自分用のボイラープレートを作った"
emoji: "📦"
type: "tech"
topics: ["claudecode", "ai", "開発効率化", "ボイラープレート"]
published: false
---

## はじめに

Claude Codeを使い始めると、最初にやることは`.claude/`ディレクトリの構成を考えることです。

CLAUDE.mdにプロジェクトの方針を書いて、Skillsでスラッシュコマンドを定義して、rulesでコーディング規約を設定して、agentsで専門家エージェントを用意して...。

1つのプロジェクトなら問題ないんですが、新しいプロジェクトを立ち上げるたびに「あれ、前のプロジェクトのSkillsってどういう構成だったっけ？」「rulesのフロントマター、pathsの書き方どうだっけ？」となるのが地味にストレスでした。

**毎回ゼロから考えるのが面倒になったので、自分用のボイラープレートを作りました。**

https://github.com/KT057/claude-code-boilerplate

この記事では、このボイラープレートがどういう構成になっているかを解説します。

## ディレクトリ構成の全体像

```
claude-code-boilerplate/
├── .claude/
│   ├── CLAUDE.md                      # .claude内の説明メモ
│   ├── settings.json                  # チーム共有設定（権限・Hooks）
│   ├── settings.local.json.example    # 個人設定テンプレート
│   │
│   ├── agents/                        # 専門家エージェント
│   │   ├── reviewer.md                #   コードレビュー
│   │   ├── architect.md               #   設計・アーキテクチャ
│   │   ├── test-writer.md             #   テスト作成
│   │   └── debugger.md                #   デバッグ
│   │
│   ├── skills/                        # スラッシュコマンド
│   │   ├── overview/SKILL.md          #   /overview
│   │   ├── gen/SKILL.md               #   /gen
│   │   ├── add-test/SKILL.md          #   /add-test
│   │   ├── create-pr/SKILL.md         #   /create-pr
│   │   ├── check/SKILL.md             #   /check
│   │   ├── troubleshoot/SKILL.md      #   /troubleshoot
│   │   └── doc/SKILL.md               #   /doc
│   │
│   ├── rules/                         # 自動適用ルール
│   │   ├── code-style.md              #   コーディングスタイル
│   │   ├── git-workflow.md            #   Gitワークフロー
│   │   ├── testing.md                 #   テスト規約
│   │   ├── security.md                #   セキュリティ
│   │   └── documentation.md           #   ドキュメント規約
│   │
│   └── hooks/                         # 自動実行スクリプト
│       └── protect-files.sh           #   機密ファイル保護
│
├── CLAUDE.md                          # プロジェクトルートの指示ファイル
├── .gitignore
├── init-claude.sh                     # セットアップスクリプト
└── README.md
```

大きく分けると**5つのレイヤー**で構成しています。

| レイヤー | ディレクトリ | 役割 |
|----------|------------|------|
| 指示 | `CLAUDE.md` | プロジェクトの方針・ルール参照 |
| エージェント | `.claude/agents/` | 専門家への自動委譲 |
| スキル | `.claude/skills/` | オンデマンドのスラッシュコマンド |
| ルール | `.claude/rules/` | コンテキストに応じた自動適用ルール |
| 安全装置 | `.claude/settings.json` + `hooks/` | 権限制御・ファイル保護 |

順番に見ていきます。

## CLAUDE.md — プロジェクトの司令塔

ルートの`CLAUDE.md`はTODOプレースホルダー形式のテンプレートにしています。

```markdown
# プロジェクト名

<!-- TODO: プロジェクト名を記入 -->

## 概要

<!-- TODO: プロジェクトの概要を1-2行で -->

## 技術スタック

<!-- TODO: 使用技術を記入 -->

## 主要コマンド

<!-- TODO: よく使うコマンドを記入 -->

- dev:
- build:
- test:
- lint:

## ルール

@.claude/rules/code-style.md
@.claude/rules/git-workflow.md
@.claude/rules/testing.md
@.claude/rules/security.md
@.claude/rules/documentation.md
```

ポイントは2つあります。

**1. TODOプレースホルダー**

新しいプロジェクトにコピーしたら、`<!-- TODO: -->` の部分を埋めるだけ。「何を書くべきか」を毎回考えなくて済みます。

**2. `@file`インポートでルールを外部化**

`@.claude/rules/code-style.md` のように書くと、外部ファイルの内容をインポートできます。ルールをCLAUDE.mdに直書きすると肥大化するので、`.claude/rules/`に分離してインポートする構成にしました。

もう1つ、Persistent Memoryの更新タイミングも定義しています。

```markdown
## Persistent Memory（自動知見蓄積）

以下のタイミングでPersistent Memoryを更新すること：

- エラーを解決した時 → 「過去のトラブルと解決策」に記録
- 新しい設計判断をした時 → 「コードベースの重要な知見」に記録
- ユーザーの好みや方針がわかった時 → 「ユーザーの好み・スタイル」に記録
- 繰り返し使うコマンドやパターンを発見した時 → 「よく使うコマンド・パターン」に記録
```

Claudeが自動で蓄積するメモリに「いつ・何を記録するか」のガイドラインを入れておくと、セッションを跨いだ知見の蓄積が安定します。

## agents/ — 4人の専門家チーム

カスタムサブエージェントは、特定の専門性を持ったエージェントを定義できる機能です。Claudeが「これはレビューの仕事だな」と判断すると、自動的に該当エージェントに委譲してくれます。

このボイラープレートでは4つのエージェントを用意しました。

### reviewer（コードレビュー）

```yaml
---
name: reviewer
description: Expert code review specialist. Proactively reviews code for quality, security vulnerabilities, and performance issues.
tools: [Read, Grep, Glob]
disallowedTools: [Write, Edit, Bash]
model: sonnet
maxTurns: 15
memory: project
skills: [check]
---
```

- **読み取り専用**（`disallowedTools: [Write, Edit, Bash]`）にしているのがポイント。レビューアーがコードを勝手に書き換えたら困ります
- `skills: [check]` で `/check` スキルの知識を注入。lint/typecheck/testの実行基準を知った上でレビューできます
- セキュリティ(Critical) / パフォーマンス(Warning) / コード品質(Info) の3段階で評価してくれます

### architect（設計・アーキテクチャ）

```yaml
---
name: architect
description: Architecture and design specialist. Use when planning new features, deciding directory structures, making technology choices, or creating refactoring plans.
tools: [Read, Grep, Glob]
disallowedTools: [Write, Edit]
model: opus
maxTurns: 20
memory: project
---
```

- **`model: opus`** を指定。設計判断には高度な推論力が必要なのでOpusを使います
- こちらも読み取り専用。設計を「提案する」だけで、実装はメインエージェントに任せます
- 最低2案のPros/Consを出す、YAGNI/KISS原則を重視する、といったルールを本文に書いています

### test-writer（テスト作成）

```yaml
---
name: test-writer
description: Test writing specialist. Use when writing tests, improving test coverage, or designing test strategies.
tools: [Read, Grep, Glob, Write, Bash]
model: sonnet
maxTurns: 30
memory: project
---
```

- **書き込み権限あり**（Write, Bash）。テストは実際にファイルを作って実行する必要があるためです
- `maxTurns: 30` と多めに設定。テストを書く → 実行 → 失敗 → 修正のループを想定しています
- jest, vitest, pytest, go testなどテストFWの自動検出に対応

### debugger（デバッグ）

```yaml
---
name: debugger
description: Debugging specialist. Use when tracking down bug root causes, analyzing errors, or isolating problems.
tools: [Read, Grep, Glob, Bash]
disallowedTools: [Write, Edit]
model: opus
maxTurns: 25
memory: project
---
```

- **Bashは使えるがWrite/Editは不可**。ログの確認やコマンドの実行はできるが、コードの修正はしない
- 「修正コードは書かない（メインエージェントに委ねる）」という原則。原因を特定するまでが仕事です
- 系統的デバッグ（情報収集 → 仮説立案 → 調査検証 → 根本原因特定）のフローを本文に記載

### エージェント設計の方針

4つのエージェントに共通する設計方針があります。

| 方針 | 理由 |
|------|------|
| **権限を最小限にする** | レビューアーやデバッガーがコードを書き換えると責任の所在が曖昧になる |
| **モデルを使い分ける** | 速度重視のタスクはSonnet、深い推論が必要なタスクはOpus |
| **`memory: project`で知見を蓄積** | セッションを跨いでプロジェクト固有の知見を覚えてくれる |

## skills/ — 7つのスラッシュコマンド

Skillsは`/コマンド名`で呼び出せるオンデマンドの機能です。必要なときだけコンテキストに読み込まれるので、コンテキストウィンドウを無駄に消費しません。

### 一覧

| コマンド | 用途 | context | 特記 |
|----------|------|---------|------|
| `/overview` | プロジェクト俯瞰 | `fork` | Exploreエージェントで実行 |
| `/gen [type] [name]` | コード生成 | - | 既存パターンに追従 |
| `/add-test [file]` | テスト追加 | - | 実行まで確認 |
| `/create-pr` | PR作成 | - | `disable-model-invocation: true` |
| `/check` | 品質チェック | `fork` | lint→typecheck→test一括 |
| `/troubleshoot [error]` | エラー解決 | - | WebSearch使用可 |
| `/doc [target]` | ドキュメント生成 | - | JSDoc/README/API仕様 |

いくつかピックアップして解説します。

### /overview — `context: fork` でコンテキストを守る

```yaml
---
name: overview
description: プロジェクトの全体構造、主要ファイル、依存関係を素早く把握する
context: fork
agent: Explore
allowed-tools: [Read, Glob, Grep, Bash]
---
```

`context: fork` を指定すると、サブエージェントとして別コンテキストで実行されます。プロジェクト構造の探索は大量のファイルを読み込むので、メインのコンテキストウィンドウを消費しないようにしています。

### /create-pr — AIの自動実行を防ぐ

```yaml
---
name: create-pr
description: 現在の変更内容からPRを作成する
disable-model-invocation: true
allowed-tools: [Read, Bash, Grep, Glob]
---
```

`disable-model-invocation: true` がポイント。これを設定すると、Claudeが「PRを作った方がいいかも」と判断しても自動では実行されません。ユーザーが明示的に `/create-pr` と打ったときだけ動きます。PRの作成は意図しないタイミングで実行されると困るので、この制御は必須です。

### /troubleshoot — Web検索でエラーを調査

```yaml
---
name: troubleshoot
description: エラーメッセージやログから原因を特定し修正案を提示する
argument-hint: "[error message or description]"
allowed-tools: [Read, Grep, Glob, Bash, WebSearch]
---
```

`allowed-tools` に `WebSearch` を含めているのが特徴。エラーメッセージでStack OverflowやGitHub Issuesを検索して、最新の解決策を提案してくれます。

## rules/ — パス限定のモジュラールール

`.claude/rules/` に置いたMarkdownファイルは、Claude Codeが自動的に適用します。5つのルールを用意しました。

| ファイル | 内容 |
|----------|------|
| `code-style.md` | インデント2スペース、セミコロンなし、命名規則など |
| `git-workflow.md` | Conventional Commits、ブランチ命名規則、mainへの直push禁止 |
| `testing.md` | Arrange-Act-Assert、テストファイル配置、モック方針 |
| `security.md` | APIキーのハードコード禁止、SQLインジェクション対策 |
| `documentation.md` | パブリックAPIへのJSDoc、README更新タイミング |

### パス限定ルールの活用

`testing.md` にはYAMLフロントマターで `paths` を設定しています。

```yaml
---
paths:
  - "**/*.test.*"
  - "**/*.spec.*"
  - "**/__tests__/**"
  - "**/tests/**"
---
```

これにより、テストファイルを操作しているときだけこのルールが適用されます。テストと無関係な作業中にテスト規約が読み込まれることがないので、コンテキストの効率が良くなります。

## settings.json — 権限とHooksの設定

### 権限の3段階設定

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(pnpm run *)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git stash *)"
    ],
    "deny": [
      "Bash(rm -rf /)",
      "Bash(rm -rf ~)",
      "Read(.env)",
      "Read(.env.*)",
      "Read(**/*.pem)",
      "Read(**/*secret*)"
    ],
    "ask": [
      "Bash(git push *)",
      "Bash(git checkout main)",
      "Bash(git merge *)",
      "Bash(npm publish *)"
    ]
  }
}
```

- **allow**: パッケージマネージャーの`run`コマンドやgitの読み取り系は自動許可
- **deny**: `rm -rf` や `.env` / `.pem` / `secret` を含むファイルの読み取りは完全ブロック
- **ask**: `git push` や `npm publish` など外部に影響する操作は毎回確認

### Hooksでファイル保護と自動フォーマット

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/protect-files.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

**PreToolUse**: ファイルの編集・作成の前に `protect-files.sh` を実行。`.env`やロックファイルへの書き込みをexit code 2でブロックします。

**PostToolUse**: ファイルの編集・作成の後に `npx prettier --write` を実行。Claudeが生成したコードを自動でフォーマットしてくれます。

### protect-files.sh の中身

```bash
#!/bin/bash
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

PROTECTED_PATTERNS=(".env" ".pem" "secret" "credential")
LOCK_FILES=("package-lock.json" "yarn.lock" "pnpm-lock.yaml" "bun.lockb")

for pattern in "${PROTECTED_PATTERNS[@]}"; do
  if [[ "$FILE_PATH" == *"$pattern"* ]]; then
    echo "Blocked: $FILE_PATH matches protected pattern '$pattern'" >&2
    exit 2
  fi
done

for lock in "${LOCK_FILES[@]}"; do
  if [[ "$FILE_PATH" == *"$lock"* ]]; then
    echo "Blocked: Direct editing of lock file '$lock' is not allowed" >&2
    exit 2
  fi
done

exit 0
```

stdinからJSON入力を受け取り、ファイルパスをパターンマッチでチェック。該当したら `exit 2` で編集をブロックします。`settings.json` の `deny` ルールと二重防御になっています。

## セットアップスクリプト

`init-claude.sh` で任意のプロジェクトにボイラープレートを展開できます。

```bash
./init-claude.sh /path/to/your-project
```

やっていることは以下の通りです。

1. `.claude/` ディレクトリ一式をコピー
2. `CLAUDE.md` をコピー（既存がなければ）
3. `.gitignore` にローカル設定の除外エントリを追加
4. パッケージマネージャーの自動検出（bun, pnpm, yarn, npm, python, go, rust）
5. `settings.local.json` をexampleから作成

パッケージマネージャーの自動検出は地味に便利です。`bun.lockb` があればbun、`pnpm-lock.yaml` があればpnpmとして `settings.json` のallowルールを自動調整してくれます。

## .gitignoreの設計

```
.claude/settings.local.json
.claude/CLAUDE.local.md
.claude/agent-memory-local/
```

チームで共有すべきもの（`settings.json`, `agents/`, `skills/`, `rules/`）はVCS管理し、個人設定（`settings.local.json`, `CLAUDE.local.md`）やローカルメモリは`.gitignore`で除外する構成にしています。

## 使ってみた感想

新しいプロダクトを始めるときに`.claude`の設定を考えなくて良くなったのが一番のメリットです。

以前は新規プロジェクトのたびに「CLAUDE.mdに何書くか」「Skillsどう構成するか」「権限設定どうするか」を考えていましたが、今は `init-claude.sh` を実行して `CLAUDE.md` のTODO部分を埋めるだけで開発に入れます。

特にrulesとhooksは一度いい設定ができればプロジェクト間で使い回せるので、ボイラープレート化の恩恵が大きいです。

## まとめ

Claude Codeの`.claude/`ディレクトリは、使いこなすほど複雑になっていきます。毎回ゼロから構成を考えるのは非効率なので、自分なりのベストプラクティスをボイラープレートとして固めておくと楽です。

このボイラープレートが参考になれば幸いです。自分のプロジェクトに合わせてカスタマイズして使ってみてください。

https://github.com/KT057/claude-code-boilerplate

## 参考

- [Claude Code 公式ドキュメント](https://code.claude.com/docs/en/)
- [CLAUDE.md / Memory](https://code.claude.com/docs/en/memory)
- [Skills](https://code.claude.com/docs/en/skills)
- [Custom Subagents](https://code.claude.com/docs/en/agents)
- [Hooks](https://code.claude.com/docs/en/hooks)
- [Settings](https://code.claude.com/docs/en/settings)
- [Permissions](https://code.claude.com/docs/en/permissions)
