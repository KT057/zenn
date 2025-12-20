---
title: "CrewAIでマルチエージェントチームを作ってみた｜役割分担の面白さと想定外のコスト"
emoji: "👥"
type: "tech"
topics: ["crewai", "ai", "python", "agent", "automation"]
published: false
---

## はじめに

「1人のAIじゃなくて、チームで作業させたらもっと賢くなるんじゃない？」

最近よく聞くマルチエージェントという概念。複数のAIエージェントに役割を与えて協力させるというもので、なんかカッコいいですよね。

で、実際どうなの？というのを試したくて、**CrewAI** というフレームワークを使ってみました。技術記事のリサーチから執筆までを「チーム」にやらせてみた話です。

結論から言うと、**役割分担を自然言語で定義できるのが最高に楽しい**。一方で、**APIコストが予想以上にかかる**という現実も。この記事では、そのへんのリアルな体験を共有します。

## なぜ CrewAI を選んだか

マルチエージェントのフレームワークはいくつかあります。

- **AutoGen（Microsoft）**: 会話ベースで柔軟だけど、ちょっと複雑そう
- **LangGraph**: グラフ構造で自由度が高いけど、マルチエージェント特化ではない
- **CrewAI**: 「チーム」というメタファーがわかりやすい

CrewAI を選んだ決め手は、**「役割（role）」「目標（goal）」「背景（backstory）」という人間っぽい設定でエージェントを定義できる**こと。

```python
Agent(
    role="リサーチャー",
    goal="最新のAI技術トレンドを調査する",
    backstory="あなたは10年以上の経験を持つテックアナリストです"
)
```

これ、読むだけでエージェントが何をするか想像できますよね。この直感的さが気に入りました。

