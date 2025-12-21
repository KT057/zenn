---
title: "Claude Agent SDKでセッション間メモリ管理ツールを自作してみた"
emoji: "🧠"
type: "tech"
topics: ["claudeagentsdk", "ai", "typescript", "memory", "mcp"]
published: false
---

## はじめに

「あれ、さっき話したこと覚えてないの...？」

Claude Codeを使っていると、セッションが切れた瞬間にそれまでの文脈がリセットされる問題に悩まされていました。長時間の開発セッションで積み上げた「このプロジェクトではこういう方針で進める」みたいな暗黙知が、再起動するたびに消えてしまう。

毎回同じ説明を繰り返すのが面倒になって、「自分でメモリ管理ツール作れないかな」と思い立ちました。調べてみると、Claude Agent SDKを使えばClaude Codeと同じ基盤でカスタムツールが作れるらしい。

この記事では、実際にセッション間でコンテキストを保持するメモリ管理ツールを自作した体験を共有します。

## なぜClaude Agent SDKを選んだか

メモリ管理の選択肢はいくつかありました。

- **claude-mem（既存プラグイン）**: 便利そうだけど、自分の用途に合わせてカスタマイズしたい
- **LangChain + ベクトルDB**: RAGを組むには重厚すぎる
- **Claude Agent SDK**: Claude Codeと同じ基盤、MCPでツール追加できる

Claude Agent SDKを選んだ理由は、**シンプルなファイルベースのアプローチ**が取れること。AnthropicがClaude Codeで採用している方法も、複雑なベクトルDBではなく`CLAUDE.md`というMarkdownファイルでメモリを管理している。この設計思想に共感しました。

あと、TypeScriptで型安全に書けるのも大きい。

