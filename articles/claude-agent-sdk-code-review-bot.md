---
title: "Claude Agent SDKでコードレビューbotを作ってみた｜ハマったこと・よかったこと"
emoji: "🤖"
type: "tech"
topics: ["claudeagentsdk", "ai", "typescript", "codereview", "automation"]
published: true
---

## はじめに

「PR のレビュー、時間かかるんだよな...」

チームで開発していると、コードレビューって地味に時間を取られますよね。特に細かい指摘（変数名とか、typo とか）は人間がやらなくてもいいんじゃないかと思って、Claude Agent SDK でコードレビュー bot を作ってみました。

この記事では、実際に作ってみて「これは辛かった」「これは助かった」というリアルな感想を共有します。エンジニアになりたての方で「AI エージェント、気になるけど実際どうなの？」という人の参考になれば嬉しいです。

## なぜ Claude Agent SDK を選んだか

コードレビュー自動化を考えたとき、いくつか選択肢がありました。

- **LangChain/LangGraph**: 有名だけど、ちょっと重厚すぎる印象
- **OpenAI Agents SDK**: GPT 系で安定してそう
- **Claude Agent SDK**: Claude Code と同じ基盤、コード関連に強そう

Claude Agent SDK を選んだ決め手は、**Claude Code を動かしているのと同じ基盤**だということ。普段 Claude Code を使っていて「コードの理解力すごいな」と感じていたので、その力をそのまま活かせるのは魅力的でした。

あと、TypeScript でサクッと書けるのもポイント。Python も対応していますが、普段 TypeScript 書いているのでそっちで進めることに。

