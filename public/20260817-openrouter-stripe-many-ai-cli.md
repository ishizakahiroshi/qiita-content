---
title: OpenRouter を many-ai-cli に載せるなら、独立プロバイダにはしない
tags:
  - OpenRouter
  - stripe
  - ClaudeCode
  - CLI
  - 個人開発
private: false
updated_at: '2026-08-17T23:29:29+09:00'
id: 4feda29aa885644c982b
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-17_openrouter-many-ai-cli_hero.png)

2026 年 7 月、The Wall Street Journal が Stripe と OpenRouter の買収協議を報じました。売却額は 100 億ドル前後になり得る、という話です。

https://www.wsj.com/tech/ai/stripe-in-talks-to-buy-buzzy-ai-model-marketplace-openrouter-decc6a74

円に直すと、為替次第で 1 兆円を超える規模です。モデルを作っていない会社に、それだけ払うのか。最初はそう見えました。

その後、2026 年 8 月 16 日の Bloomberg 報道を TechCrunch が伝えています。協議は合意に進み、価格は 70 億ドル超だ、と。Stripe 側は TechCrunch に対し、噂や憶測にはコメントしないと答えています。成立した、と書いてしまうにはまだ早いです。

https://techcrunch.com/2026/08/16/stripe-will-reportedly-acquire-ai-gateway-startup-openrouter-for-7b/

https://www.bloomberg.com/news/articles/2026-08-16/stripe-nears-deal-to-buy-ai-firm-openrouter-for-over-7-billion

数字が動いても、見ている場所は同じです。OpenRouter は API の中継所であり、市場です。その位置が、手元で複数の AI CLI を束ねている many-ai-cli に刺さるのか。そこを考えました。

結論から書きます。載せる価値はある。ただし独立した 7 つ目のプロバイダとしては載せない。既存 CLI の接続先として差す。しかも第一選択にはしない。

![記事の要約（インフォグラフィック）](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-17_openrouter-many-ai-cli_infographic.png)

## 自作ツールを持っている方へ