参考: [Agent SDK overview - Claude Docs](https://platform.claude.com/docs/en/agent-sdk/overview)

## 実装してみた

### セットアップ

まずはSDKをインストール。

```bash
npm install @anthropic-ai/claude-agent-sdk zod
```

環境変数にAPIキーを設定。

```bash
export ANTHROPIC_API_KEY=your-api-key
```

### 設計方針

最初に考えたのは「どこにメモリを保存するか」。選択肢は3つ。

1. **ファイルシステム（JSONやMarkdown）**: シンプル、デバッグしやすい
2. **SQLite**: 構造化データに強い、検索も可能
3. **ベクトルDB**: セマンティック検索できる、でも重い

今回は**ファイルシステム**を選びました。理由は単純で、Claude自身がファイルを読み書きできるから。複雑なDB接続とか考えなくていい。

### MCPツールを作る

メモリ管理用に3つのツールを作りました。

```typescript
import {
  query,
  tool,
  createSdkMcpServer,
} from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";
import * as fs from "fs/promises";
import * as path from "path";

const MEMORY_DIR = path.join(process.env.HOME || "", ".claude-memory");

// メモリ保存ツール
const saveMemoryTool = tool(
  "save_memory",
  "重要な情報やコンテキストをメモリに保存する",
  {
    key: z.string().describe("メモリのキー（例: project-context, user-preferences）"),
    content: z.string().describe("保存する内容"),
    tags: z.array(z.string()).optional().describe("検索用タグ"),
  },
  async (args) => {
    await fs.mkdir(MEMORY_DIR, { recursive: true });

    const memory = {
      key: args.key,
      content: args.content,
      tags: args.tags || [],
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    };

    const filePath = path.join(MEMORY_DIR, `${args.key}.json`);
    await fs.writeFile(filePath, JSON.stringify(memory, null, 2));

    return {
      content: [{ type: "text", text: `メモリを保存しました: ${args.key}` }],
    };
  }
);

// メモリ取得ツール
const getMemoryTool = tool(
  "get_memory",
  "保存されたメモリを取得する",
  {
    key: z.string().describe("取得するメモリのキー"),
  },
  async (args) => {
    const filePath = path.join(MEMORY_DIR, `${args.key}.json`);

    try {
      const data = await fs.readFile(filePath, "utf-8");
      const memory = JSON.parse(data);
      return {
        content: [{ type: "text", text: JSON.stringify(memory, null, 2) }],
      };
    } catch {
      return {
        content: [{ type: "text", text: `メモリが見つかりません: ${args.key}` }],
      };
    }
  }
);

// メモリ一覧ツール
const listMemoriesTool = tool(
  "list_memories",
  "保存されているメモリの一覧を取得する",
  {
    tag: z.string().optional().describe("フィルタリング用タグ"),
  },
  async (args) => {
    try {
      const files = await fs.readdir(MEMORY_DIR);
      const memories = [];

      for (const file of files) {
        if (!file.endsWith(".json")) continue;

        const data = await fs.readFile(path.join(MEMORY_DIR, file), "utf-8");
        const memory = JSON.parse(data);

        // タグでフィルタリング
        if (args.tag && !memory.tags.includes(args.tag)) continue;

        memories.push({
          key: memory.key,
          tags: memory.tags,
          updatedAt: memory.updatedAt,
        });
      }

      return {
        content: [{ type: "text", text: JSON.stringify(memories, null, 2) }],
      };
    } catch {
      return {
        content: [{ type: "text", text: "メモリディレクトリが空です" }],
      };
    }
  }
);
```

ポイントは`~/.claude-memory/`というディレクトリにJSONファイルとして保存すること。プロジェクトごとではなくユーザー単位で保存するので、どのプロジェクトからでもアクセスできます。

### MCPサーバーとして公開

作ったツールをMCPサーバーにまとめます。

```typescript
const memoryServer = createSdkMcpServer({
  name: "memory-tools",
  version: "1.0.0",
  tools: [saveMemoryTool, getMemoryTool, listMemoriesTool],
});
```

### 使ってみる

実際に動かしてみます。

```typescript
async function main() {
  const result = query({
    prompt: `
あなたはメモリ管理機能を持つアシスタントです。

まず、list_memoriesで既存のメモリを確認してください。
次に、以下の情報をメモリに保存してください：

- key: "coding-style"
- content: "このプロジェクトではTypeScriptを使用。変数名はcamelCase、定数はUPPER_SNAKE_CASEで統一"
- tags: ["style", "typescript"]

保存後、get_memoryで保存内容を確認してください。
    `,
    mcpServers: [memoryServer],
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

main();
```

動いた！ちゃんとファイルに保存されて、次のセッションでも取得できる。

## 辛かったポイント

正直、ここからが本当の戦いでした。

### 1. MCPツールの命名規則、また忘れてた

前回の記事でも書いたんですが、MCPサーバー経由のツール名は自動変換されます。

```
元: save_memory
変換後: mcp__memory-tools__save_memory
```

プロンプトで「save_memoryを使って」と書いても動かない。学習しない自分...。

### 2. 非同期処理でファイルが壊れる

最初、複数のメモリを同時に保存しようとしたらJSONが壊れました。

```typescript
// ダメなパターン（並列書き込み）
await Promise.all([
  saveMemory("key1", "content1"),
  saveMemory("key2", "content2"),
]);
```

同じファイルに書き込もうとしてrace conditionが発生。解決策として、キーごとに別ファイルにする設計に変更しました。

### 3. メモリの取捨選択が難しい

「何を保存するか」の判断が難しい。全部保存するとノイズが多くなるし、厳選しすぎると大事な情報が漏れる。

結局、**タグでカテゴリ分け**して、必要なときに必要なタグのメモリだけ読み込む方式にしました。Anthropicが推奨している「最小限の高シグナルなトークン」という考え方に近い。

参考: [Context is Key - Claude-mem](https://typevar.dev/articles/thedotmack/claude-mem)

### 4. セッション開始時の自動読み込み

「セッション開始時に自動でメモリを読み込みたい」と思ったんですが、これが意外と難しい。Agent SDKには`SessionStart`フックがあるものの、そこからツールを呼び出す方法がわからなくて断念。

結局、プロンプトの最初に「まずlist_memoriesでメモリを確認して」と書く運用でカバーしています。ちょっとダサいけど動く。

## よかったポイント

辛いことばかり書きましたが、よかったこともあります。

### 1. ファイルベースの透明性

保存されたメモリは普通のJSONファイルなので、直接確認・編集できます。デバッグが楽。

```bash
cat ~/.claude-memory/coding-style.json
```

ベクトルDBだとこうはいかない。

### 2. Zodで型安全

ツールの引数をZodで定義するので、TypeScriptの恩恵をフルに受けられます。

```typescript
{
  key: z.string().describe("メモリのキー"),
  content: z.string().describe("保存する内容"),
  tags: z.array(z.string()).optional(),
}
```

`describe()`でツールの説明も書けるので、Claudeが使い方を理解しやすい。

### 3. 組み込みツールとの連携

SDK組み込みのファイル読み書きツールと組み合わせると、「この設定ファイルの内容をメモリに保存して」みたいな操作もできます。

### 4. サブエージェントでコンテキスト分離

複雑なタスクのとき、サブエージェントを使ってメモリ操作だけ分離できます。メインのコンテキストを汚さずに済む。

## 完成形のコード

最終的なコードをまとめます。

```typescript
// memory-agent.ts
import {
  query,
  tool,
  createSdkMcpServer,
} from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";
import * as fs from "fs/promises";
import * as path from "path";

const MEMORY_DIR = path.join(process.env.HOME || "", ".claude-memory");

// メモリ保存
const saveMemoryTool = tool(
  "save_memory",
  "重要な情報やコンテキストをメモリに保存する。プロジェクトの方針、ユーザーの好み、学んだことなどを保存できる",
  {
    key: z.string().describe("メモリのキー（例: project-context, user-preferences）"),
    content: z.string().describe("保存する内容（Markdown形式推奨）"),
    tags: z.array(z.string()).optional().describe("検索用タグ（例: ['style', 'typescript']）"),
  },
  async (args) => {
    await fs.mkdir(MEMORY_DIR, { recursive: true });

    const filePath = path.join(MEMORY_DIR, `${args.key}.json`);

    // 既存のメモリがあれば更新
    let existingMemory = null;
    try {
      const data = await fs.readFile(filePath, "utf-8");
      existingMemory = JSON.parse(data);
    } catch {
      // 新規作成
    }

    const memory = {
      key: args.key,
      content: args.content,
      tags: args.tags || [],
      createdAt: existingMemory?.createdAt || new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    };

    await fs.writeFile(filePath, JSON.stringify(memory, null, 2));

    return {
      content: [{
        type: "text",
        text: existingMemory
          ? `メモリを更新しました: ${args.key}`
          : `メモリを保存しました: ${args.key}`
      }],
    };
  }
);

// メモリ取得
const getMemoryTool = tool(
  "get_memory",
  "保存されたメモリを取得する",
  {
    key: z.string().describe("取得するメモリのキー"),
  },
  async (args) => {
    const filePath = path.join(MEMORY_DIR, `${args.key}.json`);

    try {
      const data = await fs.readFile(filePath, "utf-8");
      const memory = JSON.parse(data);
      return {
        content: [{ type: "text", text: memory.content }],
      };
    } catch {
      return {
        content: [{ type: "text", text: `メモリが見つかりません: ${args.key}` }],
      };
    }
  }
);

// メモリ一覧
const listMemoriesTool = tool(
  "list_memories",
  "保存されているメモリの一覧を取得する。タグでフィルタリング可能",
  {
    tag: z.string().optional().describe("フィルタリング用タグ"),
  },
  async (args) => {
    try {
      await fs.mkdir(MEMORY_DIR, { recursive: true });
      const files = await fs.readdir(MEMORY_DIR);
      const memories = [];

      for (const file of files) {
        if (!file.endsWith(".json")) continue;

        const data = await fs.readFile(path.join(MEMORY_DIR, file), "utf-8");
        const memory = JSON.parse(data);

        if (args.tag && !memory.tags.includes(args.tag)) continue;

        memories.push({
          key: memory.key,
          tags: memory.tags,
          preview: memory.content.substring(0, 100) + "...",
          updatedAt: memory.updatedAt,
        });
      }

      if (memories.length === 0) {
        return {
          content: [{ type: "text", text: "保存されているメモリはありません" }],
        };
      }

      return {
        content: [{ type: "text", text: JSON.stringify(memories, null, 2) }],
      };
    } catch (error) {
      return {
        content: [{ type: "text", text: `エラー: ${error}` }],
      };
    }
  }
);

// メモリ削除
const deleteMemoryTool = tool(
  "delete_memory",
  "保存されたメモリを削除する",
  {
    key: z.string().describe("削除するメモリのキー"),
  },
  async (args) => {
    const filePath = path.join(MEMORY_DIR, `${args.key}.json`);

    try {
      await fs.unlink(filePath);
      return {
        content: [{ type: "text", text: `メモリを削除しました: ${args.key}` }],
      };
    } catch {
      return {
        content: [{ type: "text", text: `メモリが見つかりません: ${args.key}` }],
      };
    }
  }
);

// MCPサーバー
const memoryServer = createSdkMcpServer({
  name: "memory-tools",
  version: "1.0.0",
  tools: [saveMemoryTool, getMemoryTool, listMemoriesTool, deleteMemoryTool],
});

// メイン関数
async function runWithMemory(userPrompt: string) {
  const systemPrompt = `
あなたはメモリ管理機能を持つアシスタントです。

## メモリの使い方
- セッション開始時: mcp__memory-tools__list_memories で既存のメモリを確認
- 重要な情報を学んだとき: mcp__memory-tools__save_memory で保存
- 過去の情報が必要なとき: mcp__memory-tools__get_memory で取得

## 保存すべき情報
- プロジェクトの方針やコーディングスタイル
- ユーザーの好みや要望
- 解決した問題とその解決策
- 学んだ教訓やTips
`;

  const result = query({
    prompt: `${systemPrompt}\n\n## ユーザーからのリクエスト\n${userPrompt}`,
    mcpServers: [memoryServer],
    options: {
      maxTurns: 15,
    },
  });

  for await (const message of result) {
    if (message.type === "assistant") {
      console.log(message.content);
    }
  }
}

