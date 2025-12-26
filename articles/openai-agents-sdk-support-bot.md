---
title: "OpenAI Agents SDKで問い合わせ振り分けbotを作ってみた｜Handoff機能が想像以上に便利だった話"
emoji: "🤝"
type: "tech"
topics: ["openaiagentssdk", "ai", "python", "chatbot", "multiagent"]
published: false
---

## はじめに

「問い合わせ対応、毎回同じような質問に答えるの面倒だな...」

個人開発でちょっとしたWebサービスを運営しているんですが、問い合わせが増えてくると対応が大変になってきました。料金の質問、使い方の質問、バグ報告...内容によって対応が全然違うので、まず「これは何の問い合わせか」を判断するところから始まる。

「AIで自動振り分けできないかな」と思って調べていたら、2025年3月にOpenAIがAgents SDKをリリースしたという記事を見つけました。複数のAIエージェントを連携させて、タスクを分担できるらしい。

これ、まさに今やりたいことじゃん。

というわけで、OpenAI Agents SDKを使って問い合わせ振り分けbotを作ってみました。この記事では、実際に触ってみてわかった「これは便利」「ここでハマった」というリアルな感想を共有します。

## なぜ OpenAI Agents SDK を選んだか

エージェント系のフレームワークって、正直たくさんありすぎて迷いました。

- **LangChain** - 有名だけど、抽象化が多くて学習コストが高そう
- **CrewAI** - マルチエージェント特化だけど、今回の用途には大げさかも
- **Claude Agent SDK** - TypeScript派の自分には魅力的だけど、今回はPythonで試したい

そこで出てきたのがOpenAI Agents SDK。調べてみると：

- **最小限の抽象化** - Agents、Handoffs、Guardrailsという3つの概念だけ
- **Pythonネイティブ** - 関数をそのままツールとして使える
- **トレーシング機能付き** - デバッグが楽そう

「概念が少ない」というのが決め手でした。LangChainは便利そうだけど、Chains、Agents、Tools、Memory、Callbacks...と覚えることが多くて、最初の一歩が重い。Agents SDKは「まず動かす」までが短そうだったので、試してみることにしました。

## 実装してみた

### セットアップ

まずはインストールから。Python 3.9以上が必要です。

```bash
pip install openai-agents
```

環境変数にAPIキーを設定：

```bash
export OPENAI_API_KEY="sk-..."
```

### 最初に書いたコード

公式のQuickstartを参考に、まずは単純なエージェントを作ってみました。

```python
from agents import Agent, Runner

agent = Agent(
    name="サポートアシスタント",
    instructions="あなたは親切なカスタマーサポートです。"
)

result = Runner.run_sync(agent, "料金プランを教えてください")
print(result.final_output)
```

動いた。シンプル。

でも、これだと普通のChatGPTと変わらない。本題の「振り分け」を実装していきます。

### Handoffで振り分けを実装

OpenAI Agents SDKの目玉機能が **Handoff**（ハンドオフ）です。これは「あるエージェントから別のエージェントにタスクを渡す」機能。

今回作りたいのは：
1. **受付エージェント** - 問い合わせ内容を判断して振り分け
2. **料金担当エージェント** - 料金・プランの質問に回答
3. **技術担当エージェント** - 使い方・バグ報告に対応

```python
from agents import Agent, Runner
import asyncio

# 専門エージェントを定義
billing_agent = Agent(
    name="料金担当",
    instructions="""
    あなたは料金・プラン担当です。
    - 無料プラン: 月10回まで利用可能
    - Proプラン: 月額980円、無制限
    - Teamプラン: 月額2,980円、5人まで
    これらの情報をもとに、丁寧に回答してください。
    """
)

tech_agent = Agent(
    name="技術担当",
    instructions="""
    あなたは技術サポート担当です。
    使い方の質問やバグ報告に対応します。
    具体的な解決策を提示し、必要に応じてドキュメントを案内してください。
    """
)

# 受付エージェント（振り分け役）
triage_agent = Agent(
    name="受付",
    instructions="""
    あなたは問い合わせ受付担当です。
    ユーザーの質問内容を判断し、適切な担当に引き継いでください。
    - 料金、プラン、支払いに関する質問 → 料金担当
    - 使い方、エラー、バグに関する質問 → 技術担当
    """,
    handoffs=[billing_agent, tech_agent]
)

async def main():
    # 料金の質問
    result = await Runner.run(triage_agent, "Proプランの料金を教えてください")
    print(f"回答: {result.final_output}")

    # 技術的な質問
    result = await Runner.run(triage_agent, "ログインできないのですが...")
    print(f"回答: {result.final_output}")

asyncio.run(main())
```

