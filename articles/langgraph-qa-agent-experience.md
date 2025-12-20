---
title: "LangGraphで質問回答エージェントを作ってみた｜LangChainとの違いで混乱した話"
emoji: "🔀"
type: "tech"
topics: ["langgraph", "langchain", "python", "ai", "rag"]
published: false
---

## はじめに

「RAGで検索できるのはわかった。でも、検索結果がイマイチだったら自動でリトライしてほしいな...」

LangChainでRAG（検索拡張生成）を組んでいると、こういう欲が出てきますよね。検索結果の品質が低かったら別のクエリで再検索するとか、ユーザーの質問が曖昧だったら聞き返すとか。

LangChainの直線的なチェーンだと、こういう「条件分岐」や「ループ」が書きにくい。そこで出てくるのが **LangGraph** です。

この記事では、LangGraphで質問回答エージェントを作ってみた体験を共有します。「LangChainとの違いがよくわからん」「実際どこでハマるの？」という人の参考になれば。

## なぜ LangGraph を選んだか

### LangChainの限界

LangChainは便利なんですが、構造が基本的に **一方通行** なんですよね。

```
入力 → 処理A → 処理B → 処理C → 出力
```

この直線的な流れ（DAG: 有向非巡回グラフ）では、「処理Bの結果がダメだったら処理Aに戻る」みたいなことがやりづらい。

### LangGraphの強み

LangGraphは **ループや分岐が書ける** のが最大の特徴です。

```
入力 → 検索 → 評価 → （品質OK?）
                    ├─ Yes → 回答生成 → 出力
                    └─ No → クエリ修正 → 検索に戻る
```

こういう「状態マシン」的なフローが自然に書けます。

ちなみにLangGraphはLangChainの「置き換え」ではなく「拡張」です。LangChainのコンポーネント（プロンプト、ツール、モデル）はそのまま使えるので、既存資産を活かしつつ複雑なフローが組めます。