// 実行例
runWithMemory("まず既存のメモリを確認して、なければこのプロジェクトの方針を保存してください");
```

## 学び・Tips

### 次にやるなら気をつけること

1. **MCPツールの命名規則は最初に確認** - `mcp__サーバー名__ツール名`の形式
2. **ファイルベースならキーごとに分離** - race condition回避
3. **タグ設計は最初に考える** - 後から変えるのは大変

### おすすめの運用パターン

- **プロジェクト固有のメモリ**: `project-{プロジェクト名}`のキーで管理
- **ユーザー設定**: `user-preferences`に好みを保存
- **学習ログ**: `learned-{日付}`で日々の学びを記録

### 参考リソース

- [Agent SDK overview - Claude Docs](https://platform.claude.com/docs/en/agent-sdk/overview)
- [Memory tool (Beta) - Claude Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- [Building agents with the Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)
- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

### 公式のMemory Tool（ベータ版）

実は、Claude APIにはベータ版の[Memory Tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)があります。`context-management-2025-06-27`ヘッダーで有効化できるらしい。自作しなくてよかったかも...でも、カスタマイズできる自作版も価値はあると思います。

## まとめ

### Claude Agent SDKでメモリ管理が向いている場面

- セッション間でコンテキストを保持したい
- プロジェクト固有の知識を蓄積したい
- シンプルなファイルベースの実装で十分
- カスタマイズしたい（タグ、検索、フィルタリングなど）

### 向いていない場面

- 大量のメモリを高速検索したい（ベクトルDB検討）
- 複雑なリレーションが必要（RDB検討）
- 既存のclaude-memで十分な場合

### 最後に

「セッションが切れても文脈を覚えていてほしい」という素朴な願いから始めたプロジェクトでしたが、結果的にClaude Agent SDKの理解が深まりました。

MCPでカスタムツールを作る体験は、AIエージェント開発の第一歩として良い入門になると思います。ファイルベースのシンプルな設計なら、デバッグも楽だし、何が起きているかわかりやすい。

もっと本格的にやるなら公式のMemory Tool（ベータ）や、claude-memプラグインも検討してみてください。

---

この記事で参照したリンク:

- [Agent SDK overview - Claude Docs](https://platform.claude.com/docs/en/agent-sdk/overview)
- [Memory tool (Beta) - Claude Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- [Building agents with the Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)
- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Context is Key - Claude-mem](https://typevar.dev/articles/thedotmack/claude-mem)
- [GitHub - thedotmack/claude-mem](https://github.com/thedotmack/claude-mem)
