---
title: "Pydantic AIで日報から感情分析するエージェントを作ってみた｜型安全の安心感がすごい"
emoji: "🎯"
type: "tech"
topics: ["pydanticai", "ai", "python", "llm", "typedhint"]
published: false
---

## はじめに

「LLMの出力、たまに変な形式で返ってきて困るんだよな...」

AIを使ったアプリを作っていると、LLMの出力が安定しない問題に悩まされます。JSONで返してほしいのに、余計な説明文が混じってたり、フィールド名が微妙に違ってたり。毎回パースで苦労する。

そんなとき見つけたのが **Pydantic AI**。Pydanticの開発元が作ったLLMエージェントフレームワークで、「型安全な構造化出力」が売り。FastAPIでPydanticを使い慣れている自分には、かなり刺さるコンセプトでした。

試しに、社内日報からモチベーションや感情を分析するエージェントを作ってみました。この記事では、実際に触ってみた感想を共有します。

## なぜ Pydantic AI を選んだか

LLMエージェントのフレームワークって、正直たくさんありすぎて迷いました。

- **LangChain** - 機能が豊富すぎて、学習コストが高い。300MB近いパッケージ群も重い
- **OpenAI Agents SDK** - シンプルだけど、構造化出力の型安全性はそこまで強くない
- **CrewAI** - マルチエージェント向けで、今回の用途にはオーバースペック

そこで見つけたのがPydantic AI。調べてみると：

- **Pydanticベース** - FastAPIで慣れ親しんだ書き方がそのまま使える
- **型安全な出力** - BaseModelで定義すれば、LLMの出力を自動バリデーション
- **シンプル** - 概念が少なく、すぐ動かせる
- **マルチモデル対応** - OpenAI、Anthropic、Geminiなど幅広くサポート

「FastAPIの感覚をGenAIアプリにも」というコンセプトに惹かれて、試してみることにしました。

## 実装してみた

### セットアップ

インストールはシンプル。

```bash
pip install pydantic-ai
```

Anthropicを使う場合：

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

### 最初に書いたコード

まずは公式ドキュメントの例をそのまま動かしてみました。

```python
from pydantic_ai import Agent

agent = Agent(
    'anthropic:claude-sonnet-4-0',
    instructions='Be concise, reply with one sentence.',
)

result = agent.run_sync('Where does "hello world" come from?')
print(result.output)
```

動いた。シンプル。

でも、これだと普通にChat APIを叩いてるのと変わらない。本題の「構造化出力」を試していきます。

### 日報分析エージェントを作る

今回作りたいのは、日報テキストから以下を抽出するエージェント：

1. **達成したタスク**のリスト
2. **課題・困っていること**
3. **モチベーションスコア**（1〜10）
4. **全体的な感情**（ポジティブ/ニュートラル/ネガティブ）

Pydanticの`BaseModel`で出力の型を定義します。

```python
from typing import Literal
from pydantic import BaseModel, Field
from pydantic_ai import Agent


class DailyReportAnalysis(BaseModel):
    """日報分析の結果"""

    completed_tasks: list[str] = Field(
        description="達成したタスクのリスト"
    )
    challenges: list[str] = Field(
        description="課題・困っていること"
    )
    motivation_score: int = Field(
        ge=1, le=10,
        description="モチベーションスコア（1-10）"
    )
    overall_sentiment: Literal["positive", "neutral", "negative"] = Field(
        description="全体的な感情"
    )


agent = Agent(
    'anthropic:claude-sonnet-4-0',
    output_type=DailyReportAnalysis,
    instructions="""
    あなたは日報を分析するアシスタントです。
    日報の内容から、達成タスク、課題、モチベーション、感情を抽出してください。
    モチベーションスコアは文面のトーンから推測してください。
    """
)
```

これで、エージェントの出力が`DailyReportAnalysis`型に自動でバリデーションされます。

### 実行してみる

```python
daily_report = """
今日の作業内容：
- ユーザー認証機能の実装完了
- APIエンドポイントのテスト作成
- コードレビューで指摘された箇所を修正

困っていること：
- デプロイ周りでエラーが出ていて、原因がわからない
- ドキュメントが古くて参考にならない

明日は本番環境へのデプロイを予定しています。
デプロイエラーが解決できるか少し不安ですが、頑張ります。
"""

result = agent.run_sync(daily_report)
analysis = result.output

print(f"達成タスク: {analysis.completed_tasks}")
print(f"課題: {analysis.challenges}")
print(f"モチベーション: {analysis.motivation_score}/10")
print(f"感情: {analysis.overall_sentiment}")
```