## 辛かったポイント

### 1. async/awaitが必須

最初、`Runner.run_sync()`で試していたんですが、Handoffを使うと挙動が不安定になることがありました。公式ドキュメントをよく読むと、**async/awaitの理解が前提**と書いてある。

Pythonの非同期処理に慣れていないと、ここで「あれ、動かない」ってなるかもしれません。自分も`asyncio.run()`を毎回書くのが面倒で、最初は戸惑いました。

```python
# これだと動くけど、非同期処理の恩恵が少ない
result = Runner.run_sync(agent, "質問")

# 本来はこう書くべき
async def main():
    result = await Runner.run(agent, "質問")
asyncio.run(main())
```

### 2. Handoffのデバッグが難しい

これが一番辛かった。Handoffでどのエージェントにタスクが渡されたか、**外から見えにくい**んです。

「料金の質問をしたのに、なぜか技術担当に行ってしまった」というとき、なぜそうなったのかを追うのが大変。LLMが判断しているので、deterministic（決定的）じゃないんですよね。

解決策としては、**トレーシング機能**を使います。

```python
from agents.tracing import set_tracing_export_api_key

# OpenAIのダッシュボードでトレースを確認できる
set_tracing_export_api_key("sk-...")
```

これでOpenAIのダッシュボード上で「どのエージェントがどの順番で呼ばれたか」が可視化されます。正直、これがなかったら詰んでた。

### 3. instructionsの書き方で挙動が変わりすぎる

受付エージェントの`instructions`を少し変えただけで、振り分けの精度がガラッと変わります。

最初は「適切な担当に引き継いでください」としか書いてなくて、曖昧な質問が全部技術担当に行ってしまっていました。

**Before（精度低い）：**
```python
instructions="問い合わせを適切な担当に引き継いでください"
```

**After（精度改善）：**
```python
instructions="""
あなたは問い合わせ受付担当です。
ユーザーの質問内容を判断し、適切な担当に引き継いでください。

【振り分けルール】
- 以下のキーワードが含まれる場合は料金担当へ:
  料金、プラン、価格、支払い、請求、解約、無料、有料
- 以下のキーワードが含まれる場合は技術担当へ:
  使い方、エラー、バグ、動かない、ログイン、設定

迷った場合は、まずユーザーに確認してください。
"""
```

プロンプトエンジニアリングの重要性を痛感しました。

## よかったポイント

### 1. 関数がそのままツールになる

これ、地味にめちゃくちゃ便利です。Pythonの関数を書くだけで、エージェントが使えるツールになります。

```python
from agents import Agent, Runner, function_tool

@function_tool
def get_user_plan(user_id: str) -> str:
    """ユーザーの契約プランを取得します"""
    # 実際はDBから取得
    plans = {"user_123": "Pro", "user_456": "Free"}
    return plans.get(user_id, "不明")

billing_agent = Agent(
    name="料金担当",
    instructions="ユーザーの契約プランを確認して回答してください",
    tools=[get_user_plan]
)
```

LangChainだと`Tool`クラスを継承して...とか書かないといけないのに対して、デコレータ一発。Pydanticでバリデーションも自動でやってくれるのも嬉しい。

### 2. コード量が少ない

上で書いた振り分けbot、コア部分は30行くらいしかありません。これくらいシンプルだと、「とりあえず動くもの」をサクッと作れます。

プロトタイプを作って上司に見せる、みたいなシーンで重宝しそう。

### 3. OpenAIエコシステムとの親和性

当然ですが、OpenAIのモデルとの相性は抜群です。GPT-4o、GPT-4o-mini、o1など、用途に応じてモデルを切り替えられます。

```python
from agents import Agent, OpenAIChatCompletionsModel

agent = Agent(
    name="高精度エージェント",
    model=OpenAIChatCompletionsModel(model="gpt-4o"),
    instructions="..."
)
```

## 完成形のコード

最終的にはこんな感じになりました。

