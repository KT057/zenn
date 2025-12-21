---
title: "Claude CodeのAgent Skillsで開発ワークフローを自動化してみた"
emoji: "🎯"
type: "tech"
topics: ["claudecode", "agentskills", "ai", "automation", "開発効率化"]
published: false
---

## はじめに

「CLAUDE.md、また長くなってきたな...」

Claude Codeを使い込んでいくと、プロジェクトの方針やコーディング規約をCLAUDE.mdに書き足していくことになります。最初は1,000文字くらいだったのが、気づいたら8,000文字を超えていて、肝心のツール説明をClaudeが見つけられなくなる...という経験、ありませんか？

そこで出会ったのが**Agent Skills**。特定のタスクに特化した指示をモジュール化できる機能です。使ってみたら「これ、もっと早く知りたかった」と思ったので、実践的な使い方を共有します。

## Agent Skillsとは

Agent Skillsは、Claudeに「専門知識」を追加するための仕組みです。

**CLAUDE.mdとの違い:**
- CLAUDE.md → 常に読み込まれる（全タスクに適用）
- Agent Skills → 必要なときだけ読み込まれる（関連するタスクのみ）

この「必要なときだけ」がポイント。コードレビューの方針はレビュー時だけ、テスト作成の手順はテスト時だけ読み込まれる。コンテキストウィンドウを無駄に消費しません。

公式ドキュメントには「ミニプラグイン」や「能力パック」のようなものと書かれていて、まさにそんな感じです。