出力：

```
達成タスク: ['ユーザー認証機能の実装完了', 'APIエンドポイントのテスト作成', 'コードレビュー指摘箇所の修正']
課題: ['デプロイ周りでエラーが出ていて原因不明', 'ドキュメントが古くて参考にならない']
モチベーション: 6/10
感情: neutral
```

ちゃんと構造化されたデータが返ってきた。しかも`analysis.motivation_score`のように、型付きでアクセスできる。IDEの補完も効く。これは気持ちいい。

## 辛かったポイント

### 1. 非同期処理がデフォルト

Pydantic AIは非同期処理が基本設計。`run_sync()`という同期版もあるけど、内部的には`asyncio.run()`を呼んでいるだけ。

Jupyterノートブックで試そうとしたら、イベントループの競合でエラーが出ました。

```python
# Jupyterではこれがエラーになる
result = agent.run_sync("テスト")
```

解決策として、`nest_asyncio`を使うか、ちゃんと非同期で書く必要があります。

```python
import nest_asyncio
nest_asyncio.apply()

# これで動く
result = agent.run_sync("テスト")
```

または：

```python
import asyncio

async def main():
    result = await agent.run("テスト")
    print(result.output)

asyncio.run(main())
```

### 2. バリデーションエラーのデバッグが難しい

LLMが返した値がPydanticのバリデーションに通らないと、エラーになります。これ自体は正しい動作なんですが、「なぜ通らなかったか」を追うのが大変。

例えば、`motivation_score`が1〜10の範囲外だったり、`overall_sentiment`が定義した選択肢以外の値だったりすると、バリデーションエラーが発生します。

```python
# こういうエラーが出る
pydantic_core._pydantic_core.ValidationError: 1 validation error for DailyReportAnalysis
motivation_score
  Input should be less than or equal to 10 [type=less_than_equal, input_value=11, input_type=int]
```

対策としては：

1. `Field`の`description`を詳しく書いて、LLMに制約を明示する
2. `instructions`でも制約を繰り返し伝える
3. それでもダメなら、バリデーションを緩くする（あまりおすすめしないけど）

```python
motivation_score: int = Field(
    ge=1, le=10,
    description="モチベーションスコア。必ず1から10の整数で返してください。"
)
```

### 3. ツール定義の依存性注入

エージェントにツール（外部関数）を追加するとき、依存性注入の概念が出てきます。FastAPIの`Depends`に似てるんですが、最初は戸惑いました。

```python
from pydantic_ai import Agent, RunContext

agent = Agent(
    'anthropic:claude-sonnet-4-0',
    deps_type=str,  # 依存として渡す型
)

@agent.tool
def get_employee_info(ctx: RunContext[str], employee_id: str) -> dict:
    """従業員情報を取得"""
    # ctx.deps で依存を取得できる
    api_key = ctx.deps
    # ... API呼び出し
    return {"name": "田中太郎", "department": "開発部"}
```

`RunContext`のジェネリクスとか、`deps_type`の指定とか、慣れるまで時間がかかりました。

## よかったポイント

### 1. 型安全の安心感

これが一番大きい。`output_type`に`BaseModel`を指定するだけで、LLMの出力が自動バリデーションされる。

```python
# この時点で型が確定している
analysis: DailyReportAnalysis = result.output

# IDEの補完が効く
print(analysis.motivation_score)  # int型として認識
print(analysis.overall_sentiment)  # Literal型として認識
```

LangChainだと、出力パーサーを別途設定したり、手動でパースしたりする必要があった。Pydantic AIはそこがスッキリしている。

### 2. コード量が少ない

上で書いた日報分析エージェント、コア部分は20行くらい。モデル定義を含めても40行程度。

LangChainで同じことをやろうとすると、`OutputParser`、`PromptTemplate`、`Chain`...と複数のコンポーネントを組み合わせる必要がある。Pydantic AIはそこがシンプル。

### 3. Pydanticとの親和性

FastAPIやその他Pydanticベースのプロジェクトとの相性が抜群。既存のモデル定義をそのまま使える。

```python
# 既存のPydanticモデルがそのまま使える
from my_models import DailyReportAnalysis

agent = Agent(
    'anthropic:claude-sonnet-4-0',
    output_type=DailyReportAnalysis,
)
```

## 完成形のコード

最終的にはこんな感じになりました。