参考: [Agent SDK overview - Claude Docs](https://platform.claude.com/docs/en/agent-sdk/overview)

## 実装してみた

### セットアップ

まずはインストールから。公式パッケージを入れます。

```bash
npm install @anthropic-ai/claude-agent-sdk
```

環境変数に API キーを設定。

```bash
export ANTHROPIC_API_KEY=your-api-key
```

### 最初のコード

最小限のコードはこんな感じ。本当にシンプルです。

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

async function reviewCode(code: string) {
  const result = query({
    prompt: `以下のコードをレビューしてください。
バグの可能性、改善点、セキュリティ上の問題があれば指摘してください。

\`\`\`typescript
${code}
\`\`\``,
    options: {
      maxTurns: 3,
    },
  });

  for await (const message of result) {
    if (message.type === "assistant") {
      console.log(message.content);
    }
  }
}
```

動かしてみると...ちゃんとレビューしてくれた！

でも、これだと単純に Claude にプロンプトを投げているだけ。Agent SDK の真価は**ツールを使った自律的な動作**にあります。

### MCP でカスタムツールを追加

ここからが本番。MCP サーバーを作って、PR の差分を取得するツールを追加します。

```typescript
import {
  query,
  tool,
  createSdkMcpServer,
} from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

// GitHub PRの差分を取得するツール
const getPrDiffTool = tool(
  "get_pr_diff",
  "GitHubのPRから差分コードを取得する",
  {
    owner: z.string().describe("リポジトリのオーナー"),
    repo: z.string().describe("リポジトリ名"),
    pr_number: z.number().describe("PRの番号"),
  },
  async (args) => {
    const response = await fetch(
      `https://api.github.com/repos/${args.owner}/${args.repo}/pulls/${args.pr_number}`,
      {
        headers: {
          Accept: "application/vnd.github.v3.diff",
          Authorization: `token ${process.env.GITHUB_TOKEN}`,
        },
      }
    );
    const diff = await response.text();
    return { content: [{ type: "text", text: diff }] };
  }
);

// MCPサーバーを作成
const reviewServer = createSdkMcpServer({
  name: "code-review-tools",
  version: "1.0.0",
  tools: [getPrDiffTool],
});
```

ポイントは `tool()` 関数。Zod でスキーマを定義すると、型安全にツールが作れます。

参考: [カスタムツール - Claude Docs](https://docs.claude.com/ja/api/agent-sdk/custom-tools)

## 辛かったポイント

正直、ここからが本当の戦いでした。

### 1. MCP ツールの命名規則でハマった

最初「ツールが見つからない」というエラーが出て焦りました。原因は命名規則。

MCP サーバー経由でツールを公開すると、名前が自動的に変換されます。

```
元: get_pr_diff
変換後: mcp__code-review-tools__get_pr_diff
```

プロンプトで「get_pr_diff を使って」と書いても動かない。`mcp__code-review-tools__get_pr_diff`と書かないとダメでした。これ、ドキュメント読んでなくて 30 分くらい溶かしました...。

### 2. 非同期ジェネレータの扱い

`query()` は async generator を返すので、普通の async/await とちょっと違う書き方が必要。

```typescript
// これはダメ
const result = await query({ prompt: "..." });

// こう書く
const result = query({ prompt: "..." });
for await (const message of result) {
  // ストリーミングで受け取る
}
```

最初「なんで await 付けても動かないんだ？」と混乱しました。ドキュメントには書いてあるんですが、見落としてた...。

ちなみに、V2 Interface（プレビュー版）ではこの辺がシンプルになるみたいです。

参考: [TypeScript SDK V2 interface (preview)](https://platform.claude.com/docs/en/agent-sdk/typescript-v2-preview)

### 3. 設定ファイルが読み込まれない問題

これは移行時の話ですが、Claude Code SDK から Claude Agent SDK に移行した人は注意。

CLAUDE.md や settings.json などの設定ファイルは、**明示的に指定しないと読み込まれません**。自分の場合、プロジェクトルートに CLAUDE.md 置いてたのに反映されなくて「あれ？」となりました。

参考: [Claude Agent SDK への移行](https://code.claude.com/docs/ja/sdk/migration-guide)

## よかったポイント

辛いことばかり書きましたが、よかったこともたくさんあります。

### 1. 組み込みツールが便利

ファイル読み込み、コマンド実行、Web 検索など、よく使う機能が最初から入っています。

```typescript
import {
  FileReadInput,
  BashInput,
  GrepInput,
} from "@anthropic-ai/claude-agent-sdk";
```

自分でツール実装しなくていいので、本当に作りたいもの（今回なら PR 差分取得）に集中できました。

### 2. サブエージェントが使える

複雑なタスクを分割したいとき、サブエージェントを立てられます。

例えば「セキュリティチェック」と「パフォーマンスチェック」を並列で実行とか。自動でコンテキストを管理してくれるので、長い会話でもトークン制限を気にしなくていい。

### 3. 型安全にツールが作れる

Zod スキーマで定義するので、TypeScript の恩恵をフルに受けられます。引数の typo とか、コンパイル時に気づける。

```typescript
// 型が効く！
tool(
  "example",
  "説明",
  {
    count: z.number().min(1).max(10),
    name: z.string().optional(),
  },
  async (args) => {
    // args.count は number 型
    // args.name は string | undefined 型
  }
);
```

## 完成形のコード

最終的にこんな感じになりました。

```typescript
import {
  query,
  tool,
  createSdkMcpServer,
} from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

// GitHub PR差分取得ツール
const getPrDiffTool = tool(
  "get_pr_diff",
  "GitHubのPRから差分コードを取得する",
  {
    owner: z.string(),
    repo: z.string(),
    pr_number: z.number(),
  },
  async (args) => {
    const response = await fetch(
      `https://api.github.com/repos/${args.owner}/${args.repo}/pulls/${args.pr_number}`,
      {
        headers: {
          Accept: "application/vnd.github.v3.diff",
          Authorization: `token ${process.env.GITHUB_TOKEN}`,
        },
      }
    );
    const diff = await response.text();
    return { content: [{ type: "text", text: diff }] };
  }
);

// レビューコメント投稿ツール
const postReviewCommentTool = tool(
  "post_review_comment",
  "PRにレビューコメントを投稿する",
  {
    owner: z.string(),
    repo: z.string(),
    pr_number: z.number(),
    body: z.string().describe("レビューコメントの本文"),
  },
  async (args) => {
    const response = await fetch(
      `https://api.github.com/repos/${args.owner}/${args.repo}/pulls/${args.pr_number}/reviews`,
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `token ${process.env.GITHUB_TOKEN}`,
        },
        body: JSON.stringify({
          body: args.body,
          event: "COMMENT",
        }),
      }
    );
    const result = await response.json();
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  }
);

