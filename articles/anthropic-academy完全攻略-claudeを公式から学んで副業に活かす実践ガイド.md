---
title: "Anthropic Academy完全攻略：Claudeを公式から学んで副業に活かす実践ガイド"
emoji: "🎓"
type: "tech"
topics: ["claude", "ai", "anthropic", "個人開発", "副業"]
published: true
---

「Claudeをもっと深く使いこなしたいけど、独学だとどうしても限界がある」
高校生の僕がClaude APIを触り始めた時、まさにこの壁にぶつかりました。Qiitaやブログ記事は断片的、Udemyの講座は古い情報が混ざっている。そんな中で出会ったのが、Claude開発元のAnthropicが運営する公式学習プラットフォーム **Anthropic Academy** です。

最初に開いた時、率直に「なんでこれが無料なんだ」と驚きました。プロンプトエンジニアリング、Claude API、MCP、Agent SDKまで、開発元が直接書いた教材が全部タダで読めるんです。本記事では実際に主要コースを一通り走った経験をもとに、コード例と数値を交えて活用法をまとめます。

## Anthropic Academyの全体像

Anthropic Academyは `anthropic.com/learn` 配下にある公式ラーニングサイトで、次の2系統に分かれています。

- **Claude利用者向け**：プロンプト設計、Claude.ai運用、Claude Codeの活用
- **開発者向け**：API、Tool Use、MCP、Agent SDK

特徴をまとめると、

| 項目 | 内容 |
| --- | --- |
| 料金 | 完全無料（API実行分は別途従量課金） |
| 言語 | 英語ベース、コードと図解多め |
| 形式 | Jupyter Notebook + Markdown + 動画 |
| 更新頻度 | モデルリリースに追従（Sonnet 4.6/Opus 4.7対応済） |

サードパーティ教材だと、新モデルが出てから情報が反映されるまで1〜2か月かかることが多いです。公式は出た当日に更新されるので、情報鮮度が圧倒的に違います。

## 注目コース5本と実装例

### 1. Prompt Engineering Interactive Tutorial

全9章のNotebook形式チュートリアル。XMLタグでの指示分離が一番役に立ちました。例えば曖昧なプロンプトと、構造化したプロンプトでは出力品質がここまで変わります。

```python
from anthropic import Anthropic

client = Anthropic()

prompt = """
<task>以下の議事録を要約してください</task>
<format>
- 3行以内
- 決定事項を先頭に
- アクションアイテムは箇条書き
</format>
<transcript>
{transcript_text}
</transcript>
"""

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": prompt}]
)
```

タグなしで「議事録を要約して」と頼むと、文体や粒度がバラバラになります。タグで意図を分離してからは、同じプロンプトを100回流しても出力ブレが体感で半分以下に減りました。

### 2. Claude Code in Action

ターミナル型エージェントの実演講座。リポジトリ単位でのリファクタや、テスト自動生成のデモが秀逸です。僕は自分のNext.jsプロジェクトに対して以下を1コマンドで走らせ、Jest テストを23ファイル分自動生成しました。

```bash
claude "src/utils配下の全関数に対してjestのユニットテストを生成して、coverageが80%超えるようにして"
```

手で書くと半日コースの作業が、20分で完了。ただし生成テストの妥当性レビューは必須で、3割は手直しが必要でした。「全部任せる」ではなく「8割叩き台」と割り切ると効きます。

### 3. Building with the Claude API

ストリーミング応答とTool Useを最短で実装する講座。Tool Useは最初コードが複雑に見えますが、本質は「JSON Schemaで関数定義を渡す → Claudeが呼びたい関数と引数を返す → 実行結果を再送」の3ステップです。

```python
tools = [{
    "name": "get_weather",
    "description": "指定都市の現在気温を返す",
    "input_schema": {
        "type": "object",
        "properties": {
            "city": {"type": "string", "description": "都市名"}
        },
        "required": ["city"]
    }
}]

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "東京の天気は？"}]
)
```

### 4. Model Context Protocol Fundamentals

MCPはClaudeと外部サービスを繋ぐ標準プロトコル。GitHub、Notion、Slackを連携させたAIワークフローが組めます。公式講座を読まないと「stdio transport」と「server capabilities」あたりで詰まりがちです。

### 5. Agent SDK Bootcamp

自律エージェントのループ制御・ツール呼び出し・コンテキスト管理の入門。プロダクション運用で詰まる「無限ループ」「コスト爆発」の対策が具体的に書かれていて、僕も最初に作ったエージェントが料金1日3,000円飛ばした経験があるので痛いほど共感しました。

## 副業・収益化への落とし込み

学んだ内容を即お金に変えるルートを3つ整理します。

1. **受託案件の単価アップ**：プロンプト設計力があると、クラウドソーシングで「AIライティング」案件の単価が文字単価0.8円→2円帯に上がります
2. **技術記事の有料販売**：公式情報を日本語化＋自分の検証結果を加えた記事は需要が高い
3. **個人向けSaaS**：Claude API + MCPで特定業界の自動化ツール。月額980〜3,000円帯で5〜30人捕まえれば月数万円の継続収入になります

## 失敗から学んだ3つの教訓

主要コースを2週間で走った後、振り返って「これは早く知りたかった」と思った点。

- **無料クレジット枠を一気に使わない**：演習を1日でまとめて走らせたら、$5枠を90分で溶かしました。章ごとに区切るのが正解
- **Notebookは写経が必須**：読むだけだと2日で内容を忘れます。手を動かすと定着率が体感3倍
- **学習メモを公開する**：章ごとにZennかnoteにまとめると、アウトプット前提で読むのでインプット速度が上がります

## 効率的な進め方

僕がやって効果あった順番は次の通り。

1. Prompt Engineering Tutorial（全コースの土台）
2. Building with the Claude API（手を動かしてAPIに慣れる）
3. Claude Code in Action（自分のプロジェクトで実戦投入）
4. MCP Fundamentals → Agent SDK Bootcamp（応用編）

1日30分×14日で主要5コースは一巡できます。Claude.aiやClaude Codeを横で開いて、読みながら即試すのが最速ルートです。

公式が無料で原典を出してくれている時代に、有料スクールから始める必要はないと個人的には思います。まずは Prompt Engineering Tutorial の第1章だけでもいいので開いてみてください。