自作で [many-ai-cli](https://github.com/ishizakahiroshi/many-ai-cli) という Web ダッシュボードを作っています。複数の AI コーディング CLI を並列で走らせ、承認をブラウザ 1 タブに集約。スマホからでも。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/many-ai-cli

同じ悩みを持っている方は、下記で入ります。

```bash
npm i -g many-ai-cli
```

入れたら 1 回だけ `many-ai-cli setup` を実行します。これでデスクトップにショートカットができて、次回以降はダブルクリックで起動します。`many-ai-cli` が見つからないと言われたら、シェルを開き直してください。

起動後の見え方は OS で違います。

Windows はショートカットが「Many AI Hub」の 1 個で、タスクトレイに常駐します。アイコンをクリックして「Hub を開く」を選ぶとブラウザが開きます（`http://127.0.0.1:47777`）。止めるのは同じトレイの「Hub を停止」です。

macOS と Linux は「Many AI Hub Start」と「Many AI Hub Stop」の 2 個です。Start で黒いコンソール窓とブラウザが一緒に開きますが、この窓が Hub の本体なので、閉じずに最小化してください。Linux の GNOME では初回だけ、ショートカットを右クリックして「起動を許可」を選ぶ必要があります。

ショートカットを使わず、ターミナルから直接立てても構いません。これは 3 OS 共通です。

```bash
many-ai-cli serve --open
```

ブラウザが開いたら、左下の「+ 新しいセッション」から claude / codex / copilot / cursor-agent / opencode / grok のどれかを選んで起動します。

正直に書いておくと、実機で動作を確認できているのは Windows です。ネイティブの macOS と Linux は動く想定で作っていますが、手元に環境がなく十分な検証ができていません。踏んだら Issue で教えてもらえると助かります。

この話には前があります。前回は AI コーディング CLI を 20 製品調べて、1 つも実装しないことにしました。候補の利用者数を並べる前に、自分のツールの分母を見ていなかったからです。

前回の記事: [対応 AI CLI を増やす前に、自分のツールの利用者数を測った方がいい](https://qiita.com/ishizakahiroshi/items/f491815a42b157f15932)

今回も「足す」話に見えて、実は同じ線の続きです。

Bloomberg は 70 億ドル超の合意を報じ、Stripe は公式には認めていません。OpenRouter はモデル会社ではなく API の料金所です。市場の本体は個人の月額より、企業が従量で回す API の方です。many-ai-cli に載せるなら独立プロバイダではなく、Ollama と同じ接続先。キーは Hub に置かない。第一選択にもしない。

## OpenRouter は、モデル会社ではない

OpenRouter は GPT も Claude も作っていません。OpenAI、Anthropic、Google、xAI など、各社のモデルを 1 本の API で呼べる中継所です。公式の価格ページは、従量課金プランで 500 以上のモデル、80 以上のプロバイダと書いています。

https://openrouter.ai/pricing

アプリ側は OpenRouter のキーを 1 本持てばよく、モデル名を変えるだけで行き先が変わります。API は OpenAI 互換の `/chat/completions` です。障害時のフォールバックや、安いプロバイダへの振り分けも、そちらの仕事です。

https://openrouter.ai/docs/faq

料金の取り方もはっきりしています。推論そのものには上乗せをせず、クレジット購入時に 5.5 パーセント（最低 0.80 ドル）を取る。暗号資産払いは 5 パーセント。自分のキーを持ち込む BYOK も、無料枠のあとに 5 パーセントが乗る。

https://openrouter.ai/docs/faq

モデル会社ではなく、料金所です。

## 70 億ドル超に見える理由は、売上倍率だけではない

2026 年 5 月、OpenRouter は Series B で 1 億 1300 万ドルを調達した、と公式ブログで発表しています。週次のトークン量は半年で 5 兆から 25 兆へ増え、400 以上のモデルを使う開発者は 800 万人を超えた、と書いてあります。評価額 13 億ドルは New York Times の報道です。

https://openrouter.ai/blog/announcements/series-b/

https://www.nytimes.com/2026/05/26/business/dealbook/openrouter-ai-models-fundraising.html

https://techcrunch.com/2026/05/26/openrouter-more-than-doubles-valuation-to-1-3b-in-a-year/

5 月の 13 億ドルから、7 月の「100 億ドル前後」へ。8 月の報道は「70 億ドル超」。どれも公式発表ではなく、関係者話です。それでも倍率は大きい。

TechCrunch は、CEO の Alex Atallah が New York Times の取材で、自社を AI 版 Stripe だと説明した、と書いています。入り口を 1 つにして、後ろの仕組みにはロックインされない、という言い方です。

https://techcrunch.com/2026/08/16/stripe-will-reportedly-acquire-ai-gateway-startup-openrouter-for-7b/

カード決済の Stripe が、Visa や Mastercard の前に立つ形に似ています。AI 側では、アプリの前に OpenRouter が立ち、その後ろに各モデル会社が並ぶ。どの会社が勝っても、通る量が増えれば料金所は太る。Stripe が欲しがるなら、そのオプション価値だと思います。

成立していない、という但し書きは残します。数字がまた動く可能性はある。

## 市場の本体は、月額 20 ドルの方ではない

「API を使っている人なんていない」は、個人の感覚としては正しい。手元では Claude も Codex も定額です。でも会社側の売上の置き場所は、そこではない可能性が高いです。

定額サブスクは、使えば使うほど、提供側の推論コストが先に膨らむ構造です。枠があるのはそのためだと思います。月 20 ドルでも 200 ドルでも、枠の中で Opus を回し続けると、API 定価に直した数字は月額をすぐに超えます。超えた分がそのまま提供側の赤字だとまでは、外から言い切れません。キャッシュや社内単価は公開されていません。ただ、枠で抑える設計になっていること自体は、定額が「使った分だけ儲かる」商品ではない、というサインです。

API は逆です。トークンごとに仕入れとマージンが乗った金額が請求されます。使われた分が売上になる。SaaS や IDE、社内システムは、利用者一人ひとりに Claude のサブスクを契約させるわけにいきません。サービス側が API を持ち、後ろでモデルを切り替える。人数が少なくても、1 社の開発予算から回収する方が大きい。

Anthropic 自身が、その大きさの一端を出しています。2026 年 2 月、Reuters は同社の年換算売上（run-rate）が 140 億ドル、Claude Code だけでも 25 億ドル超だと報じました。会社側の説明です。

https://www.reuters.com/technology/anthropic-valued-380-billion-latest-funding-round-2026-02-12/

同年 5 月 28 日の Series H 発表では、公式ブログが「今月初めに run-rate revenue が 470 億ドルを超えた」と書いています。

https://www.anthropic.com/news/series-h

https://www.reuters.com/business/anthropic-raises-65-billion-now-valued-965-billion-2026-05-28/

Reuters Breakingviews は、この run-rate の定義にも触れています。従量の顧客は直近 28 日を 13 倍し、サブスクは月額を 12 倍して足す。大企業が Anthropic の売上の 8 割で、その多くが消費量ベースだ、と。この 8 割は Anthropic 1 社の売上構成の話で、LLM 各社に共通する比率ではありません。瞬間風速なので、四半期の確定売上と同じ数字ではありません。それでも「一般ユーザーの月額」より「企業の従量」が大きい、という向きは公式の数字と矛盾しません。

https://www.reuters.com/commentary/breakingviews/anthropic-gives-lesson-ai-revenue-hallucination-2026-03-10/

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-17_openrouter-many-ai-cli_fig2.png)

左が個人の定額、右が企業の API です。70 億ドル超の報道が指しているのは右側です。自分の 1,600 ドルは左側の話で、混ぜると判断が崩れます。

だから Stripe が OpenRouter を欲しがる理由も、月 20 ドルのチャット課金を取りにいく話には見えません。世界中のアプリとエージェントが 24 時間 API を叩き、トークンが決済単位になる市場の料金所です。成立した、とはまだ書けません。見ている市場がそっちだ、という話です。

## 個人のヘビーユーザーには、API は高い

自分の画面では、ある日の Claude 側が API 定価換算で 1,600 ドルを超えていました。トークン量にしておよそ 20 億。これは Anthropic から請求される額ではなく、各トークンを API 価格に置き換えた概算です。キャッシュも、サブスク側の内部課金も違います。

それでも、月額のサブスクでこの使い方が許されているなら、全面的に API へ移る理由はありません。1 日で 20 万円超相当を、本当に従量で払っていたら話が終わります。「月 200 ドルの最上位で API 換算 8,000 ドル」といった話を見ることがありますが、出典を取れなかったのでここでは使いません。自分の画面の 1 日分の方が、根拠としては太いです。

OpenRouter の主戦場は、この使い方ではありません。

「API を使っている人なんていない」と「API の方が市場として大きい」は、同時に成り立ちます。財布が違うからです。

だから自分の手元では、OpenRouter を第一選択にしない。サブスクを先に使い、足りないモデルや障害時の逃げ先にする。順番を間違えると、請求が一気に跳ねます。提供側から見れば健全な従量でも、個人の口座から見れば洒落になりません。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-17_openrouter-many-ai-cli_illustration-01.png)

