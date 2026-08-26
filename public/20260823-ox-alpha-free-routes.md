---
title: Ox Alpha を無料で試す4つの経路と、正体を推測するための材料の集め方
tags:
  - OxAlpha
  - OpenRouter
  - opencode
  - LLM
  - 生成AI
private: false
updated_at: '2026-08-23T11:46:02+09:00'
id: 707b2da49958c27f5e2e
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-23_ox-alpha_hero.png)

30.9 秒でした。

バグを 1 個仕込んだ Python を渡して「テストが通るように直して」と頼んだら、ファイルを 2 つ読んで、1 文字だけ書き換えて、自分でテストを走らせて、7/7 で戻ってきた。使ったモデルの名前は Ox Alpha です。誰が作ったのかは分かりません。値段は 0 円です。

続きを書きました。正体の確認と、無料期間に手元で使ったトークン量です。

続編: [Ox Alpha の正体は Z.AI の GLM だった。無料期間に手元で使ったトークン量](https://qiita.com/ishizakahiroshi/items/36b652d03016c3f59739)

![記事の要約（インフォグラフィック）](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-23_ox-alpha_infographic.png)

## 先に結論。今どこで無料で触れるのか

2026-08-23 時点で、Ox Alpha に無料で触れる経路は 4 つあります。手元で 2 つは実際に動かしました。

| 経路 | モデル ID | 課金 | 備考 |
|---|---|---|---|
| OpenRouter | `stealth/ox-alpha` | 無料 | API と Playground。データは提供者側が保持（学習には使わない） |
| OpenCode Zen | `x-preview-f-free` | 無料 | ゼロ保持を明記。利用には請求情報の登録が必要 |
| OpenCode Go | `ox-alpha-free` | 無料 | Go は初月 $5・以降 $10/月だが、Ox Alpha 分は Go の枠を消費しない |
| Felo API | `ox-alpha` | 無料（ローンチ期間） | 終了日の記載なし |

OpenRouter のモデルページはこちらです（https://openrouter.ai/stealth/ox-alpha）。OpenCode Zen のモデル一覧は https://opencode.ai/docs/zen/ 、Go の料金は https://opencode.ai/go にあります。Felo は https://openapi.felo.ai/models/stealth/ox-alpha です。

いちばん手数が少ないのは OpenRouter の Playground です。ブラウザだけで終わります。API から叩くならこれだけです。

```bash
curl https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "stealth/ox-alpha",
    "max_tokens": 8000,
    "messages": [{"role": "user", "content": "こんにちは"}]
  }'
```

`max_tokens` を大きめに置いているのには理由があります。後で書きます。

ターミナルのコーディングエージェントから使うなら OpenCode が速いです。インストール済みなら 1 行です。

```bash
opencode run -m opencode-go/ox-alpha-free "この関数のバグを直して"
```

モデルが一覧に出るかは、これで確かめられます。

```bash
opencode models | grep -E 'ox-alpha|x-preview'
```

![無料で使える4経路の比較](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-23_ox-alpha_fig1.png)

4 経路のうち、データの扱いだけは横並びになっていません。OpenCode Zen は「ゼロ保持・学習に使わない」と書いてあり、OpenRouter は「提供者が保持する。ただし学習には使わない」と書いてあります。同じモデルでも入口で条件が違うので、業務のコードを流す前にここだけは見てください。

## そもそも Ox Alpha とは何なのか

2026-08-20 に OpenRouter と OpenCode へ同時に載った、開発元非公開のモデルです。この手のものは「ステルスモデル」と呼ばれます。

公式に出ている仕様はこのあたりです。

- コンテキスト 1,048,576 トークン（約 1M）
- 最大出力 131,072 トークン
- 入力はテキスト・画像・動画、出力はテキスト
- function calling 対応、`response_format` による JSON 出力対応（スキーマ強制はなし）
- 価格は入力も出力も $0

OpenRouter の説明文は「Ox Alpha is a reasoning model designed for coding, sustained agentic work, and production workloads」の 1 行だけです（https://openrouter.ai/stealth/ox-alpha）。作った会社の名前はどこにも書いてありません。OpenRouter 自身も「自分は開発元でも所有者でも提供者でもなく、リクエストを中継しているだけ」と明記しています。

告知は OpenCode 側が先に出しています。「Ox Alpha（ステルスモデル）は次の 1 週間無料」「1 日あたり 100 兆トークンの処理能力がある、あなたができることを試してみましょう」という挑発的なものでした（https://x.com/opencode/status/2090544355824038300）。OpenRouter 側の告知も同じ日です（https://x.com/OpenRouter/status/2090544970923184269）。

翌日には OpenCode Go にも入りました。ここで「次の 6 日間」と期限が具体化しています（https://x.com/opencode/status/2090758645499728234）。8/21 から数えて 6 日なので、8/27 前後には閉じる計算になります。ただしこれは告知文からの逆算で、終了日時そのものは公表されていません。

## 手元で試したら、GLM-5.3 より速かった

読んだだけだと分からないので、同じ課題を 3 通りで走らせました。

課題は区間マージ関数です。`(1,3)` と `(3,5)` のように端点が接する区間をマージしない、という境界バグを 1 個だけ仕込んであります。テストは 7 ケースで、素の状態では 5/7 しか通りません。

```python
def merge_intervals(intervals):
    """区間のリストを受け取り、重なる区間と隣接する区間をマージして返す。
    例: [(1,3),(3,5)] -> [(1,5)]  /  [(1,2),(4,5)] -> [(1,2),(4,5)]
    """
    if not intervals:
        return []
    intervals = sorted(intervals)
    result = [intervals[0]]
    for start, end in intervals[1:]:
        last_start, last_end = result[-1]
        if start < last_end:          # ここが原因
            result[-1] = (last_start, max(last_end, end))
        else:
            result.append((start, end))
    return result
```

指示は「test_intervals.py が全件通るように intervals.py だけを修正してください。テストファイルは変更しないでください」の 1 行です。結果はこうなりました。

| モデル | 所要 | 結果 | 修正内容 |
|---|---|---|---|
| Ox Alpha（OpenCode Go） | 30.9 秒 | 7/7 | `start < last_end` を `<=` に |
| Ox Alpha（OpenCode Zen） | 48.5 秒 | 7/7 | 同上 |
| GLM-5.3（OpenCode Go） | 84.5 秒 | 7/7 | 同上 |

![同じ課題を3通りで走らせた実測](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-23_ox-alpha_fig2.png)

3 つとも同じ 1 文字にたどり着いて、3 つとも自分で `python test_intervals.py` を実行して 7/7 を確認してから報告してきました。テストファイルを書き換えて通す、みたいなズルもしていません。

ただしこれは 1 回ずつの計測です。速度差を実力差と読むには全然足りません。時間帯の混雑でも簡単に前後します。分かるのは「この難度なら 3 つとも普通に解ける」ということと、「無料枠なのに待たされる感じはない」ということくらいです。

面白かったのは自己申告です。GLM-5.3 は報告の末尾に「使用モデル: GLM 5.3 (opencode-go/glm-5.3)」と正確に書いてきました。Ox Alpha は「使用モデル: ox-alpha」でした。正体を直接聞くと、こう返ってきます。

```
モデル名は「ox-alpha」です。開発元は非公開（undisclosed organization）のため、
企業名等はお答えできません。

学習データのカットオフ時期については、確実な情報を持っていないため
「分からない」とお答えします。
```

「知らない」ではなく「非公開なので答えられない」と言っています。隠すよう指示されている側の言い方です。

![名前を伏せた相手に素性を尋ねる場面](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-23_ox-alpha_illustration-1.jpg)

名札のところだけ黒く塗られた相手に素性を尋ねている、みたいな絵になります。答えを持っていないのではなく、答えないと決められている側の応対です。

## 正体は誰なのか。証拠の集め方が面白い

ここからが本題かもしれません。作った会社が黙っているので、界隈が総出で外側から指紋を採っています。

現時点で優勢なのは Zhipu（Z.ai）の GLM 系という説です。決め手っぽいものが 3 つあります。

**1 つ目は、サーバーが吐いたスタックトレース。** `top_p` に数値ではなく文字列を渡して壊れたリクエストを送ると、バリデーションで落ちたときに内部クラス名が漏れます。出てきたのが `com.wd.paas.api.domain.v4.chat.ChatCompletionRequest` でした。この `paas/v4/chat` という並びは Zhipu の公開 API パス `/api/paas/v4/chat/completions` とそのまま一致します。

**2 つ目は、エラーコード。** 不正な role を投げると `{"code":"1214","message":"Incorrect role information"}` が返ります。同じコードが Z.AI がホストしている GLM 群でも出て、別事業者がホストしている GLM では出ませんでした。つまりこれはモデルの指紋ではなく「誰が API を運用しているか」の指紋です。切り分けとして筋がいい。

**3 つ目は、トークナイザー。** 14 種類の文字体系・絵文字・コード・SQL にまたがる 30 個の入力で、トークン数が GLM-5.3 と 30/30 一致しました。差分は毎リクエストに乗る +75 トークンの固定分だけで、これはシステムプロンプトやルーティングのラッパーで説明がつきます。

この検証の詳細は explainx がまとめています（https://www.explainx.ai/blog/ox-alpha-what-we-know-mystery-ai-model-august-2026）。音声入力を拒否する挙動が GLM-5V と同じで MiMo とは違う、といった消去法も併せて載っています。

## Google 説も出ている。こちらは計算資源からの逆算

日本語圏では別の角度からの推測が出ていて、これが個人的にいちばん面白かった。

OpenCode が言っている「1 日 100 兆トークン」を額面どおり受け取ると、それを捌くのに必要な計算量が見積もれます。アクティブパラメータを控えめに 20B と置くと、3ND から 1 日あたり 6e24 FLOPs 以上。毎日 Llama 3 70B を事前学習するのと同じ規模で、そんな資源を遊ばせておける会社は Google くらいだろう、という筋です（https://x.com/kyo_takano/status/2090787696423952442）。

計算そのものは筋が通っています。ただ前提が 1 個弱い。「100 兆トークン」は OpenCode が告知で出した数字で、実測でも SLA でもありません。プロモーション上の上限として書かれた値なので、これを実際の消費量とみなすと逆算が丸ごとずれます。

ステルスモデルの正体当ては、だいたいこの構図になります。運用層の指紋（スタックトレース・エラーコード・トークナイザー）は再現実験ができて、第三者が同じ手順で確かめられる。一方で規模からの推論は、前提に置いた数字が公称値だと足元から崩れる。前者のほうが強い証拠だと思っています。

![GLM説とGoogle説の根拠を並べた図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-23_ox-alpha_fig3.png)

図にすると、2 つの説の性質の違いがはっきりします。GLM 説は運用層に残った痕跡から積み上げていて、誰でも同じ手順で追試できます。Google 説は公称のスループットから必要な計算量を割り出す形なので、出発点の数字が広告用だと全部が動きます。


とはいえ、どちらも当事者は何も言っていません。Zhipu も OpenRouter も OpenCode も、2026-08-23 時点で肯定も否定もしていません。

## ステルスモデルは、だいたい後で正体が分かる

この形式自体は初めてではありません。

2025-04 に OpenRouter へ現れた Quasar Alpha と Optimus Alpha は、後日 OpenRouter 自身が「OpenAI の GPT-4.1 の初期版だった」と公表しています（https://openrouter.ai/announcements/quasar-alpha-and-optimus-alpha-reveal）。無料で大量に投げさせて実戦のフィードバックを集め、正式発表で名前を明かす。プレビュー版の実戦投入としては合理的なやり方です。

なので Ox Alpha も、そのうち誰かの正式モデルとして名前が付く可能性が高い。無料期間が閉じるタイミングと発表が重なるなら、8/27 前後がその日になります。

## 触る前に知っておくと損しない落とし穴

無料だからといって、そのまま本番のワークフローに差し込むと引っかかる箇所がいくつかあります。

**`max_tokens` を小さくすると本文が空で返ることがあります。** 推論モデルなので思考分のトークンが出力枠を食います。日本語の検証記事では 4000 以上が目安とされていました（https://data-engineering.jp/ox-alpha/）。上の curl で 8000 を置いたのはこのためです。

**生成コードは見た目で通ったと判断しないでください。** OpenRouter 経由で 1 ファイル完結の HTML アプリを書かせた検証では、画面は完成しているように見えるのに、`window.confirm()` に渡す文字列の中で改行がエスケープされておらず、JavaScript が丸ごと動いていなかったそうです（https://www.fukuro-ai.tech/ox-alpha-openrouter-coding-test/）。静かに死ぬタイプの壊れ方がいちばん厄介です。

**「無料枠のレート制限」の適用範囲がはっきりしません。** OpenRouter の無料モデルは 20 リクエスト/分、1 日 50 リクエスト（生涯クレジット $10 以上で 1000）という制限が文書化されていますが、これは「ID が `:free` で終わるモデル」に適用されると書かれています（https://openrouter.ai/docs/api-reference/limits）。Ox Alpha の ID は `stealth/ox-alpha` で `:free` が付きません。同じ枠に入るのかどうか、公式には明示されていません。

**ベンチマークの数字は小さすぎるサンプルから来ています。** DeepSWE の 10 タスクで 80% を出して GPT-5.6 や Claude Fable 5 を上回った、という話が出回っていますが、元は 10 件で試行回数もモデルごとに揃っていません（https://x.com/AGTPinsights/status/2090720619083989328）。別の人が opencode-go の 25 タスクで回した表では、Ox Alpha は 100% 通過したものの、所要時間では grok-4.5 や deepseek-v4-flash の後ろで 3 位でした（https://x.com/gosrum/status/2091121214312046720）。「万能で最強」ではなく「無料の割にちゃんと戦える」が実態に近いと思います。

![閉じかけの窓と、試しかけのターミナル](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-23_ox-alpha_illustration-2.jpg)

窓が細くなっていく感じは、触っていると分かります。試したいことはまだ残っているのに、期限のほうが先に来る。

## おわりに

正体が誰であっても、手元でやることは変わりません。無料のうちに自分の課題を投げて、自分のところで速いか正しいかを見る。それだけです。

面白いのは、正体を突き止めた手口のほうでした。スタックトレースを吐かせる、エラーコードを他社ホストと比較する、トークナイザーの分割数を 30 個の入力で照合する。どれも特別な設備は要らなくて、壊れたリクエストを 1 回投げるところから始まっています。ブラックボックスの外側からでも、ここまで削れるものなんだな、と。

自分の実測のほうは 1 回ずつしか回していないので、速度についてはまだ何も言えていません。もう少し投げてみます。窓が閉じるまで、あと数日。

## あわせて読みたい

- [Ox Alpha の正体は Z.AI の GLM だった。無料期間に手元で使ったトークン量](https://qiita.com/ishizakahiroshi/items/36b652d03016c3f59739)（続編。Bloomberg への回答で正体が確定したあと、手元の利用ログを足した）
- [OpenRouter を many-ai-cli に載せるなら、独立プロバイダにはしない](https://qiita.com/ishizakahiroshi/items/4feda29aa885644c982b)
  今回の入口になった OpenRouter を、自作ツール側からどう扱うか考えた記事です。ステルスモデルのように出入りするモデルを抱える中継所を、固定のプロバイダとして実装しない理由もここに書いています。
- [Claude Code / Codex / Cursor / Copilot / OpenCode で同じ Agent Skills を共有する](https://qiita.com/ishizakahiroshi/items/6821655d5af59a32e50c)
  今回モデルを走らせた OpenCode を含む、複数 CLI で同じスキル棚を共有する構成の話です。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-08-23_ox-alpha-free-routes/

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
