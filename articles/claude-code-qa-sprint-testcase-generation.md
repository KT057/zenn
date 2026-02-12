---
title: "支援先のQAチームにClaude Codeを導入してテストケース生成を自動化した話"
emoji: "🧪"
type: "tech"
topics: ["claude", "claudecode", "jira", "qa", "agile"]
published: false
---

## きっかけ：支援先のスプリント開発で見えた課題

技術顧問として入っている会社で、スプリント開発の現場を見ていてずっと気になっていたことがあった。

**QAチームのテストケース作成がボトルネックになっている。**

具体的にはこんな状況だった。

- スプリント後半にQAの作業が溜まり、テストケース作成が間に合わない
- JIRAチケットの受け入れ条件からテストケースへの変換が完全に手作業
- QAメンバーはコードを読めないので、エンジニアに「この変更、何に影響する？」と毎回聞く
- 新しいQAメンバーが入ると、テスト観点を覚えるまでに時間がかかる

OpenObserveでも同じような課題があったらしく、[Claude Codeで自動化したところ特徴分析が45〜60分から5〜10分に短縮](https://openobserve.ai/blog/autonomous-qa-testing-ai-agents-claude-code/)、テストカバレッジも380から700以上に増えたという記事を見つけた。

これを参考に、**Claude CodeのSkills・サブエージェント・JIRA MCPを組み合わせて、QAが `/generate-testcases Sprint-5` と打つだけでテストケースを自動生成する仕組み**を作ってみた。その設計と導入の記録をまとめる。

## 設計した仕組みの全体像：3層構造で「設計」と「利用」を分離する

僕がエンジニアチーム側で仕込んで、QAチームはコマンドを叩くだけ、という構造にした。全体は3つのコンポーネントで構成されている。

### 1. JIRA MCP：チケット情報を取得する基盤

Model Context Protocol（MCP）を使ってJIRAと連携し、スプリントチケット・受け入れ条件・ストーリーポイント等を取得できるようにした。エンジニアが`.mcp.json`を一度設定すれば、QAチームもJIRAデータにアクセスできる。

### 2. Skill：QAの入口＋テスト観点のナレッジ

`.claude/skills/generate-testcases/SKILL.md`に、**QAが呼び出すスラッシュコマンド**と**プロジェクト固有のテスト観点**を一体で定義した。

ポイントは2つ。

- **`user-invocable: true`**: QAが `/generate-testcases Sprint-5` とスラッシュコマンドで呼び出せる
- **`context: fork`**: Skillが隔離されたサブエージェントとして実行される（メインの会話コンテキストを汚さない）

つまり、Skillが「QAの入口」であり「テスト観点のナレッジベース」でもあるという一石二鳥の設計。

### 3. サブエージェント：安全な実行環境

`.claude/agents/qa-executor.md`に、QAタスク用の**安全な実行環境**を定義した。

- **`disallowedTools: [Write, Edit]`**: コードの書き換えを禁止
- **`mcpServers: [atlassian]`**: JIRA MCPへのアクセスを許可
- **`model: sonnet`**: コストと能力のバランス

サブエージェントの役割は「何をするか」ではなく「どう安全に実行するか」。ワークフローやテスト観点はSkill側に書く。QAがうっかりコードを壊す心配がないので、エンジニアチームも安心して使ってもらえた。

### ユーザー（QA）の使い方の流れ

1. QAがClaude Codeを起動
2. **`/generate-testcases MYPROJECT-Sprint-5`** とスラッシュコマンドで実行
3. Skillが `context: fork` で隔離実行される
4. JIRA MCPからチケット取得 → 受け入れ条件読み取り → コード解析 → テストケース生成
5. CSVファイルとして `./testcases/` に出力され、結果がQAに返される

この設計により、**エンジニアが「何をテストすべきか」の知識をSkillに整備し、QAは `/generate-testcases` を叩くだけ**という役割分担が実現できた。

## エンジニアが作るもの①：JIRA MCP設定

まず、Claude CodeがJIRAにアクセスできるようにする。支援先はJIRA Cloudを使っていたので、Atlassian MCPサーバーを設定した。

### .mcp.jsonの設定

プロジェクトルートに`.mcp.json`を作成し、Atlassian MCPサーバーを設定する。

```json
{
  "mcpServers": {
    "atlassian": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-atlassian"],
      "env": {
        "JIRA_URL": "${JIRA_URL}",
        "JIRA_EMAIL": "${JIRA_EMAIL}",
        "JIRA_API_TOKEN": "${JIRA_API_TOKEN}"
      }
    }
  }
}
```

### 環境変数の設定

`.env`ファイルまたはシステム環境変数に以下を設定する。

```bash
JIRA_URL=https://your-domain.atlassian.net
JIRA_EMAIL=your-email@example.com
JIRA_API_TOKEN=your-api-token
```

JIRA API Tokenは、[Atlassianアカウント設定](https://id.atlassian.com/manage-profile/security/api-tokens)から発行できます。

### MCPサーバーの動作確認

Claude Codeを起動し、以下のように確認する。

```bash
claude
```

起動後、MCPツールが利用可能かを確認。

```
利用可能なMCPツールを教えて
```

`mcp__atlassian`ツールが表示されればOK。

これでClaude CodeがJIRAのチケット情報（タイトル、説明、受け入れ条件、ステータス、担当者等）を取得できるようになる。

## エンジニアが作るもの②：Skill（QAの入口＋テスト観点）

次に、QAが呼び出すSkillを作成する。ここが今回の仕組みの肝で、このSkillが**QAの入口**であり**テスト観点のナレッジベース**でもある。

### Skillの配置

`.claude/skills/generate-testcases/SKILL.md`を作成する。

````markdown
---
name: generate-testcases
description: |
  JIRAスプリントチケットからテストケースを自動生成する。
  スプリント名やチケット番号を指定されたら使う。
argument-hint: "[スプリント名 例: MYPROJECT-Sprint-5]"
user-invocable: true
disable-model-invocation: true
context: fork
allowed-tools: Read, Grep, Glob, Bash
---

# テストケース自動生成

スプリント $ARGUMENTS のテストケースを生成します。

**重要: $ARGUMENTS が空（スプリント名が未指定）の場合、処理を開始せず必ずユーザーにスプリント名を確認してください。** 例: 「対象のスプリント名を指定してください（例: MYPROJECT-Sprint-5）」

## 処理フロー

1. JIRAからスプリント内のチケット一覧を取得
2. 各チケットの受け入れ条件（Acceptance Criteria）を抽出
3. Grep/Globで該当機能の関連コードを解析
4. 下記のテスト観点を適用してテストケースを生成
5. Bashで `./testcases/` ディレクトリを作成し、CSVファイルとして出力

## テスト観点（プロジェクト固有）

### 基本テスト観点
- **正常系**: ユーザーが期待する操作を行った際の動作確認
- **異常系**: 不正データ入力、必須項目未入力、文字数超過
- **境界値**: 最小値・最大値、0件・1件・大量データ

### セキュリティ観点
- 未ログイン状態でのアクセス制御
- 権限のないユーザーでのアクセス制御
- XSS・CSRF対策の確認

### パフォーマンス観点
- 画面表示が3秒以内に完了するか
- 複数ユーザーの同時アクセス時の動作

### プロジェクト固有のビジネスルール
- 関連データの連動更新確認
- トランザクション境界・ロールバック
- 通知・アラート条件の確認

## 出力フォーマット（CSV）

テストケースをCSVファイルとして出力してください。

### 出力先

Bashで以下を実行してCSVファイルを生成する。

```bash
mkdir -p ./testcases
```

ファイル名: `./testcases/{スプリント名}.csv`（例: `./testcases/MYPROJECT-Sprint-5.csv`）

### CSVヘッダー

```csv
チケットID,機能名,テストケースNo,テスト名,テスト観点,事前条件,テスト手順,期待結果
```

### CSV出力例

```csv
チケットID,機能名,テストケースNo,テスト名,テスト観点,事前条件,テスト手順,期待結果
MYPROJECT-123,ユーザー登録機能,1,正常系 - 基本登録フロー,正常系,ユーザー登録画面が表示されている,"1. メールアドレスを入力
2. パスワードを入力
3. 「登録」ボタンをクリック","ユーザー登録が成功し確認メールが送信される
ログイン画面にリダイレクトされる"
MYPROJECT-123,ユーザー登録機能,2,異常系 - メールアドレス形式エラー,異常系・バリデーション,ユーザー登録画面が表示されている,"1. 不正なメールアドレスを入力
2. 他フィールドを正しく入力
3. 「登録」ボタンをクリック","エラーメッセージが表示される
ユーザー登録は実行されない"
```

### CSV出力ルール

- 値にカンマ・改行・ダブルクォートが含まれる場合はダブルクォートで囲む
- ダブルクォート内のダブルクォートは `""` でエスケープする
- 文字コードはUTF-8（BOM付き）で出力する（Excelでの文字化け防止）
- Bashでファイル書き出し後、出力先パスをユーザーに通知する

## 注意事項

- 受け入れ条件を必ず網羅すること
- 正常系・異常系・境界値をセットで考慮すること
- セキュリティ観点は常にチェックすること
- CSVの改行はダブルクォート内に含めること（フィールド区切りと混同しない）
````

### 設計ポイントの解説

**`user-invocable: true`** により、QAが `/generate-testcases Sprint-5` とスラッシュコマンドで直接呼び出せる。[Claude Code Skillsの公式ドキュメント](https://code.claude.com/docs/en/skills)によると、`user-invocable` なSkillは `/` メニューに表示され、`argument-hint` で引数のヒントも表示される。QAメンバーに「/ を打って選ぶだけ」と伝えたらすぐ使ってもらえた。

**`disable-model-invocation: true`** により、Claudeが自動判断でこのSkillを呼び出すことを防ぐ。QAが明示的に `/generate-testcases` で実行したときだけ動く。意図しないタイミングでJIRAアクセスやCSV出力が走るのを防ぐための安全策。

**`context: fork`** により、このSkillは隔離されたサブエージェントとして実行される。メインの会話コンテキストを汚さず、独立した環境でJIRAアクセスやコード解析を行う。

**`allowed-tools`** でSkill実行中に使えるツールを制限している。Read, Grep, Glob, Bashのみ許可し、Write/Editは含めないことで安全性を確保した。

**`$ARGUMENTS`** でQAが渡したスプリント名を受け取れる。`/generate-testcases MYPROJECT-Sprint-5` と実行すると、`$ARGUMENTS` が `MYPROJECT-Sprint-5` に展開される。

## エンジニアが作るもの③：サブエージェント（安全な実行環境）

最後に、QAタスク用の**安全な実行環境**をサブエージェントとして定義する。

### サブエージェントの役割

ここが設計上のこだわりポイント。サブエージェントには**「何をするか」は書かない**。ワークフローやテスト観点はSkill側に定義済み。サブエージェントの責務は「どう安全に実行するか」だけ。

### サブエージェントの配置

`.claude/agents/qa-executor.md`を作成する。

```markdown
---
name: qa-executor
description: |
  QAタスク用の安全な実行環境。
  コードの読み取り・解析のみ可能で、書き換えは一切できない。
disallowedTools:
  - Write
  - Edit
mcpServers:
  - atlassian
model: sonnet
maxTurns: 20
---

あなたはQA支援用の実行環境です。
コードの読み取りとJIRAチケットの参照のみ行い、コードの変更は一切行いません。
```

### 設計ポイントの解説

**`disallowedTools: [Write, Edit]`** で、このエージェントがコードを書き換えることを防ぐ。[Claude Codeの公式ドキュメント](https://code.claude.com/docs/en/permissions)によると、deny rulesは最優先で評価されるため、確実に制限が効く。エンジニアチームに説明したとき、この制限があることで「QAに使ってもらってOK」とすぐ合意が取れた。

**`mcpServers: [atlassian]`** で、JIRA MCPサーバーへのアクセスを許可する。`tools` フィールドに `mcp__atlassian` と書くのではなく、`mcpServers` フィールドで指定するのが正しい方法。

**`model: sonnet`** で、コストと能力のバランスを取った。テストケース生成にはOpusほどの能力は不要で、Sonnetで十分な品質が出る。

### Skill側にワークフローを書く理由

最初はサブエージェントのプロンプトにワークフローを直接書き、`skills:` フィールドでテスト観点を注入する設計にしていた。でも、これだとサブエージェントがSkillに依存してしまい、テスト観点を変えるたびにサブエージェント側も確認が必要になる。

結局、**Skill = 「何をするか」、サブエージェント = 「どう安全に実行するか」** と責務を明確に分離する設計に落ち着いた。テスト観点を更新したいときはSkillだけ修正すればいい。QAチームやエンジニアが「テスト観点を追加したい」と言ったときに、Skillファイル1つだけ触ればいいのは運用がラク。

## QAの使い方：実際にやってもらった

ここからはQAチームに渡した手順。実際にやってもらったら、想像以上にスムーズだった。

### ステップ1: Claude Codeを起動

プロジェクトディレクトリで、通常通りClaude Codeを起動する。

```bash
cd /path/to/your/project
claude
```

### ステップ2: スラッシュコマンドで実行

`/` を入力すると、利用可能なSkillの一覧が表示される。その中から `generate-testcases` を選ぶか、直接入力する。

```
/generate-testcases MYPROJECT-Sprint-5
```

`argument-hint` で `[スプリント名 例: MYPROJECT-Sprint-5]` と表示されるので、QAはどんな引数を渡せばいいか迷いません。

### ステップ3: Skillが隔離実行される

Skillの `context: fork` 設定により、隔離されたサブエージェント環境で以下の処理が自動実行される。

1. JIRA MCPからSprint-5のチケット一覧を取得
2. 各チケットの受け入れ条件を読み取り
3. Grep/Globで該当機能のコードを解析
4. Skill内に定義されたテスト観点を適用
5. テストケースを生成し、CSVファイルとして `./testcases/` に出力

### ステップ4: 結果の確認

`./testcases/MYPROJECT-Sprint-5.csv` にCSVファイルが生成される。実際に出てきた結果はこんな感じだった。

```csv
チケットID,機能名,テストケースNo,テスト名,テスト観点,事前条件,テスト手順,期待結果
MYPROJECT-123,ユーザー登録機能,1,正常系 - 基本登録フロー,正常系,ユーザー登録画面が表示されている,"1. メールアドレスフィールドに「test@example.com」を入力
2. パスワードフィールドに「SecurePass123!」を入力
3. パスワード確認フィールドに「SecurePass123!」を入力
4. 「登録」ボタンをクリック","ユーザー登録が成功し確認メールが送信される
ログイン画面にリダイレクトされる"
MYPROJECT-123,ユーザー登録機能,2,異常系 - メールアドレス形式エラー,異常系・バリデーション,ユーザー登録画面が表示されている,"1. メールアドレスフィールドに「invalid-email」を入力
2. パスワード等の他フィールドを正しく入力
3. 「登録」ボタンをクリック","エラーメッセージ「有効なメールアドレスを入力してください」が表示される
ユーザー登録は実行されない"
MYPROJECT-123,ユーザー登録機能,3,境界値 - パスワード最小文字数,境界値,パスワード最小文字数が8文字と定義されている,"1. パスワードフィールドに7文字のパスワードを入力
2. パスワードフィールドに8文字のパスワードを入力
3. それぞれで「登録」ボタンをクリック","7文字の場合: エラーメッセージが表示される
8文字の場合: 正常に登録される"
MYPROJECT-123,ユーザー登録機能,4,セキュリティ - パスワードの暗号化,セキュリティ・データ保護,ユーザー登録が完了している,1. データベースを確認,"パスワードがハッシュ化されて保存されている
平文のパスワードは保存されていない"
```

CSVなのでExcelやGoogleスプレッドシートで開ける。テスト管理ツール（Zephyr等）にもそのままインポート可能。

### QAにやってもらうこと

生成されたCSVファイルを確認し、必要に応じて調整してもらう。

- CSVをExcel/スプレッドシートで開いてレビュー
- プロジェクト固有の観点が不足していれば行を追加
- 受け入れ条件に明記されていない暗黙の要件があれば補足
- テスト管理ツール（Zephyr等）にインポート

**ゼロからテストケースを作るのではなく、AIが生成した8割のベースを調整する作業に変わる。** これが一番大きな変化だった。QAメンバーからも「CSVで出てくるからそのままインポートできて助かる」という声が出た。[Safieの事例](https://engineers.safie.link/entry/claude-code-test-automation-workflow-subagents)でも同様に「面倒なテストコードはAIに任せる」ワークフローが報告されている。

## スプリントへの組み込み：実際に回してみたサイクル

支援先のスプリント開発に組み込んでみて、以下のような流れに落ち着いた。

### Sprint Planning（スプリント計画）

- ストーリー作成時に**受け入れ条件を充実させる**
- 受け入れ条件が詳細であるほど、テストケース生成の精度が上がる
- 例: 「ユーザーが登録できる」 → 「ユーザーがメールアドレス・パスワード（8文字以上）を入力し、登録ボタンをクリックすると、確認メールが送信され、ログイン画面にリダイレクトされる」

### Development（開発）

- エンジニアがコードを実装
- 必要に応じてSkillのテスト観点を更新（新しいビジネスルール追加時等）

### QA Test Design（テスト設計）

- QAがClaude Codeを起動し、`/generate-testcases MYPROJECT-Sprint-X` を実行
- `./testcases/MYPROJECT-Sprint-X.csv` が生成される
- CSVをExcel/スプレッドシートで開いてレビュー・調整
- テスト管理ツール（Zephyr等）にCSVインポート

### Testing（テスト実行）

- テストケースに基づいてテスト実行
- 必要に応じてテストケースを追加・修正

### Sprint Review（スプリントレビュー）

- テストカバレッジを可視化
- 抜け漏れがあった観点をSkillに追加（継続的改善）

## チームへの効果：導入してみて変わったこと

### テスト設計速度

一番わかりやすく変わったのはここ。テストケースの「叩き台」を作る時間が大幅に減った。OpenObserveの事例では特徴分析が**45〜60分から5〜10分に短縮**されたとのことだが、支援先でも体感としてかなり近い効果が出た。

### カバレッジの向上

[OpenObserveの事例](https://openobserve.ai/blog/autonomous-qa-testing-ai-agents-claude-code/)では、テスト数が**380から700以上に増加**している。支援先でも、AIが「人間が見落としがちな境界値テスト」を自動的に提案してくれるので、カバレッジの抜けが減った。

### QAの立ち上がり速度

途中からジョインしたQAメンバーが、Skillのテスト観点を読んで `/generate-testcases` を叩くだけで、すぐにテストケース作成に参加できた。従来の「プロジェクトのテスト観点を先輩から教わる」時間が大幅に短縮された。

### エンジニアとQAの共通言語

Skillに定義されたテスト観点が「共通言語」になった。「このケース、セキュリティ観点でテストした？」といった会話が自然に生まれるようになり、エンジニアとQAの距離が縮まった実感がある。

## 注意点：実際にハマったところ

導入は順調だったわけではない。正直ハマったポイントをまとめておく。

### JIRA API Tokenの権限不足

最初にハマったのがこれ。JIRA API Tokenを発行したものの、「プロジェクトの閲覧」権限が足りなくてチケット情報を取得できなかった。管理者に依頼して「課題の閲覧」権限も追加してもらう必要がある。地味だけど最初に確認しておくべき。

### 受け入れ条件が曖昧だとテストケースも曖昧

これが一番インパクトが大きかった。受け入れ条件が「ユーザーが登録できる」くらいしか書いていないチケットだと、テストケースも「ユーザー登録のテスト」くらいしか出てこない。結果、Sprint Planningで**受け入れ条件を詳細に書く文化**がチームに根付いた。これは想定外の副次効果だった。

### Skillのメンテナンスが必要

2スプリント回したあたりで、「この観点が抜けてるな」というケースが出てきた。Sprint Reviewで「今回抜けた観点」をSkillに追加するルーティンを入れたら安定した。Skillのメンテナンスをサボるとテストケースの質が下がるので注意。

### allowed-toolsとdisallowedToolsの設定ミス

初回のセットアップで、Skillの `allowed-tools` にうっかりWriteを入れてしまい、テスト中にClaude Codeがファイルを作ろうとした。[公式ドキュメント](https://code.claude.com/docs/en/permissions)にあるように、deny rulesは最優先で評価されるため、**サブエージェント側のdisallowedToolsは最後の砦**として必ず設定しておくべき。

### Skillのdescriptionは具体的に書く

Skillの `description` は、Claudeが「このSkillをいつ使うべきか」を判断する材料になる。最初は説明が曖昧で、QAが自然言語で「テストケースを作って」と指示しても正しくSkillが選ばれないことがあった。descriptionを具体的に書き直したら解消した。

## まとめ：導入してみてどうだったか

### 向いているチーム

支援先での経験から、以下のようなチームには特に効果が出ると感じた。

- アジャイル・スプリント開発を採用している
- JIRAで課題管理をしている
- QAチームが独立していて、エンジニアとの情報格差がある
- テストケース作成がボトルネックになっている

### 向いていないケース

- JIRA以外の課題管理ツールを使っている（Atlassian MCP非対応）
- プロジェクト固有のテスト観点が少なく、汎用的なテストで十分
- QAチームがおらず、エンジニアが自分でテストを書いている

### 導入するなら

まず、**1つのフィーチャーだけで試す**のがおすすめ。支援先でも最初は1チケットだけで試してもらい、出てきたテストケースの品質を確認してからスプリント全体に広げた。いきなり全チケットをカバーしようとすると、Skillのテスト観点が不十分で微妙な結果になる。

エンジニアが「お膳立て」を整備し、QAが `/generate-testcases` を叩くだけの仕組み。Claude CodeのSkills・サブエージェント・JIRA MCPを組み合わせることで、非エンジニアのQAメンバーでもAIの力を使えるようになった。「Claude Code = エンジニア専用ツール」という思い込みを壊せたのが、個人的には一番の成果だったと思う。

本記事で紹介した設定ファイル一式はGitHubで公開している。

https://github.com/KT057/qa-sprint-testcase-generator

## 参考リンク

- [Claude Code 公式ドキュメント - Skills](https://code.claude.com/docs/en/skills)
- [Claude Code 公式ドキュメント - サブエージェント](https://code.claude.com/docs/en/sub-agents)
- [Claude Code 公式ドキュメント - Permissions](https://code.claude.com/docs/en/permissions)
- [OpenObserve - How AI Agents Automated Our QA](https://openobserve.ai/blog/autonomous-qa-testing-ai-agents-claude-code/)
- [Safie - Claude Code テスト半自動化ワークフロー](https://engineers.safie.link/entry/claude-code-test-automation-workflow-subagents)
- [Testomat.io - Jira MCP Server Integration](https://testomat.io/blog/building-jira-mcp-server-integration-with-test-management/)
