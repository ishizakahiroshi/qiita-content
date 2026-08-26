---
title: Ox Alpha の正体は Z.AI の GLM だった。無料期間に手元で使ったトークン量
tags:
  - OxAlpha
  - GLM
  - OpenRouter
  - LLM
  - 生成AI
private: false
updated_at: '2026-08-26T21:10:16+09:00'
id: 36b652d03016c3f59739
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![ヒーロー（記事トップ）](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-26_ox-alpha-glm_hero.png)

3 日前に書いた記事の結論は、半分当たって半分保留だった。

当たっていたのは「運用層の指紋は GLM 系」。保留にしていたのは「当事者は肯定も否定もしていない」。2026-08-26、Z.AI（智谱）が Bloomberg に答えて、保留が消えた。

この話には前があります。無料で触れる 4 経路と、スタックトレースやトークナイザーから正体を絞り込む手口を書いたものです。

前回の記事: [Ox Alpha を無料で試す4つの経路と、正体を推測するための材料の集め方](https://qiita.com/ishizakahiroshi/items/707b2da49958c27f5e2e)

## 先に結論。分かったことと、まだ分からないこと

2026-08-26 時点の整理です。

- 正体: Z.AI（Zhipu）の GLM シリーズ新世代。Bloomberg への回答で本人確認が入った（https://www.bloomberg.com/news/articles/2026-08-26/china-s-z-ai-made-ox-alpha-stealth-model-that-rivals-deepseek）
- 型番: Bloomberg の本文に「GLM-5.3 Flash」とは書いていない。コミュニティ側の読み
- 規模: OpenRouter の週間ランキング 1 位、23.2 兆トークン（集計終端 2026-08-25）（https://openrouter.ai/rankings）。OpenCode は 6 日で 42 兆トークンと発表（https://x.com/opencode/status/2092551628935065662）
- 手元: OpenCode Go と Zen のアシスタント応答を合算すると、新規入力・出力・推論で約 1,710 万トークン。キャッシュ読みは約 2.3 億。課金は 0 円
- 無料: 執筆時点でも OpenCode のモデル一覧に `Ox Alpha Free (Unlimited)` と出ている。8/27 前後で閉じる、は前作の逆算で、締切そのものはまだ出ていない
- 重み: 「今夜公開する」と Bloomberg に答えている段階。執筆時点では手元にファイルは無い

OpenRouter のモデルページは、確認翌日でもまだ「第三者で匿名」のままです（https://openrouter.ai/stealth/ox-alpha）。カードの更新は、名乗った速度より遅い。

## 名乗ったのは、製品ページからではなかった

Luz Ding 記者の Bloomberg 記事は 2026-08-26 9:00 UTC 付です。書き出しは短い。

オンラインの利用ランキングを無償で駆け上がったモデルは、中国の Z.AI Co.（Zhipu）が作った。同社は水曜日、Ox Alpha が GLM シリーズの新世代だという推測を認め、今夜ウェイトを出すと Bloomberg の問い合わせに答えた。

一次情報はこれです。Z.AI のブログ投稿を先に見つけたわけではない。問い合わせに答えた、という形で出た。

heise も同日、「Z.ai が Bloomberg に確認した」と書いている（https://www.heise.de/en/background/Ox-Alpha-Anonymous-AI-model-between-hype-and-data-protection-11426444.html）。RuntimeWire は Bloomberg を一次情報として、張鵬氏が共同創業した北京のラボだと補足している（https://runtimewire.com/article/z-ai-confirms-ox-alpha-glm-model-weight-release）。

「GLM-5.3 Flash だった」は、X ではすぐ定着した。Lumina が「Zai have confirmed... Looks like the GLM 5.3 Flash theory aged pretty well」と書き（https://x.com/LuminaBench/status/2092563960100745247）、Adam Holter が「stealth model releases are an incredible form of marketing」とまとめている（https://x.com/AdamHoltererer/status/2092579670407397667）。ただし Bloomberg が印刷した語は「a new iteration of its GLM series」まで。Flash という枝番は、指紋側の読みが残っている。

OpenCode は確認の数時間前に、「Reveal in a few hours」とだけ書いて 42 兆のグラフを出していた。中身は grafo で、社名はまだ無い。

## 指紋は当たっていた。100 兆からの逆算は、前提が弱いままだった

前作で優勢だとした材料は、確認後も形を変えて残っている。

1 つ目はトークナイザー。implicator.ai は、公開済み GLM-5 語彙との照合が 95/95、平均絶対誤差 0.00 だったと書いている（https://www.implicator.ai/ox-alpha-zhipu-glm-tokenizer-match/）。explainx は前作でも引いた、スタックトレースとエラーコードのまとめだ（https://www.explainx.ai/blog/ox-alpha-what-we-know-mystery-ai-model-august-2026）。

2 つ目は運用層。不正な role を投げると `{"code":"1214","message":"Incorrect role information"}` が返る。ctgt.ai は、温度の上限がちょうど 1.0 であることと合わせて、Google / OpenAI / xAI を温度 2.0 側へ落とす、と書いている（https://www.ctgt.ai/research/behaviorally-fingerprinting-ox-alphas-provenance）。unclecode の Modelprint は GLM-5.3 に 9 項中 6 項、トークナイザー 4/4 と出している（https://github.com/unclecode/modelprint）。

3 つ目は、過去のステルスの型。日本語の整理では、2026-02 の Pony Alpha が GLM-5 だった、と書かれている（https://joho-todai.com/ox-alpha-tokenizer-analysis-identity/）。heise も Pony Alpha を同じ系譜に置いている。Ox Alpha は「初めての匿名公開」ではなく、「規模が段違いの匿名公開」だった。

外れたほうも、はっきりしている。前作で面白かった Google 説は、OpenCode の「1 日 100 兆トークン」を実消費と置いて FLOPs を逆算するものだった（https://x.com/kyo_takano/status/2090787696423952442）。計算の式は筋が通っていた。弱いのは出発点で、100 兆は告知上の処理能力であって、測ったトラフィックではなかった。

確認後に残る数字は、告知より小さい。それでも十分に大きい。

![告知の100兆と実測の42兆、手元の1710万](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-26_ox-alpha-glm_fig1.png)

上の図は、同じ「トークン」でも層が 3 つある、という話です。いちばん上は宣伝の処理能力。真ん中はプラットフォームが発表した実測。いちばん下は、1 台の OpenCode が残したログです。桁が違うものを、同じ土俵の「すごさ」として足してはいけない。

## 6 日で 42 兆。告知の 100 兆とは、桁が違う

OpenCode の投稿は 1 行目が数字だけだ。「42T tokens of Ox Alpha in 6 days / Most used model after DeepSeek Flash's 56 day run」（https://x.com/opencode/status/2092551628935065662）。DeepSeek Flash が 56 日かけて作った席を、Ox Alpha は 6 日で次点まで持っていった、という意味になる。

OpenRouter 側は、トークン量の週次ランキングで測っている。2026-08-25 終端の表では Ox Alpha が 23.2 兆で 1 位、2 位の DeepSeek V4 Flash 0731 が 11.6 兆。倍以上開いている（https://openrouter.ai/rankings）。ページ自身が「品質ランキングではない。OpenRouter を通ったトークン量である」と注記している。品質の話と採用の話を混ぜないための注記で、今回は採用の話だけが突出している。

heise は日次まで落としている。8/21 に約 2,700 万リクエスト、4 日後には約 7,900 万。日次トークンは約 2 兆から約 5.8 兆。8/25 までの累計リクエストは約 2.95 億（https://www.heise.de/en/background/Ox-Alpha-Anonymous-AI-model-between-hype-and-data-protection-11426444.html）。llmrumors は OpenRouter の日次フィードを 8/21 の 1.9909 兆から 8/24 の 5.9296 兆まで拾っている（https://www.llmrumors.com/news/ox-alpha-mystery-model-openrouter-usage）。

日本語でも、確認前から規模の話は出ていた。ITmedia の整理（Yahoo 転載、8/25）は、OpenRouter 累計 17.5 兆、OpenCode 31 兆・ユーザー 38 万人、Cline では公開 48 時間で推論量 8% と書いている（https://news.yahoo.co.jp/articles/df7922372cd91ed7b8de56ce25957af15d0ca56f）。GIGAZINE は仕様と匿名ラボの処理能力の話を、OpenRouter のカードから起こしている（https://gigazine.net/news/20260824-ox-alpha/）。日付が違うと同じ「兆」でも値が動く。引用するときは、いつの数字かを残したほうがいい。

性能のほうは、最初に拡散した数字が小さすぎた。前作でも書いた DeepSWE 10 タスク 80% は、Decrypt が「113 タスクの別ランではおよそ 63%」と相対化している（https://decrypt.co/376396/mysterious-ai-model-ox-alpha）。heise は 113 タスクで 66 完走、58.4% と書いている。公式リーダボードの Opus 5（74%）や GPT-5.6 Sol（73%）の並びとは条件が違う、とも注記がある。無料の割に戦える、が今もいちばん近い読みだと思う。

![名札の外れたマスクが机に置いてある](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-26_ox-alpha-glm_illustration-unmask.png)

名札を外したあとも、机の上の道具は同じです。触り方は変わらない。変わったのは、答え合わせができるようになったことだけです。

## 正体当てゲームは、広告だった

ここが、確認そのものより面白いかもしれない。

ステルスモデルの型は、OpenRouter 自身が 2025-04 の Quasar Alpha / Optimus Alpha で「あとから GPT-4.1 の初期版だった」と明かしている（https://openrouter.ai/announcements/quasar-alpha-and-optimus-alpha-reveal）。無料で大量に投げさせて実戦のフィードバックを集め、正式発表で名前を出す。合理的ではある。

Ox Alpha で新しかったのは、規模と、当てゲームの熱量だった。TechCrunch は Stripe CEO の Patrick Collison が “very impressive” と評した、と書いている（https://techcrunch.com/2026/08/23/whos-behind-the-new-stealth-model-ox-alpha/）。Stripe は OpenRouter の買収で動いている当事者でもある。中継側の経営者が「すごい」と言うと、モデルの話とプラットフォームの話が同時に増幅する。

Business Insider Japan は、Collison の発言と OpenCode の 100 兆トークンを、Visa の月次トークン量の約 100 倍、という比喩で紹介している（https://www.businessinsider.jp/article/2608-ox-alpha-ai-model-mystery/）。INSIDE（台湾）は指紋の表と、経路ごとのデータ保持の食い違いを並べている（https://www.inside.com.tw/article/42174-ox-alpha-stealth-model-openrouter-glm-5-3）。日本語と中国語と英語で、同じ 1 週間を別々の人が別々の入口から書いていた。

heise の整理が素直だと思う。無償提供は計算資源のコストがかかる。その代わり、開発者が本番に近いコーディングを流してくれる。匿名であること自体が、推測記事と X のスレッドを増やす。マーケティング計算がある、と llmrumors も書いている。

OpenRouter の Apps 欄には、確認前から面白い行があった。`zcode.z.ai` の ZCode が Ox Alpha へ 830B トークンを流している（https://openrouter.ai/stealth/ox-alpha）。上には Hermes Agent が 5.04 兆、Claude Code が 2.58 兆と並ぶ。ZCode が自社製品かどうかは、URL のホスト名以上のことはここでは断言しない。ただ、匿名カードの下に自社ドメインらしいアプリが載っている、という見え方は、ステルスの「完全な匿名」とは少し温度が違う。

名前を出さずに 1 週間、開発者のタイムラインを占有する。確認の日にウェイト公開を予告する。広告費をトークンで払った、と言っても言い過ぎではない。しかも払った先は、自社 API の将来客になりうる人たちだ。

## よくまあ、あの計算資源を用意できた

前作の Google 説が使った式は、アクティブパラメータ 20B、1 日 100 兆トークン、3ND で 6e24 FLOPs、というものだった。毎日 Llama 3 70B を事前学習するのと同じ、という結論になる。前提の 100 兆が実測ではなかったので、結論も一緒に動く。

実測に置き換える。OpenCode の 42 兆 / 6 日は、単純平均で 1 日あたり約 7 兆。告知の 1/14 くらい。OpenRouter の 23.2 兆は「この週に OpenRouter を通った分」で、OpenCode 直叩きや Cline、Hermes は別経路だ。全部を足すと、無料プレビュー 1 本で、普通の有料モデルの週次を軽く超える。

それでも「Google くらいしか遊ばせておけない」規模か、は別問題になる。コミュニティが Flash と読んだ理由のひとつは、そこだ。公開済みの GLM-5.3 は OpenRouter で入力 $1.40 / 出力 $4.40（https://openrouter.ai/z-ai/glm-5.3）。テキスト専用で、Ox Alpha のように画像・動画を表の入力に書いていない。同じ系列の、より安い推論器を無償で配っている、という読みなら、6 日 42 兆は宣伝費として成立しうる。Adam Holter が「効率のいいモデルだから兆単位を補助できる」と書いたのは、この筋である。

確定ではない。ウェイトが出て、ライセンスとパラメータ数が分かってから、式はもう一度置ける。今夜出すと言っているものが、本当に今夜出るかも、執筆時点では分からない。

![ラックの灯が奥まで続いている](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-26_ox-alpha-glm_illustration-compute.png)

灯の数を数えても、1 日 100 兆には届かない。届かなくても、6 日で 42 兆を無償で流せる箱を、誰かが用意していた、という事実は残る。

もう 1 つ、データの扱いだけは前作から変わっていない。OpenRouter のカードは「提供者が保持する。学習には使わない」と書き、Stealth Model Terms 側は評価や改善への利用にも触れる、と heise が指摘している。業務のコードを流す前に、入口の文面を見る、は継続でいい。

## 無料期間に、手元ではこのくらい使った

自分のところの数字が欲しくて、OpenCode が残している利用ログを集計した。会話の本文は出さない。モデル名と、アシスタント応答に付いている tokens だけを足した。

対象は 2 経路。Go の `ox-alpha-free` と、Zen の `x-preview-f-free`。前作で「同じモデルの別入口」と書いたやつだ。期間は 2026-08-23 から 2026-08-25。

| 経路 | アシスタント応答 | 入力 | 出力 | 推論 | 新規の合計 | キャッシュ読 |
|---|---:|---:|---:|---:|---:|---:|
| OpenCode Go（`ox-alpha-free`） | 2,266 | 1,163 万 | 116 万 | 43 万 | 1,323 万 | 1 億 9,292 万 |
| OpenCode Zen（`x-preview-f-free`） | 450 | 341 万 | 35 万 | 11 万 | 387 万 | 3,600 万 |
| 合計 | 2,716 | 1,504 万 | 151 万 | 54 万 | 1,710 万 | 2 億 2,892 万 |

新規の合計は、入力 + 出力 + 推論。キャッシュ読は、同じリクエストで再利用された接頭辞で、新規に生成した量ではない。足すと「処理した」には見えるが、課金や実計算のイメージとしては分けたほうが正直だ。Go も Zen も cost は 0.00。ログ上も無料のまま終わっている。

OpenCode 全体の 42 兆（https://x.com/opencode/status/2092551628935065662）の横に置くと、手元の新規 1,710 万は 0.00004% くらいになる。自分が「結構投げた」と思っていた量が、プラットフォームのノイズにも届かない。それが分かっただけでも、ログを開いた甲斐があった。

セッション表のほうでは、Go が 70 本、Zen が 19 本だった。最後に選んでいたモデルで数えているので、途中で切り替えた分は漏れる。応答件数の 2,716 のほうが、投げた量には近い。

![手元の新規1710万とキャッシュ2.3億](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-26_ox-alpha-glm_fig2.png)

棒の長さを OpenCode の 42 兆に合わせると、手元の新規は線にならない。だからこの図では、手元の内訳だけを並べています。キャッシュが新規の 10 倍以上あるのは、エージェントが同じリポジトリを何度も読み直すからだと思います。

モデル選択の画面は、確認当日もこう出ていた。

![OpenCode のモデル一覧。Unlimited と Free](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-26_ox-alpha-glm_screenshot-models.png)

Go と Zen の両方に `Ox Alpha Free (Unlimited)` とある。Zen 側には Free の印も付いている。前作では、8/21 の「次の 6 日間」から 8/27 前後と逆算した。画面はまだ Unlimited のままなので、締切は画面からは読めない。期限は告知文の寿命で、UI の寿命とは限らない。

## 重みが出たら、指紋のほうは授業になる

今夜ウェイトを出す、は Bloomberg への回答であって、Hugging Face の URL ではない。ライセンスが商用を許すか、自己ホストに何枚 GPU が要るかは、ファイルを見てからでないと書けない。

確認で終わる話では、たぶんない。名前が付いたあとに残るのは、無料で 1 週間、開発者の作業ループに乗せるという手法のほうだ。Quasar のときより桁が違う。次のステルスが出たとき、界隈はまたスタックトレースを吐かせる。手順はもう公開されている。

自分のところでは、無料のうちに自分の課題を投げて、ログを残しておく。それだけは、正体が誰でも同じだった。

## あわせて読みたい

- [Ox Alpha を無料で試す4つの経路と、正体を推測するための材料の集め方](https://qiita.com/ishizakahiroshi/items/707b2da49958c27f5e2e)（前回の記事。無料 4 経路と、確認前の指紋の集め方）
- [OpenRouter を many-ai-cli に載せるなら、独立プロバイダにはしない](https://qiita.com/ishizakahiroshi/items/4feda29aa885644c982b)（今回の入口になった OpenRouter を、自作ツール側からどう抱えるかの話）
- [Claude Code / Codex / Cursor / Copilot / OpenCode で同じ Agent Skills を共有する](https://qiita.com/ishizakahiroshi/items/6821655d5af59a32e50c)（今回ログを集計した OpenCode を含む、複数 CLI のスキル棚の話）

## おわりに

名前が付いても、手元でやることは変わらない。自分の課題を投げて、自分のところで速いか正しいかを見る。

変わったのは、答え合わせができることだけだ。スタックトレースを吐かせて、エラーコードを他社ホストと比べて、トークナイザーを 95 個当てた人たちの手順は、授業として残る。自分のログは 1,710 万トークン分だけ残った。42 兆の横では点にしかならない。点でも、無料の窓の内側にいた記録にはなる。

小さく。次のステルスが来たら、また壊れたリクエストを 1 回投げてみる。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-08-26_ox-alpha-glm-reveal/

※ ヘッダー画像と本文の挿絵は AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