参考: [Agent Skills - Claude Docs](https://platform.claude.com/docs/ja/agents-and-tools/agent-skills/overview)

## なぜAgent Skillsを使うのか

### 1. 段階的な情報開示

Skillsは3段階で読み込まれます。

| レベル | 読み込まれるタイミング | トークンコスト |
|--------|------------------------|----------------|
| メタデータ | 常に（起動時） | 約100トークン/Skill |
| 指示 | Skillがトリガーされたとき | 5,000トークン未満推奨 |
| リソース | 必要に応じて | 実質無制限 |

つまり、10個のSkillをインストールしても、起動時に消費するのは約1,000トークンだけ。実際に使うときだけ詳細が読み込まれる。

### 2. 再利用性

一度作ったSkillは、複数プロジェクトで使い回せます。

- **個人用Skill**: `~/.claude/skills/` に保存
- **プロジェクトSkill**: `.claude/skills/` に保存（gitでチーム共有可能）

### 3. Claudeが自律的に使う

スラッシュコマンドと違って、ユーザーが明示的に呼び出す必要がありません。Claudeがタスクの内容を見て「このSkillを使うべきだ」と判断してくれます。

## 実装してみた

### 基本的なSkillの作成

まず、最もシンプルなSkillを作ってみます。

```bash
mkdir -p ~/.claude/skills/code-reviewer
touch ~/.claude/skills/code-reviewer/SKILL.md
```

`SKILL.md`の中身:

```markdown
---
name: code-reviewer
description: コードレビューを行います。コードの品質、バグの可能性、セキュリティ上の問題をチェックするときに使用します。
---

# コードレビュー

## レビュー観点

1. **バグの可能性**: nullチェック、境界値、エラーハンドリング
2. **セキュリティ**: インジェクション、認証・認可、機密情報の露出
3. **パフォーマンス**: N+1問題、不要なループ、メモリリーク
4. **可読性**: 命名、関数の長さ、コメントの適切さ
5. **テスタビリティ**: 依存性注入、副作用の分離

## レビューの進め方

1. まず全体の構造を把握する
2. 変更の意図を理解する
3. 上記の観点でチェックする
4. 具体的な改善案を提示する

## 出力フォーマット

- 重要度: 🔴 Critical / 🟡 Warning / 🟢 Suggestion
- 該当箇所: ファイル名と行番号
- 問題点: 何が問題か
- 改善案: どう直すか
```

これだけ。Claude Codeを再起動すると、コードレビューを依頼したときに自動的にこのSkillが使われます。

### スクリプト付きSkillの作成

もう少し実践的な例として、テスト生成Skillを作ってみます。

```
~/.claude/skills/test-generator/
├── SKILL.md
├── templates/
│   └── jest.template.ts
└── scripts/
    └── analyze_coverage.sh
```

`SKILL.md`:

```markdown
---
name: test-generator
description: TypeScript/JavaScriptのユニットテストを生成します。テストを書く、テストを追加する、カバレッジを上げるときに使用します。
---

# テスト生成

## 使用するテンプレート

テストは `templates/jest.template.ts` をベースに生成します。

## テスト生成の手順

1. 対象ファイルを読み込む
2. 関数・クラスの一覧を抽出
3. 各関数に対して以下のテストケースを生成:
   - 正常系: 期待通りの入力で期待通りの出力
   - 境界値: 空配列、0、null、undefined
   - 異常系: 不正な入力に対するエラー

## テスト命名規則

```
describe('[クラス名/関数名]', () => {
  describe('[メソッド名]', () => {
    it('should [期待する動作] when [条件]', () => {
      // ...
    });
  });
});
```

## カバレッジ確認

テスト生成後、`scripts/analyze_coverage.sh` を実行してカバレッジを確認します。
```

`templates/jest.template.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
// または import { describe, it, expect, beforeEach, afterEach, jest } from '@jest/globals';

describe('{{CLASS_NAME}}', () => {
  let instance: {{CLASS_NAME}};

  beforeEach(() => {
    instance = new {{CLASS_NAME}}();
  });

  afterEach(() => {
    vi.clearAllMocks();
  });

  describe('{{METHOD_NAME}}', () => {
    it('should return expected result when given valid input', () => {
      // Arrange
      const input = {{SAMPLE_INPUT}};
      const expected = {{EXPECTED_OUTPUT}};

      // Act
      const result = instance.{{METHOD_NAME}}(input);

      // Assert
      expect(result).toEqual(expected);
    });

    it('should throw error when given invalid input', () => {
      // Arrange
      const invalidInput = {{INVALID_INPUT}};

      // Act & Assert
      expect(() => instance.{{METHOD_NAME}}(invalidInput)).toThrow();
    });
  });
});
```

`scripts/analyze_coverage.sh`:

```bash
#!/bin/bash
# カバレッジレポートを生成して要約を出力

npm run test:coverage 2>/dev/null || yarn test:coverage 2>/dev/null || pnpm test:coverage 2>/dev/null

if [ -f coverage/coverage-summary.json ]; then
  echo "=== Coverage Summary ==="
  cat coverage/coverage-summary.json | jq '.total'
else
  echo "Coverage report not found. Make sure vitest or jest is configured with coverage."
fi
```

## 辛かったポイント

正直、最初はハマりまくりました。

### 1. descriptionの書き方が難しい

「いつ使うか」を明示的に書かないと、Claudeが「このSkillを使うべきだ」と判断してくれません。

```yaml
# ダメな例
description: コードレビューを行います

# 良い例
description: コードレビューを行います。コードの品質、バグの可能性、セキュリティ上の問題をチェックするときに使用します。「レビューして」「コードを見て」と言われたときに起動します。
```

書きすぎると意図しない場面で起動するし、曖昧だと全然起動しない。この塩梅を見つけるのに時間がかかりました。

### 2. SKILL.mdが読み込まれない

作ったのに全然使われない...という状況。原因は大体これ:

- **YAMLフロントマターの構文エラー**: インデントずれ、コロンの後のスペース忘れ
- **ファイル名の間違い**: `skill.md`じゃなくて`SKILL.md`（大文字）
- **ディレクトリ構造**: `.claude/skills/skill-name/SKILL.md`の階層を守る

デバッグモードで起動すると読み込みエラーが見えます:

```bash
claude --debug
```

### 3. スクリプトの実行権限

`scripts/`に置いたシェルスクリプトが動かない。原因は実行権限。

```bash
chmod +x scripts/analyze_coverage.sh
```

これを忘れがち。

### 4. Subagentとの使い分け

最初は「全部Skillにすればいいじゃん」と思っていたんですが、使い分けがあることに気づきました。

- **Skill向き**: 結果だけでなく「過程」も重要なタスク（コードレビュー、リファクタリング）
- **Subagent向き**: 試行錯誤を含むタスク（エラー調査、情報収集）

Skillはコンテキストを共有するので、TDDのようにRED→GREEN→REFACTORのサイクルで文脈を保持したまま作業したいときに向いています。

## よかったポイント

### 1. CLAUDE.mdがスッキリした

8,000文字あったCLAUDE.mdが2,000文字に。プロジェクト全体に関わる方針だけ残して、タスク固有の指示はSkillに分離できました。

### 2. チームで共有できる

`.claude/skills/`に入れてgitにコミットすれば、チームメンバー全員が同じSkillを使えます。「このプロジェクトのテストはこう書く」というルールが自然に浸透する。

### 3. 段階的な情報開示

大量のドキュメントをSkillにバンドルしても、使わなければトークンを消費しません。API仕様書や設計書を参照ファイルとして入れておいて、必要なときだけ読み込ませる。

### 4. 公式のSkillsがそのまま使える

Anthropicが提供している事前構築Skillsがあります:

- `pptx`: PowerPoint作成
- `xlsx`: Excel操作
- `docx`: Word文書作成
- `pdf`: PDF操作

これらはclaude.aiやAPIで最初から使えます。自分で作る前に公式を確認するといいかも。

参考: [GitHub - anthropics/skills](https://github.com/anthropics/skills)

## 実践的なSkillの例

実際に使っているSkillをいくつか紹介します。

### PRレビュースキル

```markdown
---
name: pr-reviewer
description: GitHubのPull Requestをレビューします。PRレビュー、コードレビュー、マージ前チェックのときに使用します。
---

# PRレビュー

## チェックリスト

### 1. 変更の妥当性
- [ ] PRのタイトルと説明が変更内容を正確に反映している
- [ ] 変更が1つの目的に集中している（巨大PRは分割を提案）

### 2. コード品質
- [ ] 既存のコーディング規約に従っている
- [ ] 重複コードがない
- [ ] 適切な抽象化がされている

### 3. テスト
- [ ] 新機能にテストがある
- [ ] 既存テストが壊れていない
- [ ] エッジケースがカバーされている

### 4. セキュリティ
- [ ] 機密情報がハードコードされていない
- [ ] 入力のバリデーションがある
- [ ] 認証・認可が適切

## レビューコメントのフォーマット

- 🔴 **Must Fix**: マージをブロックする問題
- 🟡 **Should Fix**: 修正が望ましいがブロックしない
- 🟢 **Nitpick**: 好みの問題、任意
- 💡 **Suggestion**: 改善案の提案
- ❓ **Question**: 理解のための質問
```

### コミットメッセージ生成スキル

```markdown
---
name: commit-message
description: 変更内容から適切なコミットメッセージを生成します。コミット、git commit、変更をまとめるときに使用します。
---

# コミットメッセージ生成

## フォーマット

```
<type>(<scope>): <subject>

<body>

<footer>
```

## Type

- `feat`: 新機能
- `fix`: バグ修正
- `docs`: ドキュメントのみ
- `style`: コードの意味に影響しない変更
- `refactor`: バグ修正でも新機能でもないコード変更
- `perf`: パフォーマンス改善
- `test`: テストの追加・修正
- `chore`: ビルドプロセスやツールの変更

## ルール

1. subjectは50文字以内
2. bodyは72文字で折り返す
3. 現在形で書く（「added」ではなく「add」）
4. なぜ変更したかを説明する（何をではなく）
```

## 学び・Tips

### Skillを作るときのコツ

1. **descriptionは具体的に**: 「いつ使うか」を明示する
2. **SKILL.mdは5,000トークン以下**: 長くなりすぎたら分割
3. **スクリプトは実行してテスト**: 権限とパスを確認
4. **まずシンプルに始める**: リソースファイルは後から追加

### おすすめの運用

- **個人Skill**: `~/.claude/skills/` に自分の好みを反映したSkillを置く
- **プロジェクトSkill**: `.claude/skills/` にプロジェクト固有のルールを置く
- **チーム共有**: プロジェクトSkillをgitで管理して、全員が同じSkillを使う

### 参考リソース

- [Agent Skills Overview - Claude Docs](https://platform.claude.com/docs/ja/agents-and-tools/agent-skills/overview)
- [Claude Code Skills - Claude Code Docs](https://code.claude.com/docs/ja/skills)
- [GitHub - anthropics/skills](https://github.com/anthropics/skills)
- [Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

## まとめ

### Agent Skillsが向いている場面

- CLAUDE.mdが長くなってきた
- タスク固有のルールが増えてきた
- チームで開発ルールを統一したい
- 繰り返し同じ説明をするのが面倒

### 向いていない場面

- プロジェクト全体に適用するルール → CLAUDE.mdに書く
- 試行錯誤が多いタスク → Subagentを使う
- 一回きりの作業 → プロンプトに直接書く

### 最後に

Agent Skillsは「Claudeに専門知識を教える」ための仕組みです。最初はdescriptionの書き方に悩んだり、読み込まれなくて焦ったりしましたが、コツを掴むと開発が格段に楽になりました。

特にチーム開発で効果を発揮します。「このプロジェクトではこうする」というルールをSkillにまとめておけば、新メンバーが来てもすぐに同じ品質でコードが書ける。

Simon Willison氏が「MCPより大きなインパクトがあるかも」と言っていましたが、使ってみてその意味がわかりました。まずは1つ、簡単なSkillから作ってみてください。

---

この記事で参照したリンク:

- [Agent Skills Overview - Claude Docs](https://platform.claude.com/docs/ja/agents-and-tools/agent-skills/overview)
- [Claude Code Skills - Claude Code Docs](https://code.claude.com/docs/ja/skills)
- [GitHub - anthropics/skills](https://github.com/anthropics/skills)
- [Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Claude Skills are awesome, maybe a bigger deal than MCP - Simon Willison](https://simonwillison.net/2025/Oct/16/claude-skills/)