参考: [CrewAI 公式ドキュメント](https://docs.crewai.com/)

## 実装してみた

### セットアップ

まずは環境構築。Python 3.10〜3.12が必要です。

```bash
pip install "crewai[tools]"
```

CrewAI は内部で `uv` というパッケージマネージャを使っているらしく、インストールがめちゃくちゃ速いです。これは地味に嬉しいポイント。

### 最初のコード

「リサーチャー」と「ライター」の2人体制で、技術トピックを調査→記事化するチームを作ってみました。

```python
from crewai import Agent, Task, Crew

# リサーチ担当
researcher = Agent(
    role="リサーチャー",
    goal="指定されたトピックについて最新情報を収集し、要点をまとめる",
    backstory="あなたは好奇心旺盛なテックジャーナリストです。正確な情報収集が得意。",
    verbose=True
)

# 執筆担当
writer = Agent(
    role="ライター",
    goal="リサーチ結果を元に、読みやすい技術記事を書く",
    backstory="あなたはエンジニア向けメディアで5年間執筆してきたライターです。",
    verbose=True
)

# タスクを定義
research_task = Task(
    description="CrewAIの最新アップデートと活用事例を調査してください",
    expected_output="箇条書きで5つ以上の要点",
    agent=researcher
)

write_task = Task(
    description="リサーチ結果を元に、500文字程度の紹介記事を書いてください",
    expected_output="読みやすい日本語の記事",
    agent=writer
)

# チームを編成
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task]
)

# 実行
result = crew.kickoff()
print(result)
```

### 動かしてみた結果

`verbose=True` にしておくと、エージェントの思考過程が見えます。

```
[リサーチャー] 思考: まずCrewAIの公式ドキュメントを確認しよう...
[リサーチャー] 行動: web_searchツールを使用
[リサーチャー] 結果: CrewAIはv0.100で大幅に安定化...
```

こんな感じで、**誰が何を考えて行動しているかがリアルタイムでわかる**。これが想像以上に面白い。

## 辛かったポイント

### 1. APIキー設定でハマった

最初に動かそうとしたとき、こんなエラーが出ました。

```
litellm.AuthenticationError: The api_key client option must be set
```

CrewAI はデフォルトで OpenAI の API を使うので、`OPENAI_API_KEY` の環境変数が必要です。`.env` ファイルを作って設定したつもりが読み込まれていなくて、30分くらい悩みました。

```bash
export OPENAI_API_KEY="sk-..."
```

これを `.bashrc` や `.zshrc` に追加するか、`python-dotenv` を使いましょう。

### 2. 想定外のAPIコスト

**これが一番痛かった。**

2回テスト実行しただけで、OpenAI のクレジットが $5 近く消費されていました。マルチエージェントなので、エージェント間のやり取りごとにAPIコールが発生するんですね。

特に `verbose=True` で動かすと、思考過程も含めてトークンを消費するので注意。本番で使う前に、**必ず課金上限を設定**しておくことをおすすめします。

### 3. Gemini使用時の不安定さ

「OpenAI高いからGemini使おう」と思って試したら、500エラーが頻発...。

```python
agent = Agent(
    llm="gemini/gemini-pro",
    # ...
)
```

LiteLLM経由でGeminiを呼ぶと、429（レート制限）や503（サービス利用不可）が結構な頻度で発生しました。2025年初頭時点では、OpenAI か Claude を使うのが安定です。

### 4. ドキュメントがちょっと不親切

「ツールの引数ってどうやって設定するの？」みたいな基本的なことが、ドキュメントからすぐに見つからない。**エージェントが自分で考えて引数を設定する**という仕組みなんですが、それを理解するのに時間がかかりました。

## よかったポイント

### 1. 役割分担の定義が楽しい

`role`、`goal`、`backstory` で自然言語でエージェントを定義できるのが最高。

```python
Agent(
    role="批評家",
    goal="記事の改善点を厳しく指摘する",
    backstory="あなたは辛口で有名な編集者。良い記事を作るためなら容赦しない。"
)
```

こういうキャラ付けができるので、**チーム構成を考えるのがゲームみたいで楽しい**。

### 2. タスク間の連携が自動

あるタスクの出力が、自動的に次のタスクに渡されます。明示的にデータを受け渡すコードを書かなくていいのは楽。

```python
# research_taskの結果が自動的にwrite_taskに渡される
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task]
)
```

### 3. 10行で動く手軽さ

セットアップから動作確認まで、30分もあれば試せます。**マルチエージェントの概念を体験するには最適**なフレームワークだと思います。

## 最終的なコード

少し改良して、コスト意識と安定性を高めたバージョン：

```python
from crewai import Agent, Task, Crew, Process
import os

# 環境変数の確認
if not os.getenv("OPENAI_API_KEY"):
    raise ValueError("OPENAI_API_KEY を設定してください")

researcher = Agent(
    role="リサーチャー",
    goal="指定トピックの最新情報を収集し、重要なポイントを5つにまとめる",
    backstory="テック業界を10年見てきたアナリスト。簡潔さを重視。",
    verbose=True,
    allow_delegation=False  # 他のエージェントに丸投げしない
)

writer = Agent(
    role="ライター",
    goal="リサーチ結果を元に、初心者にもわかりやすい記事を書く",
    backstory="エンジニアブログで人気のライター。専門用語を使わない説明が得意。",
    verbose=True,
    allow_delegation=False
)

research_task = Task(
    description="{topic}について、最新の動向と活用事例を調査してください",
    expected_output="箇条書きで5つの重要ポイント",
    agent=researcher
)

write_task = Task(
    description="リサーチ結果を元に、500文字程度の紹介記事を書いてください",
    expected_output="見出し付きの読みやすい記事",
    agent=writer
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential,  # 順番に実行
    verbose=True
)

# 実行
result = crew.kickoff(inputs={"topic": "AIエージェント"})
print(result)
```

## 学び・Tips

### 次にやるなら気をつけること

1. **最初は `verbose=False` でトークン節約** - 動作確認できたらオフにする
2. **`allow_delegation=False` を明示** - 予期しないエージェント間のやり取りを防ぐ
3. **OpenAI の課金上限を設定** - ダッシュボードから必ず設定

### おすすめリソース

- [CrewAI 公式ドキュメント](https://docs.crewai.com/)
- [DeepLearning.AI の CrewAI コース](https://learn.deeplearning.ai/courses/multi-ai-agent-systems-with-crewai/) - 創業者本人が教えてくれる
- [CrewAI GitHub](https://github.com/crewAIInc/crewAI)

## まとめ

**CrewAI が向いている場面：**
- マルチエージェントの概念を手軽に試したい
- 役割分担が明確なワークフローを自動化したい
- Python で素早くプロトタイプを作りたい

**向いていない場面：**
- APIコストを極限まで抑えたい
- 複雑な条件分岐やループが必要（→ LangGraph の方が向いてる）
- 本番環境での安定稼働が最優先

個人的には、**「マルチエージェントってこういうことか！」を体感するには最高のフレームワーク**だと思います。役割を考えてチームを組むのが純粋に楽しい。

ただ、本番で使うならAPIコストの見積もりをちゃんとやってから。テストで $5 飛んだのは痛かった...。

マルチエージェントに興味があるなら、まずは CrewAI で遊んでみるのがおすすめです。