```python
from typing import Literal
from pydantic import BaseModel, Field
from pydantic_ai import Agent


class DailyReportAnalysis(BaseModel):
    """日報分析の結果"""

    completed_tasks: list[str] = Field(
        description="達成したタスクのリスト。箇条書きの項目を抽出。"
    )
    challenges: list[str] = Field(
        description="課題・困っていること。問題や障害を抽出。"
    )
    motivation_score: int = Field(
        ge=1, le=10,
        description="モチベーションスコア（1-10）。文面のトーンから推測。"
    )
    overall_sentiment: Literal["positive", "neutral", "negative"] = Field(
        description="全体的な感情。positive/neutral/negativeのいずれか。"
    )
    summary: str = Field(
        description="日報の要約（50文字以内）"
    )


agent = Agent(
    'anthropic:claude-sonnet-4-0',
    output_type=DailyReportAnalysis,
    instructions="""
    あなたは日報を分析するアシスタントです。

    以下の観点で日報を分析してください：
    1. 達成したタスクを箇条書きから抽出
    2. 困っていること・課題を抽出
    3. 文面のトーンからモチベーションを1-10で評価
       - 10: とても前向き、達成感がある
       - 5: 普通、特に感情が見えない
       - 1: ネガティブ、困っている
    4. 全体的な感情をpositive/neutral/negativeで判定
    5. 50文字以内で要約

    日報の内容を正確に反映してください。
    """,
)


def analyze_daily_report(report_text: str) -> DailyReportAnalysis:
    """日報を分析する"""
    result = agent.run_sync(report_text)
    return result.output


# 使用例
if __name__ == "__main__":
    sample_report = """
    本日の作業：
    - 新機能のプロトタイプ作成（完了）
    - チームミーティングでデモ実施
    - バグ修正3件

    良かったこと：
    - デモがうまくいって、チームから好評だった

    改善点：
    - テストコードがまだ書けていない

    明日の予定：
    - テストコード追加
    - ドキュメント整備
    """

    analysis = analyze_daily_report(sample_report)

    print("=== 日報分析結果 ===")
    print(f"達成タスク: {', '.join(analysis.completed_tasks)}")
    print(f"課題: {', '.join(analysis.challenges)}")
    print(f"モチベーション: {analysis.motivation_score}/10")
    print(f"感情: {analysis.overall_sentiment}")
    print(f"要約: {analysis.summary}")
```

## 学び・Tips

### 1. Fieldのdescriptionは具体的に

LLMに制約を伝えるために、`Field`の`description`を詳しく書くのが重要。曖昧だとバリデーションエラーが増える。

```python
# NG: 曖昧
score: int = Field(description="スコア")

# OK: 具体的
score: int = Field(
    ge=1, le=10,
    description="モチベーションスコア。1（低い）〜10（高い）の整数で評価。"
)
```

### 2. Logfireでトレーシング

Pydantic AIは同じ開発元の**Logfire**と連携できます。デバッグ時に「LLMが何を返したか」「バリデーションでどこで落ちたか」が可視化されて便利。

```python
import logfire

logfire.configure()
logfire.instrument_pydantic_ai()
```

### 3. 複数の出力タイプを切り替える

`output_type`にUnion型を使えば、LLMが文脈に応じて出力型を選べる。

```python
from typing import Union

class SuccessResult(BaseModel):
    message: str
    data: dict

class ErrorResult(BaseModel):
    error: str
    code: int

agent = Agent(
    'anthropic:claude-sonnet-4-0',
    output_type=Union[SuccessResult, ErrorResult],
)
```

### 参考リンク

- [Pydantic AI 公式ドキュメント](https://ai.pydantic.dev/)
- [GitHub - pydantic/pydantic-ai](https://github.com/pydantic/pydantic-ai)
- [Function Tools - Pydantic AI](https://ai.pydantic.dev/tools/)
- [Structured Output - Pydantic AI](https://ai.pydantic.dev/output/)

## まとめ

Pydantic AIで日報分析エージェントを作ってみました。

**向いている場面：**
- 型安全なLLM出力が欲しい
- FastAPIなどPydanticに慣れている
- シンプルに始めたい
- 構造化データの抽出・変換がメイン

**向いていない場面：**
- 複雑なマルチエージェント連携（CrewAIの方が向いてる）
- 非Pythonで開発したい
- Pydanticに馴染みがない（学習コストがかかる）

**こういう人におすすめ：**
- 「LLMの出力パースで消耗している」人
- FastAPIでバックエンドを書いている人
- 型安全が好きな人

正直、LangChainよりずっとシンプルで使いやすかったです。「フレームワーク」というより「ライブラリ」に近い軽さ。余計な抽象化がなくて、Pythonの素直なコードが書ける。

Pydantic AI、型安全が好きな人には本当におすすめです。
