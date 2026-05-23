---
title: "AI音声合成ツール5種を実際に叩いて比較した — 高校生がfaceless YouTube用に検証"
emoji: "🎙️"
type: "tech"
topics: ["ai", "音声合成", "voicevox", "elevenlabs", "python"]
published: true
---

faceless YouTubeのナレーション用にAI音声合成ツールを選ぶ必要があり、主要5種類を実際にAPIから叩いて品質・コスト・実装難易度を比較しました。結論から書くと、用途別に2〜3個を使い分けるのが最適解でした。本記事ではPythonでの呼び出しコード、実測ファイルサイズ、ハマったポイントまで具体的に共有します。

## 検証した5ツールと環境

| ツール | 課金形態 | API/CLI | 検証バージョン |
|---|---|---|---|
| VOICEVOX | 無料(OSS) | ローカルHTTP | 0.21.x |
| にじボイス | 月¥500〜 | REST API | 2026/5時点 |
| ElevenLabs | $5〜/月 | REST API | v1 |
| VOICEPEAK | 買切¥15,840 | CLI | 1.2.x |
| edge-tts | 無料 | Python lib | 7.0系 |

実行環境はMacBook Pro 16(M3 Pro / 48GB)。検証テキストは同一で「人生は短い、と賢者は言いました。けれど本当は、私たちが時間を短くしているだけなのです」(60文字、Stoic系ナレーションを想定)。

## VOICEVOXをPythonから叩く最小実装

VOICEVOXはローカルでHTTPサーバが立ち上がるので、Python側からはrequestsで叩くだけです。

```python
import requests, json

VOICEVOX_URL = "http://127.0.0.1:50021"

def synthesize(text: str, speaker_id: int = 13) -> bytes:
    # 1) audio_query で音声合成用パラメータ生成
    q = requests.post(
        f"{VOICEVOX_URL}/audio_query",
        params={"text": text, "speaker": speaker_id},
        timeout=30,
    )
    q.raise_for_status()
    query = q.json()

    # 2) ピッチ・速度を調整 (Stoic系は0.95倍速が刺さる)
    query["speedScale"] = 0.95
    query["pitchScale"] = -0.03

    # 3) synthesis で実際にwavを生成
    s = requests.post(
        f"{VOICEVOX_URL}/synthesis",
        params={"speaker": speaker_id},
        data=json.dumps(query),
        headers={"Content-Type": "application/json"},
        timeout=60,
    )
    s.raise_for_status()
    return s.content

wav = synthesize("人生は短い、と賢者は言いました。", speaker_id=13)
open("out.wav", "wb").write(wav)
```

`speaker_id=13`は「青山龍星(れいせい)」で、低音男性ボイス。Stoic系・解説系のナレーションに最も刺さる声です。`/speakers`を叩けば全話者IDを取得できます。

**実測:** 60文字でwav生成が約0.4秒。ファイルサイズは約480KB(24kHz/16bit)。エンコードがwavなのでmp3に変換する一手間が必要です。

## edge-ttsで完全無料の日本語Shorts量産

意外と知られていませんが、Microsoft Edgeの読み上げエンジンを叩くPythonライブラリ`edge-tts`は完全無料で、日本語の品質もVOICEVOXに匹敵します。

```python
import asyncio
import edge_tts

async def synth(text: str, voice: str, out_path: str):
    rate = "-5%"  # Stoic系はやや遅く
    communicate = edge_tts.Communicate(text, voice, rate=rate)
    await communicate.save(out_path)

asyncio.run(synth(
    "人生は短い、と賢者は言いました。",
    "ja-JP-KeitaNeural",  # 男性低音
    "out.mp3",
))
```

`ja-JP-KeitaNeural`(男性) と `ja-JP-NanamiNeural`(女性) が安定。**出力がそのままmp3で、ファイルサイズはVOICEVOXのwav比で約1/8**。Shortsを量産する場合のディスク圧縮効果が地味にデカいです。

ただし利用規約上、商用利用はグレーゾーンに近いので、収益化が本格化するならVOICEVOXかにじボイスに移行する前提で使うのが安全です。

## ElevenLabsをAPIから叩く

英語の自然さは2026年時点で頭ひとつ抜けています。faceless YouTubeを英語圏に展開するなら必須レベル。

```python
import requests

ELEVEN_API_KEY = "..."  # 環境変数からロード推奨
VOICE_ID = "pNInz6obpgDQGcFmaJgB"  # Adam

def synth_eleven(text: str, out_path: str):
    url = f"https://api.elevenlabs.io/v1/text-to-speech/{VOICE_ID}"
    headers = {
        "xi-api-key": ELEVEN_API_KEY,
        "Content-Type": "application/json",
    }
    payload = {
        "text": text,
        "model_id": "eleven_multilingual_v2",
        "voice_settings": {"stability": 0.55, "similarity_boost": 0.75},
    }
    r = requests.post(url, json=payload, headers=headers, stream=True)
    r.raise_for_status()
    with open(out_path, "wb") as f:
        for chunk in r.iter_content(4096):
            f.write(chunk)
```

`stability`を低くするほど抑揚が激しくなり、高くするほど淡々と読みます。Stoic系は0.55前後、ドキュメンタリー系は0.7前後が体感の正解。

**コスト注意:** Starter $5/月で月3万文字。60文字のショートが500本作れる計算ですが、リテイクすると一気に消えるので、本番前にVOICEVOXで台本確定→ElevenLabsで本収録の流れが必須です。

## 失敗から学んだ3つの落とし穴

### 1. wav→mp3変換を忘れてYouTubeアップが激重に
最初VOICEVOXのwavをそのままffmpegで動画に乗せていて、60秒Shortsが30MB超えていました。映像コーデックを変えるより先に**音声をmp3 128kbpsに落とす**だけで12MBまで縮みます。

```bash
ffmpeg -i out.wav -codec:a libmp3lame -b:a 128k out.mp3
```

### 2. 商用利用の規約を読まずに月¥3000プランを契約
にじボイス契約前にVOICEVOXで月10本生成していたんですが、最初の収益が出るまで無料縛りにすべきでした。月¥500でも年¥6000、これは高校生のお小遣いだと地味に効きます。先にVOICEVOXで30日試して、需要があると確信してから月額に切り替える順番が安全です。

### 3. 同じ声で量産してアルゴリズムに学習されすぎた
VOICEVOXの青山龍星だけで20本連続UPしたら、YouTubeのアルゴリズムに「テンプレ判定」されたのか初動再生が落ちました。**話者IDを3〜5人ローテーションさせる**だけで初動の表示数が戻りました。テンプレ検知のロジックは公開されていないですが、声紋が同一っぽいシリーズは慎重に扱った方がいいです。

## 使い分けの結論

実際にAPIから叩いて比較した結果、こう落ち着きました。

- **テスト・量産パイプライン:** VOICEVOX(完全無料、ローカルで叩ける)
- **本番ナレーション(日本語):** にじボイス or VOICEVOX青山龍星
- **英語/多言語展開:** ElevenLabs
- **オフライン環境・プロ品質:** VOICEPEAK(買切なので長期使うほど得)

僕も最初は「いきなり一番いいやつを契約しよう」と思ってElevenLabsの$22プランに突っ込みかけました。でも実際に動かしてみると、Shortsの量産フェーズではVOICEVOXで十分で、ElevenLabsは「ここぞ」の本番だけで足りる。**ツール選びは「最高品質」より「自分のフェーズに合う」が正解**だと初めて知った時、地味に驚きました。

最新の料金・規約・話者IDは各公式ドキュメントが正なので、プロダクションに乗せる前に必ず公式リファレンスを確認してください。