// MCPサーバー
const reviewServer = createSdkMcpServer({
  name: "code-review-tools",
  version: "1.0.0",
  tools: [getPrDiffTool, postReviewCommentTool],
});

// メイン関数
async function reviewPr(owner: string, repo: string, prNumber: number) {
  const result = query({
    prompt: `
あなたはシニアエンジニアです。
${owner}/${repo}のPR #${prNumber}をレビューしてください。

以下の観点でチェックしてください:
1. バグの可能性
2. セキュリティ上の問題
3. パフォーマンスの問題
4. 可読性・保守性

まずmcp__code-review-tools__get_pr_diffでPRの差分を取得し、
レビューが完了したらmcp__code-review-tools__post_review_commentで結果を投稿してください。
    `,
    mcpServers: [reviewServer],
    options: {
      maxTurns: 10,
    },
  });

  for await (const message of result) {
    if (message.type === "assistant") {
      console.log(message.content);
    }
  }
}

// 実行
reviewPr("myorg", "myrepo", 123);
```

## 学び・Tips

### 次にやるなら気をつけること

1. **MCP ツールの命名規則は最初に確認する** - 地味にハマる
2. **async generator の扱いに慣れておく** - V2 が安定したらそっちがおすすめ
3. **公式のデモリポジトリを見る** - 実装パターンの参考になる

### おすすめリソース

- [anthropics/claude-agent-sdk-demos](https://github.com/anthropics/claude-agent-sdk-demos) - 公式デモ集
- [anthropics/claude-code-action](https://github.com/anthropics/claude-code-action) - GitHub Actions 連携ならこっちが楽
- [Agent SDK reference - TypeScript](https://docs.claude.com/en/api/agent-sdk/typescript) - 公式リファレンス

### GitHub Actions で自動化するなら

実は、単純な PR レビューなら公式の[Claude Code GitHub Action](https://github.com/marketplace/actions/claude-code-action-official)を使うほうが楽です。`@claude`とメンションするだけで動きます。

自分で SDK を使うメリットは、**カスタマイズ性**。特定のルール（「このプロジェクトでは snake_case を使う」とか）を組み込みたいなら、自作する価値があります。

## まとめ

### Claude Agent SDK が向いている場面

- コード関連のタスク（レビュー、リファクタリング、ドキュメント生成）
- 既存の MCP ツールと組み合わせたい
- TypeScript/Python で型安全に書きたい
- Claude Code の機能をカスタマイズして使いたい

### 向いていない場面

- 単純なチャット bot（普通の API 呼び出しで十分）
- リアルタイム性が求められる場面（ツール呼び出しで時間がかかる）
- 超シンプルなタスク（オーバーキル感がある）

### 最後に

「PR の軽微な指摘を AI に任せる」という目的は達成できました。人間は設計レベルのレビューに集中できるようになって、チーム全体の生産性が上がった気がします。

Claude Agent SDK、最初はちょっとクセがありますが、慣れると「こういうの作りたいな」がサクッと形になります。AI エージェント開発に興味ある人は、ぜひ公式デモから触ってみてください。

---

この記事で参照したリンク:

- [Agent SDK overview - Claude Docs](https://platform.claude.com/docs/en/agent-sdk/overview)
- [カスタムツール - Claude Docs](https://docs.claude.com/ja/api/agent-sdk/custom-tools)
- [TypeScript SDK V2 interface (preview)](https://platform.claude.com/docs/en/agent-sdk/typescript-v2-preview)
- [Claude Agent SDK への移行](https://code.claude.com/docs/ja/sdk/migration-guide)
- [anthropics/claude-agent-sdk-demos](https://github.com/anthropics/claude-agent-sdk-demos)
- [Claude Code GitHub Action](https://github.com/marketplace/actions/claude-code-action-official)
