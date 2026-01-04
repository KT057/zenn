---
title: "Google ADKで調査レポート自動生成エージェントを作ってみた｜辛かったこと・よかったこと"
emoji: "🔍"
type: "tech"
topics: ["googleadk", "aiagent", "python", "gemini"]
published: false
---

## はじめに

「この技術について調べてレポートまとめといて」

仕事でよくある依頼だと思うんですが、正直これが結構面倒なんですよね。複数のサイトを調べて、情報を整理して、それっぽい形にまとめる。単純作業なんだけど、地味に時間がかかる。

「これ、AIエージェントにやらせたら楽になるんじゃない？」

そう思って、Googleが公開しているAgent Development Kit（ADK）を使って調査レポート自動生成エージェントを作ってみました。今回はその体験談を共有します。

## なぜGoogle ADKを選んだか

AIエージェントのフレームワークって、今いろいろあるじゃないですか。LangChain、CrewAI、OpenAI Agents SDKなど。その中でなぜADKを選んだかというと、理由は3つあります。

### 1. セットアップが簡単そうだった

公式ドキュメントを見たら「20分で動かせる」って書いてあったんですよね。実際、`adk create` コマンドでプロジェクト作って、`adk web` で即座にチャットUIが起動する。これは魅力的でした。

### 2. マルチエージェントが標準サポート

調査→要約→レポート生成という流れを複数のエージェントで分担させたかったので、SequentialAgentやParallelAgentが最初から用意されているのは助かりました。

### 3. Geminiとの相性

普段からGemini使っているので、同じGoogle製のADKなら連携がスムーズだろうという期待がありました。これは実際その通りでした。

## 実装してみた

### 環境構築

まずはADKをインストールします。

```bash
pip install google-adk
```

プロジェクトを作成。

```bash
adk create research_agent
cd research_agent
```

これで以下のような構成ができます。

```
research_agent/
├── agent.py
├── .env
└── __init__.py
```

### APIキーの設定

`.env` ファイルにGemini APIキーを設定します。Google AI Studioから取得できます。

```bash
GOOGLE_API_KEY=your_api_key_here
```

### 最初のエージェント

まずは簡単なエージェントから。`agent.py` を編集します。

```python
from google.adk.agents import Agent

root_agent = Agent(
    name="research_assistant",
    model="gemini-2.0-flash",
    instruction="""
    あなたは調査アシスタントです。
    ユーザーから調査テーマを受け取り、情報をまとめてレポートを作成してください。
    """,
)
```

動かしてみましょう。

```bash
adk web
```

ブラウザで http://localhost:8000 を開くと、チャットUIが表示されます。左上のドロップダウンからエージェントを選んで、会話を始められます。

ここまで10分くらい。確かにセットアップは簡単でした。

## 辛かったポイント

### 1. `root_agent` という変数名じゃないと認識されない

これが最初にハマったところです。

```python
# これだとADK Webが認識しない
my_agent = Agent(...)

# これじゃないとダメ
root_agent = Agent(...)
```

エラーメッセージも出ないので、「なんで動かないんだ？」と30分くらい悩みました。公式ドキュメントをよく読んだら書いてあったんですが、最初は見落としてました。

### 2. ツールのdocstringがめちゃくちゃ重要

カスタムツール（関数）を作ったとき、docstringをちゃんと書かないとエージェントがツールを正しく使ってくれません。

```python
# これだとエージェントが何のツールかわからない
def search(query):
    return search_api(query)

# こう書かないとダメ
def search_web(query: str) -> str:
    """
    指定されたキーワードでWeb検索を行い、検索結果を返します。

    Args:
        query: 検索キーワード

    Returns:
        検索結果のテキスト
    """
    return search_api(query)
```

LLMはdocstringを見て「このツールは何をするか」「いつ使うべきか」を判断するので、説明が雑だと意図しない動作になります。これで2時間くらい溶かしました。

### 3. マルチエージェント間のデータ受け渡し

SequentialAgentで複数のエージェントを順番に実行する場合、前のエージェントの出力を次のエージェントに渡す方法がわかりにくかったです。

最初は「自動でつながるんでしょ？」と思ってたんですが、明示的にデータの流れを定義する必要がありました。公式サンプルを何度も読み直してようやく理解できました。

### 4. 429エラー（APIクォータ制限）

テストを繰り返してたら429エラーが頻発するように。Gemini APIのクォータ制限に引っかかってました。開発中はリクエスト数を意識しないと、すぐ制限に達します。

## よかったポイント

### 1. adk webのUIがめちゃ便利

ローカルでチャットUIが即座に起動するのは本当に便利です。いちいちCLIでテストしなくていいし、会話の流れも確認しやすい。

デバッグ機能もついていて、エージェントがどのツールを呼んだか、何を考えて判断したかが可視化されます。これがなかったら開発効率は半分以下だったと思います。

### 2. カスタムツールの追加が簡単

Python関数を定義して、エージェントの`tools`パラメータに渡すだけ。LangChainみたいに`@tool`デコレータとか`Tool`クラスでラップする必要がなくて、シンプルです。

```python
def get_current_date() -> str:
    """今日の日付を取得します。"""
    from datetime import datetime
    return datetime.now().strftime("%Y年%m月%d日")

root_agent = Agent(
    name="research_assistant",
    model="gemini-2.0-flash",
    instruction="...",
    tools=[get_current_date],
)
```

