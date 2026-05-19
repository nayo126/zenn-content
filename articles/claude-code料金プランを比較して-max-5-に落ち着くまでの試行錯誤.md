---
title: "Claude Code料金プランを比較して、Max 5×に落ち着くまでの試行錯誤"
emoji: "💰"
type: "idea"
topics: ["claude", "ai", "claudecode", "副業", "個人開発"]
published: true
---

「Claude Codeの料金プラン、結局どれが最適なんだ」と1時間悩んで時間を溶かした高校生です。Pro($20)からMax 20×($200)まで触り倒し、API従量課金にも手を出した結果、自分の使い方に合うラインがやっと見えてきたのでログを残します。

「副業でAIに月いくら払うのが妥当か」を真面目に数字で比較したい人向けの記事です。

## プラン構造を最短で把握する

Claude Codeの課金経路は大きく2系統あります。

| 経路 | 料金 | モデル上限 | 主な制約 |
|---|---|---|---|
| Pro | $20/月 | Sonnet 4.6中心 / Opus 4.7少量 | 5時間枠が狭い |
| Max 5× | $100/月 | Sonnet + Opus 4.7 | Proの約5倍枠 |
| Max 20× | $200/月 | Opus 4.7メイン可 | Proの約20倍枠 |
| API | 従量 | 全モデル選択可 | 入力Sonnet $3 / Opus $15 / 100万tok |

肝は「5時間ごとに利用枠がリセットされる」という仕組みで、これを知らないとProでも体感は意外と窮屈です。

## 自分の使用量を計測しないと話が始まらない

「Proで足りるか」を感覚で判断するとほぼ失敗します。まず実利用トークン数を測る習慣を作るのが先。Claude Code CLIにはセッション統計が出るので、それをログに残すスクリプトを噛ませました。

```bash
#!/usr/bin/env bash
# ~/bin/cc-usage-log.sh
LOG="$HOME/.claude/usage.csv"
[ -f "$LOG" ] || echo "date,model,input_tok,output_tok,cost_usd" > "$LOG"

claude --print --output-format json "$@" \
  | tee /tmp/cc-out.json \
  | jq -r '[
      now | strftime("%Y-%m-%d %H:%M:%S"),
      .model,
      .usage.input_tokens,
      .usage.output_tokens,
      .cost_usd
    ] | @csv' >> "$LOG"
```

これで毎回のセッションが行追加されていきます。1週間分を集計するとプラン判断は一瞬で終わりました。

```bash
awk -F, 'NR>1 {c+=$5; i+=$3; o+=$4} END {
  printf "Cost: $%.2f / Input: %d / Output: %d\n", c, i, o
}' ~/.claude/usage.csv
```

自分の場合、副業の月で換算すると入力1,400万トークン / 出力180万トークンに到達していて、API換算だと$70/月程度。「APIで十分か?」と思ったものの、Opus 4.7に切り替えた日だけで$25跳ねていたので、不安定さを考えてMax 5×に着地しました。

## モデルの自動振り分けでコストを倍持たせる

Pro契約で一番効いたのが「軽い質問はSonnet、設計判断だけOpus」というルール化です。CLIの`--model`フラグで切り替えできるので、ラッパーで自動分岐させています。

```bash
#!/usr/bin/env bash
# ~/bin/cc
prompt="$*"
heavy_keywords="(refactor|architect|debug|設計|なぜ|原因)"

if [[ "$prompt" =~ $heavy_keywords ]]; then
  model="claude-opus-4-7"
else
  model="claude-sonnet-4-6"
fi

claude --model "$model" "$prompt"
```

最初に知った時驚いたのが、Opus 4.7とSonnet 4.6の単価差が**5倍**もあること(入力 $15 vs $3 / 100万tok)。全部Opusで投げてた最初の3日間、Proの枠を1日で使い切ったのは今でも笑い話です。

## プロンプトキャッシュで実費を3割削った話

長期プロジェクトで同じコードベースを読ませる場合、プロンプトキャッシュが効きます。API直叩きする場合の例。

```python
import anthropic

client = anthropic.Anthropic()
system_doc = open("project_context.md").read()  # 5万トークン想定

resp = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": system_doc,
            "cache_control": {"type": "ephemeral"},
        }
    ],
    messages=[{"role": "user", "content": "このリポジトリの依存関係を要約して"}],
)
print(resp.usage)
```

`cache_control`を付けるだけで、同一systemを5分以内に再利用したリクエストは**入力単価が1/10**になります。試したプロジェクトで月の請求が$48 → $33に下がりました。サブスクのMaxでも、5時間枠の消費は減るのでお得です。

## 失敗から学んだ3つの設定ミス

実際にやらかしたパターンを共有します。

1. **夜中に8時間ぶっ続けでOpusを使う** → 5時間枠がリセット待ちで翌朝まで作業停止。今は朝・昼・夕方の3分散ルール。
2. **`-c`で過去セッションを延々と継続** → コンテキストが膨れて1リクエスト10万トークン超え。30分以上経ったら新セッションを切る。
3. **APIキーをそのまま `.env` に置く** → ある日リポジトリに混入しかけた。1Passwordの`op run`経由に変更済み。

```bash
# 安全な呼び出し方
op run --env-file="./.env.tpl" -- python script.py
```

## 結論:利用量を測ってから払う

副業で毎日触るなら**Max 5×($100)で枠を気にせず動かせる安心感**が一番大きいです。ただ、最初からMaxに突っ込むのは反対で、まずProで2週間使ってログを取り、自分が1日何トークン使うか可視化してから判断するのが失敗しない順番でした。

- 試したいだけ → Pro ($20)
- 副業で毎日数時間使う → Max 5× ($100)
- フリーランス本業 → Max 20× ($200)
- アプリ組み込み → API従量

「料金プランどれ?」より先に「自分のトークン消費はどれくらい?」を1週間測ってみてください。そこから先は迷う要素がほぼなくなります。