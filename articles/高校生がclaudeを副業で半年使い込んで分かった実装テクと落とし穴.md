---
title: "高校生がClaudeを副業で半年使い込んで分かった実装テクと落とし穴"
emoji: "🤖"
type: "tech"
topics: ["claude", "ai", "副業", "個人開発", "anthropic"]
published: true
---

# はじめに

高校生で個人開発と副業をやっている者です。Claude（Anthropic製のLLM）を半年ガッツリ使ってきて、Web版だけじゃなくAPI経由でツールを20個以上作ってきました。

この記事では「Claudeとは何か」みたいな概論ではなく、**実際にコードを書いて自動化に組み込むときに刺さるポイント**を、自分が踏んだ罠と合わせて書いていきます。

# Claude を技術観点で見るときの3つの軸

公式の紹介文を見ても「長文に強い」くらいしか分からないので、自分が判断軸にしているのは以下の3つです。

| 軸 | 数値 (Sonnet 4.5 / Opus 4.x 想定) |
|---|---|
| コンテキストウィンドウ | 200K トークン (拡張版で 1M) |
| 出力上限 | 64K トークン (モデルによる) |
| Tool Use / MCP 対応 | あり (関数呼び出し・ファイル操作) |

200Kトークンってどれくらいかというと、**日本語で約14万〜15万字**。新書1冊+αを丸ごと食わせて要約させられます。初めて検証で2万字の議事録をぶち込んだとき、ChatGPTでは途中で文脈が壊れていた要素もきれいに拾ってきて衝撃でした。

# API を実際に叩く最小コード

最初はWeb版で十分でしたが、定期実行や大量処理をしたくなった瞬間からAPIが必要になります。

```python
import anthropic

client = anthropic.Anthropic()  # ANTHROPIC_API_KEY を環境変数で

message = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "1〜100の素数を一行で全部出して"}
    ],
)

print(message.content[0].text)
print(f"input: {message.usage.input_tokens} tok")
print(f"output: {message.usage.output_tokens} tok")
```

`usage` が返ってくるので、毎回トークン数をログに残しておくと費用感が掴めます。自分は最初これをサボって月の請求で目を剥きました(後述)。

# プロンプトキャッシュで料金が 1/10 になる話

API使い始めの頃、長い system プロンプトを毎回送っていて、**1日 500円 → 月15,000円**くらい飛んでました。それを救ったのが **Prompt Caching**。

```python
message = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "あなたは技術記事のリライト専門アシスタントです...",
        },
        {
            "type": "text",
            "text": LONG_STYLE_GUIDE,  # 5000トークンくらいのスタイルガイド
            "cache_control": {"type": "ephemeral"},
        },
    ],
    messages=[{"role": "user", "content": user_input}],
)
```

`cache_control` を付けたブロックは**5分間キャッシュ**され、2回目以降のリクエストは**入力単価が約1/10**になります(キャッシュヒット時は0.1倍、書き込み時は1.25倍)。

実測で、似たプロンプトを連続実行するワークフローでは**月コストが14,800円 → 1,600円**まで下がりました。連投する自動化スクリプトを書く人は最初から入れた方がいいです。

# Tool Use でClaude自身に行動させる

Claudeに「JSONで返して」と頼むより、`tools` 定義で関数呼び出し風にした方が圧倒的に安定します。

```python
tools = [{
    "name": "save_post",
    "description": "投稿文をDBに保存する",
    "input_schema": {
        "type": "object",
        "properties": {
            "text": {"type": "string", "description": "本文 (200字以内)"},
            "tone": {"type": "string", "enum": ["question", "fact", "story"]},
        },
        "required": ["text", "tone"],
    },
}]

resp = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "AIの便利な使い方を1本書いて"}],
)

for block in resp.content:
    if block.type == "tool_use":
        save_to_db(**block.input)  # ここで実関数を呼ぶ
```

Web版でも `JSON Mode 的なプロンプト` を頑張って書いてましたが、Tool Use にしてから**フォーマット崩れによるパース失敗率が約23% → ほぼ0%** になりました。

# 失敗から学んだ罠 3つ

## 1. `max_tokens` 設定ミスで途中で切れる

長文要約タスクで `max_tokens=1024` のまま走らせていて、出力が突然「...」で切れる事故を量産しました。**出力上限は明示的に上げる**(Sonnet系なら64Kまで指定可)、かつ `stop_reason` を必ずチェックする。

```python
if resp.stop_reason == "max_tokens":
    raise RuntimeError("出力が切られた、max_tokens 増やせ")
```

## 2. system プロンプトに動的な値を入れるとキャッシュが壊れる

「今日の日付」「ユーザー名」みたいな**毎回変わる値をsystem側に入れるとキャッシュキーが毎回違うものとして扱われ、キャッシュヒット率0%**。動的な値は `messages` 側に逃がすのが正解です。これに気付くまで1週間料金が下がらず焦りました。

## 3. レート制限はモデルごとに違う

ティア1だと Sonnet 4.5 は分間50リクエスト、Opus はさらに厳しい。並列で叩くスクリプトを書くと一瞬で429が返ってきます。`anthropic` SDKは自動リトライしてくれるけど、`max_retries` を明示しておくのが安心。

```python
client = anthropic.Anthropic(max_retries=4)
```

# 他LLMと役割分担した方が結果がいい

「Claudeだけで完結させる」より、適材適所で混ぜた方が品質も費用もいいです。自分の構成はこんな感じ:

| 用途 | 使ってるモデル | 理由 |
|---|---|---|
| 長文要約・記事生成 | Claude Sonnet 4.5 | 文脈保持と日本語の落ち着き |
| コード生成・リファクタ | Claude (Claude Code) | 差分提案が読みやすい |
| 画像生成 | 他社 | Claudeは画像生成非対応 |
| 速報系の検索 | Web検索付きの別ツール | 即時性が必要なとき |

特にClaude Codeは、ローカルのファイルを直接読み書きしながら開発を手伝ってくれるので、副業で書き散らかしたツール群のメンテナンスが劇的に楽になりました。

# 高校生から見たコスパ感

無料枠だけでも勉強用途なら困りません。**月3,000円のProプラン**は、毎日1時間以上Web版を触る人ならすぐ元が取れます。API課金は使い方次第ですが、自分の場合プロンプトキャッシュ込みで**月1,500〜3,000円**で20本以上のスクリプトを動かせています。

最初に驚いたのは、コード生成タスクで「なぜこう書くか」を毎回理由付きで返してくれるところ。写経で終わらないので、独学スピードがあからさまに上がりました。

# おわりに

Claudeを「便利な対話AI」で止めずに、API・Tool Use・キャッシュまで踏み込むと、副業や個人開発の自動化が一気に現実的になります。料金や機能は四半期単位でアップデートされるので、本番に組み込む前に [公式ドキュメント](https://docs.anthropic.com) でモデルIDと料金は必ず最新を確認してください。