公式サイト: [LangGraph](https://www.langchain.com/langgraph)

## 実装してみた

### セットアップ

まずはインストール。LangGraphとLangChainの両方が必要です。

```bash
pip install langgraph langchain langchain-openai
```

### 基本構造を理解する

LangGraphの主要な構成要素は4つ。

| 要素 | 役割 |
|------|------|
| **State** | グラフ全体で共有する状態（辞書的なもの） |
| **Node** | 実際の処理を行う関数 |
| **Edge** | ノード間の接続 |
| **Graph** | ノードとエッジをまとめたもの |

### 最初のコード

最小限のグラフはこんな感じ。

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

# 状態の定義
class State(TypedDict):
    question: str
    answer: str

# ノードの定義（ただの関数）
def answer_question(state: State) -> State:
    # 実際はここでLLMを呼ぶ
    return {"answer": f"回答: {state['question']}について..."}

# グラフの構築
graph = StateGraph(State)
graph.add_node("answer", answer_question)
graph.add_edge(START, "answer")
graph.add_edge("answer", END)

# コンパイルして実行
app = graph.compile()
result = app.invoke({"question": "LangGraphとは？", "answer": ""})
print(result["answer"])
```

シンプルですよね。ノードは普通のPython関数で、Stateを受け取ってStateを返すだけ。

### 条件分岐を追加する

ここからがLangGraphの本領発揮。検索結果の品質を評価して、ダメならリトライするフローを組みます。

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Literal

class State(TypedDict):
    question: str
    search_query: str
    search_results: str
    quality_score: float
    answer: str
    retry_count: int

def generate_query(state: State) -> State:
    # 質問から検索クエリを生成
    return {"search_query": state["question"]}

def search_documents(state: State) -> State:
    # ドキュメント検索（実際はベクトルDBを叩く）
    return {"search_results": "検索結果...", "retry_count": state.get("retry_count", 0)}

def evaluate_results(state: State) -> State:
    # 検索結果の品質を評価（実際はLLMで判定）
    score = 0.8  # 仮の値
    return {"quality_score": score}

def refine_query(state: State) -> State:
    # クエリを改善して再検索
    return {
        "search_query": f"改善: {state['search_query']}",
        "retry_count": state["retry_count"] + 1
    }

def generate_answer(state: State) -> State:
    return {"answer": f"回答: {state['search_results']}に基づいて..."}

# 分岐のロジック
def should_retry(state: State) -> Literal["refine", "answer"]:
    if state["quality_score"] < 0.7 and state["retry_count"] < 3:
        return "refine"
    return "answer"

# グラフ構築
graph = StateGraph(State)
graph.add_node("generate_query", generate_query)
graph.add_node("search", search_documents)
graph.add_node("evaluate", evaluate_results)
graph.add_node("refine", refine_query)
graph.add_node("answer", generate_answer)

graph.add_edge(START, "generate_query")
graph.add_edge("generate_query", "search")
graph.add_edge("search", "evaluate")

# 条件付きエッジ（これがLangGraphの肝！）
graph.add_conditional_edges(
    "evaluate",
    should_retry,
    {
        "refine": "refine",
        "answer": "answer"
    }
)

graph.add_edge("refine", "search")  # ループ！
graph.add_edge("answer", END)

app = graph.compile()
```

`add_conditional_edges` がポイント。評価結果に応じて次のノードを動的に決められます。そして `refine → search` のエッジでループを実現。

これ、LangChainだけでやろうとすると結構つらいんですよね。

## 辛かったポイント

正直、最初はかなり混乱しました。

### 1. VSCodeでインポートエラーが出る

LangGraphを書いていると、VSCodeで赤線がめっちゃ出る。

```python
from langgraph.graph import StateGraph  # ← 赤線
```

エディタ上では「モジュールが見つからない」と言われるけど、実行すると普通に動く。

原因は **Pylance（VSCodeの型チェッカー）がLangChain系のモジュール構造を認識できない** こと。

`settings.json` に以下を追加すると解決します。

```json
{
  "python.analysis.extraPaths": [
    "./venv/lib/python3.11/site-packages"
  ]
}
```

地味にストレスだったので、早めに設定しておくことをおすすめします。

### 2. 循環参照エラーにハマった

「Circular reference detected」というエラーで1時間くらい溶かしました。

原因は State の中で **双方向参照** を作ってしまっていたこと。

```python
# これがダメだった
class State(TypedDict):
    current_node: "Node"  # ノードへの参照

class Node:
    state: State  # 状態への参照（循環！）
```

解決策は **参照を一方向にする** こと。

```python
# 参照ではなくIDで持つ
class State(TypedDict):
    current_node_id: str  # 参照ではなくIDで持つ
```

LangGraphの状態はシリアライズ（JSON化）されることがあるので、シンプルなデータ型を使うのが安全です。

### 3. LangChainとの使い分けがわからない

最初「LangGraphを使うならLangChainいらないんじゃない？」と思ってました。

実際は **両方使う** のが正解。

- **LangChain**: プロンプト、LLM呼び出し、ツール定義など「部品」を作る
- **LangGraph**: それらの部品を「ワークフロー」として組み立てる

LangGraphはオーケストレーション層で、実際の処理はLangChainのコンポーネントを使うイメージ。

## よかったポイント

辛いことばかり書きましたが、慣れると本当に便利です。

### 1. 条件分岐がめちゃくちゃ書きやすい

`add_conditional_edges` 一発で分岐が書けるの、本当に楽。

```python
graph.add_conditional_edges(
    "node_a",
    lambda state: "yes" if state["score"] > 0.5 else "no",
    {"yes": "node_b", "no": "node_c"}
)
```

if文を書く感覚で複雑なフローが組めます。

### 2. 状態管理を気にしなくていい

各ノードは「今の状態を受け取って、更新した部分だけ返す」という設計。

```python
def my_node(state: State) -> State:
    # 変更した部分だけ返せばOK
    return {"answer": "新しい回答"}
```

全体の状態はLangGraphが自動でマージしてくれるので、ノード側で気にする必要がない。これはマルチエージェントとかで状態が複雑になってきたときに助かります。

### 3. デバッグしやすい

「今どのノードで止まっているか」がわかりやすい。

```python
# 実行中の状態を取得
for event in app.stream({"question": "テスト"}):
    print(f"ノード: {list(event.keys())[0]}")
    print(f"状態: {event}")
```

`stream` メソッドで各ノードの実行結果を逐次取得できるので、どこで問題が起きているか特定しやすいです。

LangGraph Studio を使えば、グラフ構造をGUIで可視化しながらデバッグもできます。

## 完成形のコード

最終的にこんな感じになりました。実際にRAGとして使える形です。

```python
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from typing import TypedDict, Literal
import os

# LLMの設定
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class State(TypedDict):
    question: str
    search_query: str
    search_results: str
    quality_score: float
    answer: str
    retry_count: int

def generate_query(state: State) -> State:
    """ユーザーの質問から検索クエリを生成"""
    prompt = ChatPromptTemplate.from_messages([
        ("system", "ユーザーの質問を検索クエリに変換してください。"),
        ("user", "{question}")
    ])
    chain = prompt | llm
    result = chain.invoke({"question": state["question"]})
    return {"search_query": result.content, "retry_count": 0}

def search_documents(state: State) -> State:
    """ドキュメント検索（実際はベクトルDBを使用）"""
    # ここでChromaやPineconeを叩く
    mock_results = f"検索結果: '{state['search_query']}' に関連するドキュメント..."
    return {"search_results": mock_results}

def evaluate_results(state: State) -> State:
    """検索結果の品質を評価"""
    prompt = ChatPromptTemplate.from_messages([
        ("system", "検索結果が質問に対して十分かを0-1のスコアで評価してください。数値のみ返してください。"),
        ("user", "質問: {question}\n検索結果: {search_results}")
    ])
    chain = prompt | llm
    result = chain.invoke({
        "question": state["question"],
        "search_results": state["search_results"]
    })
    try:
        score = float(result.content.strip())
    except ValueError:
        score = 0.5
    return {"quality_score": score}

def refine_query(state: State) -> State:
    """検索クエリを改善"""
    prompt = ChatPromptTemplate.from_messages([
        ("system", "検索結果が不十分でした。より良い検索クエリを生成してください。"),
        ("user", "元のクエリ: {query}\n質問: {question}")
    ])
    chain = prompt | llm
    result = chain.invoke({
        "query": state["search_query"],
        "question": state["question"]
    })
    return {
        "search_query": result.content,
        "retry_count": state["retry_count"] + 1
    }

def generate_answer(state: State) -> State:
    """最終回答を生成"""
    prompt = ChatPromptTemplate.from_messages([
        ("system", "検索結果に基づいて質問に回答してください。"),
        ("user", "質問: {question}\n参考情報: {search_results}")
    ])
    chain = prompt | llm
    result = chain.invoke({
        "question": state["question"],
        "search_results": state["search_results"]
    })
    return {"answer": result.content}

def should_retry(state: State) -> Literal["refine", "answer"]:
    """リトライするかどうかを判定"""
    if state["quality_score"] < 0.7 and state["retry_count"] < 3:
        return "refine"
    return "answer"

# グラフ構築
def create_qa_agent():
    graph = StateGraph(State)

    # ノード追加
    graph.add_node("generate_query", generate_query)
    graph.add_node("search", search_documents)
    graph.add_node("evaluate", evaluate_results)
    graph.add_node("refine", refine_query)
    graph.add_node("answer", generate_answer)

    # エッジ追加
    graph.add_edge(START, "generate_query")
    graph.add_edge("generate_query", "search")
    graph.add_edge("search", "evaluate")
    graph.add_conditional_edges("evaluate", should_retry, {
        "refine": "refine",
        "answer": "answer"
    })
    graph.add_edge("refine", "search")
    graph.add_edge("answer", END)

    return graph.compile()

# 実行
if __name__ == "__main__":
    agent = create_qa_agent()
    result = agent.invoke({
        "question": "LangGraphとLangChainの違いは？",
        "search_query": "",
        "search_results": "",
        "quality_score": 0.0,
        "answer": "",
        "retry_count": 0
    })
    print(result["answer"])
```

## 学び・Tips

### 次にやるなら気をつけること

1. **Stateはシンプルに保つ** - 複雑なオブジェクトより、プリミティブ型や辞書を使う
2. **ループには終了条件を必ず入れる** - `retry_count` みたいな上限を設けないと無限ループになる
3. **LangGraph Studioで可視化する** - 複雑になってきたらGUIで確認すると理解が早い

### おすすめリソース

- [LangGraph公式ドキュメント](https://langchain-ai.github.io/langgraph/)
- [LangChain Academy - Intro to LangGraph](https://academy.langchain.com/courses/intro-to-langgraph) - 無料の公式コース
- [GitHub - langchain-ai/langgraph](https://github.com/langchain-ai/langgraph) - サンプルコードが豊富

### LangChain本も参考になる

「LangChainとLangGraphによるRAG・AIエージェント［実践］入門」（技術評論社）は、両方を体系的に学べる良書です。サンプルコードも充実していて、実際に手を動かしながら学べます。

## まとめ

### LangGraphが向いている場面

- ループや条件分岐が必要なワークフロー
- マルチエージェントシステム
- リトライ、エラーハンドリングが必要な処理
- 人間の確認を挟むフロー（Human-in-the-Loop）

### 向いていない場面

- 単純な一方通行の処理（LangChainで十分）
- プロトタイプを素早く作りたい場合（学習コストがある）

### 最後に

LangGraph、最初は「LangChainとの違いがわからない」「なんで両方必要なの？」と混乱しましたが、使ってみると「なるほど、こういう棲み分けか」と腹落ちしました。

**LangChainは部品、LangGraphはその部品を組み立てるワークフローエンジン**。

複雑なAIエージェントを作りたくなったら、ぜひLangGraphを試してみてください。条件分岐やループが書けるようになると、実現できることの幅がグッと広がりますよ。