### 3. ワークフローエージェントの柔軟性

SequentialAgent（順次実行）、ParallelAgent（並列実行）、LoopAgent（繰り返し）が用意されていて、複雑なワークフローも組めます。

今回は「情報収集」→「要約」→「レポート生成」という流れをSequentialAgentで実装しましたが、将来的には複数のソースを並列で調査するようParallelAgentも使いたいなと思ってます。

## 実際のコード

最終的に作った調査レポートエージェントのコードです。

```python
from google.adk.agents import Agent, SequentialAgent
import requests

def search_web(query: str) -> str:
    """
    指定されたキーワードでWeb検索を行い、検索結果を返します。
    調査テーマに関連する情報を収集する際に使用してください。

    Args:
        query: 検索キーワード

    Returns:
        検索結果のテキスト（タイトルとスニペット）
    """
    # 実際にはBrave Search APIなどを使用
    # ここでは簡略化のためダミー実装
    return f"「{query}」の検索結果: ..."

def save_report(title: str, content: str) -> str:
    """
    作成したレポートをファイルに保存します。

    Args:
        title: レポートのタイトル
        content: レポートの本文

    Returns:
        保存結果のメッセージ
    """
    filename = f"report_{title.replace(' ', '_')}.md"
    with open(filename, "w", encoding="utf-8") as f:
        f.write(f"# {title}\n\n{content}")
    return f"レポートを {filename} に保存しました"

# 情報収集エージェント
researcher = Agent(
    name="researcher",
    model="gemini-2.0-flash",
    instruction="""
    あなたは情報収集の専門家です。
    与えられたテーマについて、search_webツールを使って情報を収集してください。
    複数の角度から検索し、できるだけ多くの情報を集めてください。
    収集した情報は整理せず、そのまま出力してください。
    """,
    tools=[search_web],
)

# 要約エージェント
summarizer = Agent(
    name="summarizer",
    model="gemini-2.0-flash",
    instruction="""
    あなたは要約の専門家です。
    前のエージェントが収集した情報を受け取り、重要なポイントを抽出して要約してください。
    箇条書きで整理し、重複する情報は統合してください。
    """,
)

# レポート作成エージェント
report_writer = Agent(
    name="report_writer",
    model="gemini-2.0-flash",
    instruction="""
    あなたはテクニカルライターです。
    要約された情報を元に、読みやすいレポートを作成してください。

    レポートの構成:
    1. 概要
    2. 主要なポイント
    3. 詳細
    4. まとめ

    作成したレポートはsave_reportツールで保存してください。
    """,
    tools=[save_report],
)

# メインエージェント（シーケンシャル実行）
root_agent = SequentialAgent(
    name="research_pipeline",
    sub_agents=[researcher, summarizer, report_writer],
)
```

このコードを`agent.py`に保存して`adk web`を実行すれば、調査テーマを入力するだけでレポートが自動生成されます。

## 学び・Tips

### 1. まずはシンプルなエージェントから始める

最初からマルチエージェントを作ろうとすると、どこで問題が起きてるかわからなくなります。まずは単一エージェント＋1ツールで動作確認してから拡張するのがおすすめです。

### 2. ツールのdocstringは丁寧に書く

「このツールは何をするか」「いつ使うべきか」「引数に何を渡すか」をLLMが理解できるよう、具体的に書きましょう。これをサボると後で泣きます。

### 3. デバッグはadk webのUI上で

CLIで`adk run`も使えますが、デバッグはWeb UIの方が圧倒的に楽です。エージェントの思考過程が可視化されるので、「なんでこのツール呼んだんだ？」みたいな疑問がすぐ解決します。

### 4. APIクォータに注意

開発中は何度もリクエストを投げがちなので、クォータ制限を意識しましょう。必要に応じて`gemini-2.0-flash`など軽量モデルを使うと節約できます。

### 参考リンク

- [ADK公式ドキュメント](https://google.github.io/adk-docs/)
- [ADK Python SDK (GitHub)](https://github.com/google/adk-python)
- [ADKサンプルエージェント集](https://github.com/google/adk-samples)
- [マルチエージェントパターンガイド](https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/)

## まとめ

Google ADKで調査レポート自動生成エージェントを作ってみました。

**向いている場面:**
- Geminiを使ったエージェント開発
- マルチエージェントで複雑なワークフローを組みたいとき
- 素早くプロトタイプを作りたいとき（`adk web`が便利）

**向いていない場面:**
- OpenAI APIをメインで使いたい場合（LiteLlm経由で使えるけど、Geminiほどスムーズではない）
- 本番環境へのデプロイを急いでいる場合（Vertex AI Agent Engineへのデプロイは別途学習コストがかかる）

**こういう人におすすめ:**
- GeminiやGoogle Cloudを普段から使っている人
- 「まずは動くものを作りたい」という人
- マルチエージェントに興味があるけど、複雑なコードは書きたくない人

ADKはまだ発展途上で、2週間ごとにリリースがあるくらい活発に開発されています。ハマりポイントもありましたが、公式ドキュメントがしっかりしているので、詰まっても解決策は見つかりやすいです。

AIエージェント開発に興味がある方は、ぜひ一度試してみてください。