定額の通路が空いているのに、従量の窓口へ並び直す必要はない、という絵です。自分の使い方だと、並び直した瞬間に高くなります。

## 載せるなら、新しい CLI としては足さない

前回、新規プロバイダは人気や他社の対応状況では決めない、と書きました。OpenRouter は CLI ですらありません。wrap する TUI がない。承認を出す相手でもない。

では無関係か。そうでもない。many-ai-cli には、すでに同じ形の差し込み口があります。Ollama です。

README にも書いてあるとおり、Ollama は独立した wrapper ではありません。Claude や Codex の接続先として選び、Hub が `ANTHROPIC_*` / `OPENAI_*` をセッションごとに差します。

OpenRouter も、同じ棚に置けます。

```
Provider: Claude
  Backend:
    Native / Subscription
    Ollama Cloud
    Ollama Local
    OpenRouter   ← ここ
```

Claude Code 側は、公式に近い手順がすでに公開されています。OpenRouter 自身が 2026 年 6 月に書いた案内で、ベース URL を差し替えるだけです。ローカルプロキシは不要、と書いてあります。

https://openrouter.ai/blog/tutorials/claude-code-openrouter/

```bash
export OPENROUTER_API_KEY="<your-openrouter-api-key>"
export ANTHROPIC_BASE_URL="https://openrouter.ai/api"
export ANTHROPIC_AUTH_TOKEN="$OPENROUTER_API_KEY"
export ANTHROPIC_API_KEY=""
```

many-ai-cli がチャット画面を新しく持つ必要はありません。強みは PTY と承認検出と Hub です。LLM クライアントを自分で実装し始めたら、土俵がずれます。

Codex 側も、OpenAI 互換エンドポイントとして差せるなら同じです。独立プロバイダを 1 本足すより、既存 CLI の backend を 1 段増やすほうが、今の設計に合います。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-17_openrouter-many-ai-cli_fig1.png)

上の図は、土俵が違う、という話です。OpenRouter は API 同士を束ねる。many-ai-cli はサブスクの CLI とローカルと API を束ね、作業セッションまで持つ。競合ではなく、一段上です。

## ただし、キーは Hub に置かない

ここが実装するなら最初に止まる場所です。