```python
from agents import Agent, Runner, function_tool
import asyncio

# ツール定義
@function_tool
def get_user_info(user_id: str) -> dict:
    """ユーザー情報を取得します"""
    users = {
        "user_123": {"name": "山田太郎", "plan": "Pro", "joined": "2024-01"},
        "user_456": {"name": "佐藤花子", "plan": "Free", "joined": "2025-01"},
    }
    return users.get(user_id, {"error": "ユーザーが見つかりません"})

@function_tool
def create_ticket(title: str, description: str, priority: str) -> str:
    """サポートチケットを作成します"""
    # 実際はDBに保存
    return f"チケット作成完了: {title} (優先度: {priority})"

# 専門エージェント
billing_agent = Agent(
    name="料金担当",
    instructions="""
    あなたは料金・プラン担当のサポートスタッフです。

    【プラン情報】
    - Free: 無料、月10回まで
    - Pro: 月額980円、無制限利用
    - Team: 月額2,980円、5人までのチーム利用

    ユーザーIDがわかる場合は、get_user_infoで契約情報を確認してください。
    """,
    tools=[get_user_info]
)

tech_agent = Agent(
    name="技術担当",
    instructions="""
    あなたは技術サポート担当です。

    【よくある問題と解決策】
    - ログインできない → パスワードリセットを案内
    - 動作が遅い → キャッシュクリアを提案
    - エラーが出る → エラーメッセージを確認し、チケット作成

    解決できない場合は、create_ticketでチケットを作成してください。
    """,
    tools=[create_ticket]
)

# 振り分けエージェント
triage_agent = Agent(
    name="受付",
    instructions="""
    あなたは問い合わせ受付担当です。
    ユーザーの質問を分析し、適切な専門担当に引き継いでください。

    【振り分けルール】
    - 料金、プラン、支払い、請求、解約 → 料金担当
    - 使い方、エラー、バグ、設定、ログイン → 技術担当

    曖昧な場合は、まずユーザーに「料金についてですか？それとも使い方についてですか？」と確認してください。
    """,
    handoffs=[billing_agent, tech_agent]
)

async def handle_inquiry(message: str):
    """問い合わせを処理"""
    result = await Runner.run(triage_agent, message)
    return result.final_output

# 使用例
async def main():
    inquiries = [
        "Proプランに変更したいのですが",
        "ログインできなくなりました。user_123です",
        "最近サービスが重いです",
    ]

    for inquiry in inquiries:
        print(f"\n📩 問い合わせ: {inquiry}")
        response = await handle_inquiry(inquiry)
        print(f"💬 回答: {response}")

if __name__ == "__main__":
    asyncio.run(main())
```

## 学び・Tips

### 1. instructionsは具体的に書く

「適切に対応して」みたいな曖昧な指示はNG。**具体的なルール、キーワード、例**を書くと精度が上がります。

### 2. トレーシングは最初から有効にしておく

デバッグで詰まったときに「あのとき有効にしておけば...」となるので、開発中は常にONにしておくのがおすすめ。

### 3. Handoffは「専門家チーム」のメタファーで考える

「この質問はAさんが得意だからAさんに聞こう」という人間の判断を、LLMにやらせているイメージ。各エージェントの「専門分野」を明確にしておくと、振り分け精度が上がります。

### 参考リンク

- [OpenAI Agents SDK 公式ドキュメント](https://openai.github.io/openai-agents-python/)
- [GitHub - openai/openai-agents-python](https://github.com/openai/openai-agents-python)
- [Handoffs - OpenAI Agents SDK](https://openai.github.io/openai-agents-python/handoffs/)

## まとめ

OpenAI Agents SDKを使って問い合わせ振り分けbotを作ってみました。

**向いている場面：**
- OpenAIモデルを使った開発をしている
- 複数エージェントの連携を試したい
- とにかくシンプルに始めたい
- Pythonで書きたい

**向いていない場面：**
- OpenAI以外のモデルをメインで使いたい
- 複雑な状態管理が必要（LangGraphの方が向いてる）
- TypeScript/JavaScriptで書きたい（今後対応予定らしい）

**こういう人におすすめ：**
- 「エージェント開発、まず何から始めればいい？」という人
- LangChainの抽象化に疲れた人
- ChatGPTの次のステップを探している人

Handoff機能、想像以上に便利でした。「まず受付→専門家に振り分け」というパターンは、カスタマーサポート以外にも応用が効きそうです。社内のヘルプデスクbot、技術的な質問を担当別に振り分けるSlackbot...アイデアが広がります。

OpenAI Agents SDK、シンプルに始められるのでぜひ試してみてください。