many-ai-cli は、公式 CLI が設定ディレクトリを選ぶ環境変数を子プロセスへ渡すことだけをやっています。認証ファイルを読まない。token や API キーを `config.yaml` に持たない。持った瞬間に、このツールが鍵束になります。Copilot や Cursor の複数ログインを見送った理由と同じです。

OpenRouter を差すなら、キーは公式 CLI 側の設定か、ユーザーが自分で用意した環境変数に置く。Hub は向き先だけを変える。Ollama の `base_url` と同じ粒度です。キーの中身をパースして保存する口は、作らない。

存在しない向き先を、別アカウントへ黙って倒すこともやらない。エラーで止める。実行中セッションの認証を差し替える口も作らない。

「400 モデルが一気に使える」は魅力です。その魅力で、鍵束化を許すと後が続きません。

## 順番はサブスクが先

載せるなら、既定の優先順位はこうです。

1. 手元のサブスク CLI
2. ローカル（Ollama）
3. 最後に OpenRouter

OpenRouter を選んだ画面では、従量課金である旨を出した方がいい。自分の 1,600 ドルを見ていると、警告なしで押せるスイッチにはできません。

指揮者セッションが子を spawn するなら、安い調査だけ OpenRouter の軽量モデルへ逃がす、という使い方はあり得ます。それも「サブスクが先」を崩さない範囲です。

OpenRouter が AI API の Stripe なら、many-ai-cli は手元にある利用権を先に使い切って、足りなければ逃がすルーターです。同じ料金所を取りにいく話ではない。

実装するかどうかは、まだ決めていません。作者が日常でその経路を使い始めてから、でいい。人気や買収額を根拠に着手しない。前回と同じ基準です。

## 今回わかったこと

- OpenRouter はモデル会社ではなく、API の料金所である。公式は推論に上乗せせず、クレジット購入時に 5.5 パーセントを取る
- Stripe との話は、7 月が協議（100 億ドル前後）、8 月 16 日が合意報道（70 億ドル超）。Stripe は公式に認めていない
- 個人の定額と企業の API は財布が違う。Anthropic 公式は 2026 年 5 月に年換算売上 470 億ドル超と書いている。大企業が消費量ベースで大半を占める、という整理がある
- 個人のヘビーユーザーはサブスクが強い。自分の画面では 1 日の API 定価換算が 1,600 ドルを超えた
- many-ai-cli に載せるなら、独立プロバイダではなく Ollama と同じ backend 差し込み。キーは Hub に持たない
- 買収額やモデル数は、着手の根拠にしない

## many-ai-cli はこんなときに刺さります

- Claude Code と Codex と Grok を同時に走らせ、状態を 1 画面で見たい人
- 各 CLI の承認待ちを、ターミナルを行き来せず 1 タブで捌きたい人
- 離席中でも、スマホから承認だけしたい人
- サブスクを先に使い、API は逃げ先にしたいと考えている人

いずれかに心当たりがあれば、`npm i -g many-ai-cli` と `many-ai-cli setup` の 2 コマンドで試せます。設定ファイルを書く必要はありません。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/many-ai-cli
- npm: https://www.npmjs.com/package/many-ai-cli

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## あわせて読みたい

- [対応 AI CLI を増やす前に、自分のツールの利用者数を測った方がいい](https://qiita.com/ishizakahiroshi/items/f491815a42b157f15932)（前回の記事。今回の「独立プロバイダにはしない」の基準を置いた回）
- [Agent Plugins を自作ツールに取り込むか検討して、見送りました。配布モデルが逆向きだった](https://qiita.com/ishizakahiroshi/items/000383ea267490d25886)（仕様の出来ではなく、自分の設計との向きで見送った話）
- [many-ai-cli v0.7.0。承認パネルが握り潰される原因を138件のダンプから特定した](https://qiita.com/ishizakahiroshi/items/fc31553bf9427109d4dc)（今の Hub が何をしているかの技術版）

## おわりに

1 兆円に見える数字を見て、最初は「API の横流しにそんな値が付くのか」と思いました。料金所だと分かり、その料金所が見ているのが個人の月額ではなく企業の従量だと分かると、話の形が変わります。

手元のツールに足す話も、同じです。モデルを増やす話ではなく、向き先を増やす話。しかも自分の使い方では、その向き先は最後でいい。会社側が儲かる市場と、自分の口座が持つ定額は、別物です。

成立したかどうかは、公式発表を待ちます。そのあいだに実装を急ぐ必要はない。サブスクを先に使い切る、という順番だけは、先に決めておきます。

---

※ ヘッダー画像は AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。

