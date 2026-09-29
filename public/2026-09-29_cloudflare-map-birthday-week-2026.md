---
title: Cloudflareのサービス全体地図2026。Workers・Pages・D1から新CLI「cf」まで、個人サイト3つの実構成で読み解く
tags:
  - cloudflare
  - CloudflareWorkers
  - CloudflarePages
  - Wrangler
  - 個人開発
private: false
updated_at: '2026-09-29T19:13:48+09:00'
id: 5b83591f7e6e90ca53b6
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![Cloudflare全体地図。Birthday Week 2026と、個人サイト3つの実構成](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-29_cloudflare-map-birthday-week-2026/01_2026-09-29_cloudflare-map_hero.png)

「で、Cloudflare って結局いくつサービスがあるんだっけ」。

9 月 29 日の夕方、X に流れてきた Birthday Week 2026 のまとめ投稿を読みながら、声に出してつぶやいていました。新しい CLI の「cf」、Forge、Vinext 1.0、Kitesurf、EmDash 1.0、BEACON。名前は目に入るのに、自分のサイトとどうつながるのかが頭の中で組み上がらない。

読んでいたのは Fado（@fado_dev）さんのまとめです（https://x.com/fado_dev/status/2104851974353310193 ）。Cloudflare が 16 歳の誕生日を迎えて、今年は発表が「AI エージェント時代の開発基盤」に振り切っている、という整理でした。

自分は Cloudflare の上で 3 つのサイトを動かしています。ポートフォリオ兼ブログの ishizakahiroshi.com、進路選びの地図アプリ manabi-map、それと非公開で作っている地図の静的サイト。毎日のように触っているのに、全体を説明しろと言われたら詰まります。デプロイも、サイトごとにやり方がばらばらでした。

そこで、手元の 3 リポジトリの実際の設定と、Cloudflare の公式ドキュメント、Birthday Week の発表記事、X や Hacker News の反応をまとめて読み込みました。この記事はその棚卸しの記録です。

先に結論を置いておきます。

- Cloudflare のサービスは「サイトの前に立つ門番」「社内を守る仕組み」「網そのもの」「作って動かす開発者プラットフォーム」の 4 つの棚で見ると、迷子になりにくいです
- 個人開発で実際に触るのは、ほぼ開発者プラットフォームと、DNS・CDN まわりだけです
- 新 CLI「cf」は Wrangler の後継です。ただしまだ open beta で、Wrangler はベータ終了後も 18 か月保守されます。急いで乗り換える理由は、少なくとも自分の 3 サイトにはありませんでした
- 「手作業が多い」と感じていた原因は、CLI ではなく、push してもデプロイまで自動で進まない作りのほうでした。3 サイトのうち、Workers Builds で自動化済みの 1 つだけは手作業がほとんど残っていません

約 4 万字あるので、気になる章だけ拾い読みしてもらって大丈夫です。

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-29_cloudflare-map-birthday-week-2026/02_2026-09-29_cloudflare-map_infographic.png)

## 「多すぎて訳がわからない」は、棚が 4 つあると知れば半分片づく

Cloudflare の公式ドキュメントには、全サービスの一覧ページがあります（https://developers.cloudflare.com/directory/ ）。開いてスクロールしてみると分かりますが、項目が多すぎて、上から読んでいくと途中で何を探していたか忘れます。

もともとの Cloudflare は、CDN と DNS の会社でした。世界中に拠点を置いて、サイトの前に立ってキャッシュを返し、攻撃を受け止める。そこへセキュリティ製品が積み上がり、2017 年に Workers が発表されて（https://blog.cloudflare.com/introducing-cloudflare-workers/ ）、今では「アプリを作って動かす場所」としての顔のほうが、開発者には目立つようになっています。

歴史の順に製品が積み上がったので、一覧が長いのは当然なんですよね。なので、自分は次の 4 つの棚に分けて見ることにしました。

| 棚 | ひとことで | 代表的なサービス | 個人開発で触るか |
|---|---|---|---|
| アプリケーションサービス | サイトの前に立つ門番 | CDN、DNS、SSL/TLS、WAF、DDoS 防御、Turnstile | DNS と SSL は必ず触る。WAF は少し |
| Zero Trust（Cloudflare One） | 社内の人と端末を守る | Access、Tunnel、Gateway、WARP | Tunnel くらい |
| ネットワークサービス | 会社の網そのものを守る・つなぐ | Magic Transit、Cloudflare WAN、Spectrum | ほぼ触らない（Enterprise 向け） |
| 開発者プラットフォーム | 作って動かす | Workers、Pages、D1、KV、R2、Workers AI | ここが本丸 |

この外側に、公開 DNS リゾルバの 1.1.1.1（https://developers.cloudflare.com/1.1.1.1/ ）や、インターネットの状況を見られる Radar のような「周辺」があります。今週の Birthday Week の発表も、半分くらいはこの周辺か、開発者プラットフォームの端っこの話でした。

![Cloudflareの4つの棚と、個人開発で触る範囲](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-29_cloudflare-map-birthday-week-2026/03_2026-09-29_cloudflare-map_fig1.png)

図にすると、個人開発者が日常的に触るのは右下の開発者プラットフォームと、左上の門番のうち DNS・SSL・CDN のあたりだけです。残りの棚は「名前を見たら、ああ会社向けの棚ね」と分かれば十分で、中身まで覚える必要はありませんでした。

以下、棚ごとに見ていきます。料金や上限の数字は 2026 年 9 月 29 日時点の公式ページから取っていて、変わることがあるので、使う前に必ずリンク先を見てください。

## 門番の棚。サイトの前に立って、速くして、守ってくれる

独自ドメインを Cloudflare に向けた時点で、この棚の大半はもう働いています。Free プランの説明にも、無料の SSL、CDN、DDoS 防御が並んでいます（https://www.cloudflare.com/plans/free/ ）。

| サービス | 何をするか | Free プランでの扱い | 出典 |
|---|---|---|---|
| Cache / CDN | 世界中の拠点にコンテンツを置いて、近くから返す | 全プランで利用可 | https://developers.cloudflare.com/cache/ |
| DNS | 権威 DNS。ドメインの住所録 | 全プランで利用可 | https://developers.cloudflare.com/dns/ |
| SSL/TLS | 証明書を自動で発行して HTTPS にする | Universal SSL は全プラン無料 | https://developers.cloudflare.com/ssl/ |
| DDoS Protection | L3 から L7 までの攻撃を自動で吸収する | 全プランで従量課金なし | https://developers.cloudflare.com/ddos-protection/ |
| WAF | 怪しいリクエストをルールで止める | Free は Free Managed Ruleset とカスタムルール | https://developers.cloudflare.com/waf/ |
| Rate Limiting Rules | 一定時間のリクエスト数を制限する | ルール数は Free で 1 | https://developers.cloudflare.com/waf/rate-limiting-rules/ |
| Bots | 悪質な bot を判定して止める | Free は Bot Fight Mode | https://developers.cloudflare.com/bots/plans/ |
| Turnstile | CAPTCHA の代わりになる人間確認 | ウィジェット 20 個まで、チャレンジは無制限 | https://developers.cloudflare.com/turnstile/plans/ |
| Rules 系 | リダイレクト、ヘッダ書き換え、キャッシュ規則など | 全プランで利用可（数の上限は各ページ） | https://developers.cloudflare.com/rules/ |
| Page Rules | 旧来の URL ごとの設定 | Free で 3 つ。ドキュメントでは deprecated 表示 | https://developers.cloudflare.com/rules/page-rules/ |
| Web Analytics | Cookie を使わないアクセス解析 | 全プランで利用可 | https://developers.cloudflare.com/web-analytics/ |
| Registrar | ドメインを原価で登録・更新する | 購入型 | https://developers.cloudflare.com/registrar/ |
| Load Balancing / Argo | 複数オリジンへの振り分け、速い経路の選択 | 有料アドオン | https://developers.cloudflare.com/load-balancing/ |
| Waiting Room | 混雑時に順番待ちページへ誘導する | Business 以上 | https://developers.cloudflare.com/waiting-room/ |

Page Rules が deprecated になっているのは、地味に知っておいたほうがいい点です。昔の記事で「Page Rules でリダイレクト」と書いてあっても、今は Redirect Rules などの Rules 系で書くのが筋になっています。

自分のサイトで実際に使っているのは、この棚のうち DNS、SSL、CDN、Web Analytics、Registrar、それと本サイトの Bot Fight Mode とレート制限ルールくらいです。どれもダッシュボードで設定したきりで、日々意識することはほとんどありません。

Registrar でドメインを取っているのには、ちょっとした理由があります。ドメインと DNS を Cloudflare に置いておけば、ホスティングの側は「使い捨ての部品」にできるからです。実際、ポートフォリオを GitHub Pages から Cloudflare に引っ越したときも、DNS を触る範囲が小さくて済みました。

逆に、Turnstile は 3 サイトのどれでも使っていません。フォームを置いていないので、出番がないんです。

## 社内を守る棚と、網そのものの棚。個人ではほぼ触らない

Zero Trust は、今は Cloudflare One という名前の入口にまとまっています（https://developers.cloudflare.com/cloudflare-one/ ）。社員がどの端末からどのアプリに入れるかを制御する Access、社内の通信を検査する Gateway、端末の通信を Cloudflare 経由にする WARP、ブラウザを隔離して開く Browser Isolation、SaaS の設定ミスを見つける CASB、情報漏えいを検出する DLP、メールのなりすましを止める Email Security。会社の情シスが扱う棚です。

個人開発者が使うとしたら Cloudflare Tunnel でしょう。自宅のマシンに公開 IP を持たせなくても、`cloudflared` から Cloudflare へ外向きにつなぐだけで、中のサービスを外から見られるようにできます（https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/ ）。自宅サーバーを遊びで公開したい人には、この棚の中で唯一の実用品だと思います。

ネットワークの棚は、もっと遠い世界です。Magic Transit はネットワーク全体を DDoS から守る Enterprise 限定の製品で（https://developers.cloudflare.com/magic-transit/ ）、拠点どうしをつなぐ Magic WAN は Cloudflare WAN に名前が変わっていました（https://developers.cloudflare.com/magic-wan/ ）。任意の TCP/UDP を守る Spectrum も、全部入りは Enterprise のアドオンです（https://developers.cloudflare.com/spectrum/ ）。

この 2 つの棚は、今回の記事では「そういう棚がある」と分かれば十分、という扱いにします。自分も業務で触ることはあっても、個人のサイトで使ったことはありません。

## 開発者プラットフォーム。いちばんややこしくて、いちばんおもしろい棚

ここからが本題です。この棚だけでも数十の製品があって、しかも名前の変更や統合が多い。2026 年に入ってからも、Browser Rendering が Browser Run に（https://developers.cloudflare.com/changelog/post/2026-04-15-br-rename/ ）、AutoRAG が AI Search に（https://developers.cloudflare.com/changelog/post/2025-09-25-ai-search-more-models/ ）名前を変えています。

ややこしさの最大の原因は、同じことをする道が 2 本ある箇所が残っていることです。代表が Pages と Workers です。

### Pages と Workers、新しく作るならどっち

結論から言うと、公式は Workers を勧めています。Pages の概要ページの冒頭に、はっきり書いてありました。

> Start new projects with Workers.

出典: https://developers.cloudflare.com/pages/

Workers が Pages の使い道の大半をカバーし、機能も広い、という説明が続きます。Pages からの移行ガイドもあります（https://developers.cloudflare.com/workers/static-assets/migration-guides/migrate-from-pages/ ）。ガイドによると、静的ファイルへのリクエストは Workers でも Pages と同じく無料で、Pages Functions の呼び出しは Workers と同じ単価で課金されます。

ただし、Pages が非推奨（deprecated）になったとは、今回読んだ範囲のどこにも書いてありませんでした。第三者のブログには「Pages はメンテナンスモード」と書いているものもありますが、公式の一次情報では確認できていません。なので自分の理解は「新規は Workers、既存の Pages は急いで動かさなくていい」です。

移行ガイドの互換性の表を見ると、差分も見えてきます。Workers にしかないのは Cloudflare Vite plugin、Durable Objects、Cron Triggers、より詳しい観測まわり。逆に Pages にしかないか、Workers では部分対応なのが、ブランチごとのデプロイ制御、ブランチの別名、ファイルベースのルーティングです。

Pages には、もう 1 つ見落としやすい制約があります。

> If you deploy using the Git integration, you cannot switch to Direct Upload later.

出典: https://developers.cloudflare.com/pages/get-started/git-integration/

Git 連携で作ったプロジェクトは、あとから Direct Upload（手元で作ったファイルを直接上げる方式）へ切り替えられない、という話です。自動デプロイを止めて Wrangler で直接上げる道は残っていますが、プロジェクトの種類そのものは変えられません。これ、後で自分の manabi-map で効いてきます。

### 動かす場所

コードを動かす部品を並べます。

| サービス | 何をするか | 無料枠・料金の目安 | 出典 |
|---|---|---|---|
| Workers | サーバーレスでコードを世界中の拠点で動かす | Free は 1 日 100,000 リクエスト、CPU 10ms。Paid は月 5 ドルから | https://developers.cloudflare.com/workers/platform/pricing/ |
| Workers static assets | Worker に静的ファイルを同梱して配る | 静的ファイルへのリクエストは無料。ファイル数は Free 20,000、Paid 100,000 | https://developers.cloudflare.com/workers/static-assets/ |
| Pages | Git 連携か直接アップロードでサイトを出す | ビルドは Free で月 500 回。ファイル数は Free 20,000 | https://developers.cloudflare.com/pages/platform/limits/ |
| Pages Functions | Pages 上のサーバー処理 | Workers Free の 1 日 100,000 リクエストを共有 | https://developers.cloudflare.com/pages/functions/pricing/ |
| Cron Triggers | Worker を cron 式で定期実行する | Free はアカウントあたり 5 個 | https://developers.cloudflare.com/workers/configuration/cron-triggers/ |
| Durable Objects | 状態を持つ Worker。部屋ごとの同時編集などに向く | Free でも SQLite バックエンドなら使える | https://developers.cloudflare.com/durable-objects/platform/pricing/ |
| Queues | 重い処理を後回しにするメッセージキュー | Free で 1 日 10,000 操作 | https://developers.cloudflare.com/queues/platform/pricing/ |
| Workflows | 途中で落ちても続きから再開できる多段処理 | Free あり（Workers のリクエスト枠を共有） | https://developers.cloudflare.com/workflows/reference/pricing/ |
| Containers | Workers から呼べるコンテナ | Paid プランのみ | https://developers.cloudflare.com/containers/pricing/ |
| Sandbox SDK | 信頼できないコードを隔離して実行する | Paid のみ（Containers 準拠） | https://developers.cloudflare.com/sandbox/ |

Pages Functions が Workers の 1 日 100,000 リクエストを共有する、というのは要注意です。公式の説明では、Functions で 50,000 回、Workers で 50,000 回使えば、それで 1 日分を使い切る計算になります。静的ファイルは Functions を通らなければ無料・無制限なので、manabi-map ではここをかなり意識して作っています（後で書きます）。

### しまう場所

データを置く部品です。ここは名前だけ見ても違いが分かりにくいので、先に「何を置くか」で分けます。

- 設定値やキャッシュのような、読むことが多く書くことが少ない小さな値は KV
- 表の形をしたデータ、SQL で検索したいものは D1
- 画像、動画、バックアップのような大きいファイルは R2
- 部屋ごとの状態、チャットやゲームのような同時接続は Durable Objects
- すでにある外部の Postgres や MySQL を速く使いたいなら Hyperdrive

| サービス | 何をするか | Free の主な上限 | 出典 |
|---|---|---|---|
| Workers KV | 世界に配るキーバリューストア | 読み取り 1 日 100,000、書き込み 1 日 1,000、保存 1 GB | https://developers.cloudflare.com/kv/platform/pricing/ |
| D1 | SQLite 系のサーバーレス SQL データベース | 読み取り 1 日 500 万行、書き込み 1 日 10 万行、保存 5 GB | https://developers.cloudflare.com/d1/platform/pricing/ |
| R2 | S3 互換のオブジェクトストレージ。転送量（egress）が無料 | 保存 10 GB-月、Class A 操作 100 万回、Class B 操作 1,000 万回 | https://developers.cloudflare.com/r2/pricing/ |
| Hyperdrive | 既存の Postgres / MySQL への接続を速くする | 1 日 100,000 クエリ | https://developers.cloudflare.com/hyperdrive/platform/pricing/ |
| Vectorize | ベクトルデータベース | 保存 500 万次元 | https://developers.cloudflare.com/vectorize/platform/pricing/ |
| Workers Analytics Engine | 時系列のメトリクスを書いて SQL で読む | 書き込み 1 日 100,000 データポイント | https://developers.cloudflare.com/analytics/analytics-engine/pricing/ |
| Pipelines | ストリームデータを取り込んで R2 などへ流す | Free 枠なし | https://developers.cloudflare.com/pipelines/platform/pricing/ |
| R2 Data Catalog | R2 上の Apache Iceberg カタログ | public beta | https://developers.cloudflare.com/r2/data-catalog/ |
| Secrets Store | アカウント共通の秘密情報置き場 | open beta | https://developers.cloudflare.com/secrets-store/ |

KV の「書き込み 1 日 1,000 回」は、個人開発で一番最初に当たる壁かもしれません。自分の本サイトでは、まさにここに当たりました。詳しくは次の章で書きます。

### AI まわり

2026 年に一番動いているのがここです。

| サービス | 何をするか | 無料枠の目安 | 出典 |
|---|---|---|---|
| Workers AI | GPU で AI モデルをサーバーレスに動かす | 1 日 10,000 Neurons まで無料 | https://developers.cloudflare.com/workers-ai/platform/pricing/ |
| AI Gateway | AI API の呼び出しを記録・キャッシュ・制限する | コア機能は無料 | https://developers.cloudflare.com/ai-gateway/reference/pricing/ |
| AI Search（旧 AutoRAG） | 検索と RAG をまとめて面倒を見る | open beta 中は上限つきで無料 | https://developers.cloudflare.com/ai-search/platform/limits-pricing/ |
| Browser Run（旧 Browser Rendering） | ヘッドレスブラウザをクラウドで動かす | Free は 1 日 10 分、同時 3 ブラウザ | https://developers.cloudflare.com/browser-run/pricing/ |
| Agents SDK | 状態やスケジュールを持つ AI エージェントを作る | ページに料金の記載なし | https://developers.cloudflare.com/agents/ |

Birthday Week で発表された Kitesurf は、この Browser Run の上で使えるブラウザエンジンの 1 つ、という位置づけです。後の章で触れます。

### メディア、メール、それと道具

| サービス | 何をするか | 無料枠・料金の目安 | 出典 |
|---|---|---|---|
| Images | 画像の変換・最適化・配信 | 変換は月 5,000 種類まで無料 | https://developers.cloudflare.com/images/pricing/ |
| Stream | 動画の保存・エンコード・配信 | 保存と配信に従量課金 | https://developers.cloudflare.com/stream/pricing/ |
| Email Routing | 受信メールを転送先や Worker に振り分ける | 無料 | https://developers.cloudflare.com/email-routing/ |
| Email Service | メールを送る | Free では使えない。Paid に月 3,000 通込み | https://developers.cloudflare.com/email-service/platform/pricing/ |
| Wrangler | Workers 開発の公式 CLI | ツール | https://developers.cloudflare.com/workers/wrangler/ |
| Workers Builds | Git に push したら Cloudflare 側でビルドとデプロイ | Free は月 3,000 分、同時 1 ビルド | https://developers.cloudflare.com/workers/ci-cd/builds/limits-and-pricing/ |
| Workers Logs | Worker のログを保存して検索する | Free は 1 日 200,000 イベント、保持 3 日 | https://developers.cloudflare.com/workers/observability/logs/workers-logs/ |

Workers Builds と Pages の Git 連携は、似ているけど課金の単位が違います。Pages は「月あたりのビルド回数」、Workers Builds は「月あたりのビルド時間（分）」です（https://developers.cloudflare.com/workers/ci-cd/builds/limits-and-pricing/ ）。数字をそのまま比べられないので、ここは地味に混乱しました。

### Free と Paid、どこが違うのか

月 5 ドルの Workers Paid に上げると何が変わるのか。自分が気にしている行だけ抜き出すと、こうなります（https://developers.cloudflare.com/workers/platform/limits/ ）。

| 項目 | Workers Free | Workers Paid |
|---|---|---|
| 料金 | 無料 | 月 5 ドルから |
| リクエスト | 1 日 100,000 | 月 1,000 万込み |
| 1 リクエストの CPU 時間 | 10ms | 既定 30 秒、最大 5 分 |
| サブリクエスト（1 リクエストから外へ出す fetch など） | 50 | 10,000 |
| Worker の数 | 100 | 500 |
| Cron Triggers（アカウント） | 5 | 250 |
| 静的ファイル数（1 バージョン） | 20,000 | 100,000 |

サブリクエスト 50 回は、見落とすと痛い数字です。自分の本サイトでは、GitHub から記事やツール情報を取ってくる処理で、ここに引っかからないよう 1 回の処理件数を絞っています。

![開発者プラットフォームの部品の選び方と、最初に当たる無料枠の壁](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-29_cloudflare-map-birthday-week-2026/04_2026-09-29_cloudflare-map_fig2.png)

選び方を 1 枚にまとめると、「動かす場所は Workers を軸に、置くものの形で KV・D1・R2 を選ぶ」の一言になります。そして無料枠で最初に当たるのは、たいてい KV の書き込み、Functions を含むリクエスト数、サブリクエストの 3 つでした。

## うちの 3 サイトは、実際どの部品で動いているか

地図を眺めただけだと、どうしても他人事になります。なので、自分の 3 サイトが上の表のどこを使っているかを、リポジトリの設定ファイルから 1 つずつ拾い直しました。

先に一覧です。

| | ishizakahiroshi.com | manabi-map | 地図の静的サイト（非公開） |
|---|---|---|---|
| 配信 | Workers + static assets | Pages（Git 連携） | Pages（Direct Upload） |
| サーバー処理 | Worker 本体（Hono） | Pages Functions | なし |
| データ | KV + Cache API + D1 | Supabase（外部）。R2 はバックアップ用 | 静的 JSON |
| 定期処理 | Cron Triggers | GitHub Actions の cron | なし |
| デプロイ | Workers Builds（push で自動） | Git 連携を止めて組み替え中 | 手元から Wrangler で直接 |
| 最初に当たる無料枠の壁 | KV の書き込み、サブリクエスト | Functions のリクエスト数、ファイル数 | ファイル数、ビルド回数 |

見事に 3 つとも作りが違います。狙ってそうしたわけではなく、作った時期と目的がばらばらだった結果です。でも、だからこそ比較の題材としてはちょうどよかった。

### ishizakahiroshi.com。Worker 1 本に、サイトと統計を相乗りさせている

このサイトは、もともと 3 つのリポジトリに分かれていたものを 1 つにまとめた構成です。静的な HTML のポートフォリオと、自作ツールのダウンロード数を集計する API を、Worker 1 本にまとめて配っています。

最初は別々の Worker にするつもりでした。ところが、同じホスト名で Custom Domain と Workers Route を併用しようとすると「同じパターンのルートがすでにある」という競合が出るうえ、Custom Domain を外すと DNS レコードまで一緒に消える、という罠を踏みました。それで「Worker は 1 本、ドメインもそこに直接付ける」に倒しています。

設定ファイルの骨格はこんな感じです（ID などの値は伏せています）。

```toml
name = "<worker-name>"
main = "worker/index.ts"
compatibility_date = "2024-12-18"
compatibility_flags = ["nodejs_compat"]

[assets]
directory = "./combined"
binding = "ASSETS"
run_worker_first = true

[[kv_namespaces]]
binding = "CACHE"
id = "<kv-namespace-id>"

[[d1_databases]]
binding = "DB"
database_name = "<database-name>"
database_id = "<database-id>"

[triggers]
crons = ["0 3 * * *"]
```

部品ごとの使い方を並べます。

- **static assets**: ビルド時に、静的サイトのファイルと管理画面の出力を 1 つのフォルダへ集めて配っています。`run_worker_first = true` にしているのは、全レスポンスにセキュリティヘッダを付けたいのと、一部のパスを Worker 側で振り分けたいからです
- **Worker 本体**: Hono で組んだ API です。ツールの統計、作品情報、記事一覧を JSON で返します
- **KV**: 外部 API の結果をキャッシュしています。期限が切れたら古い値をいったん返し、裏で `waitUntil` を使って取り直す、stale-while-revalidate の形です
- **Cache API**: KV の手前にもう 1 層置いています。拠点ごとのキャッシュにヒットすれば、KV の読み取り回数を使わずに済みます
- **D1**: ダウンロード数の日次スナップショットと、簡単な足あと（国・参照元・パスだけで、IP は持ちません）を入れています
- **Cron Triggers**: 1 日 1 回、日本時間の 12 時に走って、キャッシュを温め、スナップショットを書き、古い足あとを消しています
- **Web Analytics**: 訪問数を GraphQL で取り、国別の数字として API から返しています

記事の本文は、少し変わった配り方をしています。記事ページはデプロイ物に含めず、Worker が GitHub の main ブランチから取ってきて返します。GitHub の Contents API をトークン付きで呼び、取っていいパスは許可リストで絞り、`..` による遡りは拒否。取ってきた本文は KV に 12 時間キャッシュします。

これで「記事は push した時点で公開される」ようになりました。便利なんですが、1 つ罠があります。公開済みの記事を直したとき、KV に古い本文が 12 時間残るんです。削除用のエンドポイントはあえて作っていないので、キャッシュのキーに埋め込んだ版番号を 1 つ上げてデプロイする運用にしています。これを忘れて、直したはずの記事が半日古いまま、というのを 2 回やりました。

**無料枠で実際に当たった壁**もあります。KV の書き込み上限（1 日 1,000 回）です。記事が 130 本を超えたあたりで、クローラーが全記事を 1 周しただけで上限に届き、「KV の 1 日の無料枠の 50% を使いました」という通知が来ました。キャッシュの期限を 12 時間に延ばして、書き込みの回数そのものを減らして落ち着かせています。月 5 ドルの Paid に上げれば一瞬で消える話ではあるんですが、まずは設定で解決する、を選びました。

サブリクエストの 50 回も、同じく意識しています。1 日 1 回の cron で全ツールの情報を取り直そうとすると、GitHub への問い合わせが 50 回を軽く超えます。なので、1 回の起動で見に行くのは 10 件までにして、日替わりで順番に回しています。

**デプロイ**は Workers Builds の Git 連携です。main に push すると、Cloudflare 側で `pnpm run build` と `wrangler deploy` が走ります。今日（9 月 29 日）の実績でいうと、15 時 59 分 33 秒の push に対して、16 時 00 分 53 秒にデプロイが記録されていました。81 秒です。手元で `pnpm run deploy` する道も残していますが、自動ビルドが失敗したとき用の予備です。

1 点だけハマったのは、Workers Builds のビルド環境の pnpm が 10 系で、手元の 11 系と挙動が違ったことです。`pnpm-workspace.yaml` に `packages:` が無いと install の段階で落ちました。ビルド環境のバージョンは、手元と同じとは限りません。

### manabi-map。Pages と Functions と Supabase の組み合わせ

manabi-map は、通える高校を地図で見ながら親子で決めるための Web アプリです。[Web アプリ](https://manabi-map.app)として公開していて、[GitHub](https://github.com/ishizakahiroshi/manabi-map) でソースも公開しています。概要は[作品紹介](https://ishizakahiroshi.com/work.html?id=manabi-map)にまとめています。

React と Vite で作った画面を Cloudflare Pages で配り、ログインや利用者データは Supabase に置いています。Cloudflare 側で使っている部品はこうです。

- **Pages（Git 連携）**: main に push したら本番、develop に push したらプレビュー、という運用で作りました
- **Pages Functions の `_middleware`**: メンテナンス中は 503 を返す、地図などの SPA のルートに index.html を 200 で返す、緯度経度を含む URL に `noindex` を付ける、といった振り分けを担当しています
- **Pages Functions の API**: 管理画面用の API です。Supabase の認証で管理者かどうかを確かめ、違えば 404 を返します
- **`_headers`**: セキュリティヘッダと CSP です。ファイル名にハッシュが付いたアセットは 1 年キャッシュ、データの目録は no-store にしています
- **`_routes.json`**: 静的ファイルを Functions の対象から外しています。理由は後述
- **R2**: 毎晩のデータベースのバックアップ置き場です。GitHub Actions で `pg_dump` を取り、暗号化して、S3 互換 API で R2 に置いています。binding ではなく S3 API 経由です
- **Web Analytics**: 自動設定で有効にしていて、GitHub Actions の日次ジョブが GraphQL で訪問数を取り、Supabase に記録しています

SPA のルートを `_redirects` ではなく middleware で返しているのには、痛い理由があります。`_redirects` に `/map /index.html 200` と書くと、Pages がそれを `/` への 308 リダイレクトとして正規化してしまうんです。しかも 308 は恒久リダイレクトなのでブラウザに残り、直しても既存の利用者には届かない。この顛末は別の記事にまとめています（https://qiita.com/ishizakahiroshi/items/0b4e0b146ec0cbeb00fe ）。

`_routes.json` は無料枠の話です。Pages Functions のリクエストは Workers Free の 1 日 100,000 回を共有するので、何も書かないと、画像や JS のような静的ファイルへのアクセスまで Functions を通って枠を食います。静的ファイル 5,000 件あまりのほぼ全部を対象外にして、Functions を通るのはページと API だけにしました。

自分の試算では、これで「無料枠の壁」が月 25 万 PV あたりから、月 130 万 PV あたりまで遠のきました。あくまで自分の手元の試算で、アクセスの構成が変われば変わります。次の壁は Pages のファイル数（Free で 20,000）で、学校の種類を広げていくと、いずれここに当たります。

そして正直に書いておくと、manabi-map は今、デプロイの経路を組み替えている途中です。学校データの作り方を SQLite の原本から静的ファイルを作る形へ移していて（その設計は https://qiita.com/ishizakahiroshi/items/0ae508ed2ee095761154 に書きました）、その準備として Pages の自動デプロイを止め、手元で作った成果物を明示的に上げる形へ切り替えています。

さっきの「Git 連携で作ったプロジェクトは Direct Upload に切り替えられない」が効いてくるのが、ここです。プロジェクトの種類は Git 連携のままなので、自動デプロイを止めて Wrangler で上げるか、新しいプロジェクトを Direct Upload で作り直すかの 2 択になりました。

困ったことに、組み替えの途中で、手順書のほうが追いついていません。README 以外の運用文書には、まだ「main に push すれば本番に出る」と書いてあります。今そのとおりに作業すると、本番に出たと思い込む。今回の棚卸しで一番冷や汗が出たのは、ここでした。

### 地図の静的サイト。Pages に直接アップロードするだけの、いちばん小さい構成

3 つ目は非公開で作っている地図のサイトで、名前やアドレスはここでは伏せます。Leaflet と静的 JSON だけで動く、ビルド工程もほぼ無いサイトです。

Cloudflare 側は、Pages の Direct Upload と `_headers` だけ。Functions も KV も D1 も R2 も使っていませんし、アクセス解析も入れていません。「静的な Pages に、D1 や R2 や有料の Worker を足さない」を運用ルールとして最初に決めています。住所や精密な座標を扱う可能性がある題材なので、解析のイベントに位置を乗せる余地そのものを作らない、という判断です。

そのぶん、ビルドは慎重に作っています。

- 公開してよいファイルを許可リストで持ち、それ以外が出力に混ざったらビルドを失敗させる
- 出力に `version.json` を置いて、Git の SHA と各ファイルの SHA-256 を記録する
- 公開後に照合用のスクリプトを走らせ、公開版の版番号とハッシュ、CSP などのヘッダ、非公開にしたいパスが 404 になることを確かめる
- 戻すときは Pages の Deployments から前の版に戻す

デプロイは手元から Wrangler で打ちます。

```bash
npm run build
revision=$(git rev-parse HEAD)
wrangler pages deploy dist --project-name <project-name> --branch main --commit-hash "$revision"
```

構成は一番小さいのに、手作業は一番多い。これが、次の章の話につながります。

![3サイトの構成比較と、最初に当たる無料枠の壁](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-29_cloudflare-map-birthday-week-2026/05_2026-09-29_cloudflare-map_fig3.png)

並べてみると、使う部品が増えるほど手作業が減る、という逆転が起きています。Workers に寄せた本サイトが一番多くの部品を使い、一番自動化されていて、部品の少ない地図サイトが一番手で打つコマンドが多い。部品の多さと運用の重さは、比例しないんですね。

## 手作業が多いのは、道具のせいじゃなかった

今回の調べものは、もともと「Cloudflare の新しい CLI を使えば、デプロイの手作業を減らせるんじゃないか」という期待から始まりました。地図サイトのデプロイ手順を書き出してみると、こうなっていたからです。

1. commit して push する。CI がテストとビルドをする
2. CI が通ったかを確認する
3. 手元で同じ commit をビルドし直す
4. 手元の成果物のハッシュが、CI のビルド結果と一致するかを目で確かめる
5. `wrangler whoami` と Pages のプロジェクト一覧で、ログイン先を確認する
6. 戻し先にする今の正常なデプロイを控えておく
7. `wrangler pages deploy` を commit hash 付きで打つ
8. 公開版を照合スクリプトで確かめる
9. 画面を開いて目視する
10. デプロイの ID と確認結果を作業メモに書く

10 手順。このうち、CLI を別のものに替えて消えるのはどれか、と数えてみました。

ゼロでした。

手順 3 と 4 は、CI で作った成果物をそのまま上げていないから発生しています。5 と 7 は、デプロイできるのが手元の PC だけだから発生しています。8 は、デプロイの後に照合が自動で続かないから。どれも「push からデプロイまでが 1 本の自動の流れになっていない」ことが原因で、コマンドの名前が `wrangler` だろうと `cf` だろうと、手数は変わりません。

実際、push すれば 81 秒で本番に出る本サイトには、この種の手作業がほとんど残っていません。道具は同じ Wrangler です。違いは、デプロイを誰が起動するかだけでした。

![荷物は詰め終わっているのに、配送が来ない夕方の仕事部屋](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-29_cloudflare-map-birthday-week-2026/06_2026-09-29_cloudflare-map_illustration-undelivered.png)

push は、荷造りまでです。玄関に箱を積んでも、配送を呼ばなければ届かない。自分がやっていたのは、毎回自分で車を出して届けに行くことでした。過去に、push 済みの機能が本番では 404 のまま、という時期があったのも、この構造のせいです。

じゃあ配送をどう自動にするか。選択肢は 3 つあります。

| 方法 | 動く場所 | 向いているとき | 気をつけること |
|---|---|---|---|
| Workers Builds | Cloudflare 側 | Workers で、ビルドが素直なとき | ビルド環境のツールの版が手元と違うことがある |
| Pages の Git 連携 | Cloudflare 側 | Pages で、ビルドが素直なとき | 作った後で Direct Upload に切り替えられない |
| GitHub Actions + Wrangler | GitHub 側 | 自前の検査や照合をデプロイの前後に挟みたいとき | API トークンを GitHub に預ける必要がある |

地図サイトの場合は、CI でビルドした成果物をそのまま上げて、そのまま照合まで回したいので、GitHub Actions が合います。書くとしたらこうなります（まだ入れていない、書きかけの案です）。

```yaml
  deploy:
    needs: validate
    runs-on: ubuntu-24.04
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          name: public-${{ github.sha }}
          path: dist
      - name: Deploy to Pages
        run: npx wrangler@4 pages deploy dist --project-name "$PROJECT" --branch main --commit-hash "$GITHUB_SHA"
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          PROJECT: ${{ vars.PAGES_PROJECT }}
      - name: Verify public site
        run: node scripts/verify-public.mjs "$PUBLIC_URL"
        env:
          PUBLIC_URL: ${{ vars.PUBLIC_URL }}
```

`CLOUDFLARE_API_TOKEN` と `CLOUDFLARE_ACCOUNT_ID` の環境変数は、Wrangler が CI で読む名前です（https://developers.cloudflare.com/workers/ci-cd/external-cicd/github-actions/ ）。トークンは Pages の編集だけに権限を絞って作ります。

ただ、ここには自分で決めないといけないことが 1 つあります。地図サイトの運用ルールには「認証が失敗しても、再ログインや権限の拡張はしない」と書いてあって、トークンを GitHub に預けるのは、そのルールを変えることになるんです。便利さと、秘密を置く場所を 1 つ増やすことの交換。これはツールでは決まらない話でした。

もう 1 つ、今回見つけた地味な問題があります。Wrangler の版が、リポジトリごとにばらばらだったことです。本サイトは `package.json` に `^4.131.0` を入れていますが、地図サイトと manabi-map はリポジトリに Wrangler を入れておらず、PC にグローバルで入れた 4.127.1 や、npx が拾った 4.131.1 で動いていました。npm の最新は 4.143.0 です（https://registry.npmjs.org/wrangler ）。

どの版でデプロイしたかが日によって変わるのは、再現性の面で気持ちが悪い。CLI を替える前に、まず今の CLI の版を `package.json` で固定するほうが先でした。

## Birthday Week 2026 で出たものを、全部見ていく

ここからが、今週の話です。Cloudflare は毎年 9 月末に Birthday Week として、誕生日に「もらう側が配る」形で発表を並べます。今年は 16 回目で、10 月 2 日まで続きます。

ちなみに、Workers が発表されたのは 2017 年 9 月 29 日でした（https://blog.cloudflare.com/introducing-cloudflare-workers/ ）。この記事を書いている今日で、ちょうど 9 年です。

9 月 29 日の時点で、Birthday Week のタグ（https://blog.cloudflare.com/tag/birthday-week/ ）に並んでいた 2026 年分の記事は 10 本でした。

| 日付 | 記事 | URL |
|---|---|---|
| 9/27 | 創業者レター（2026 Annual Founders' Letter） | https://blog.cloudflare.com/cloudflares-2026-annual-founders-letter/ |
| 9/28 | 新 CLI「cf」の open beta | https://blog.cloudflare.com/cloudflare-cf-cli-launch/ |
| 9/28 | Forge（SDK・CLI・ドキュメントの生成パイプライン） | https://blog.cloudflare.com/forge-open-source-generation-pipeline/ |
| 9/28 | Vinext 1.0（Next.js を Vite で動かす） | https://blog.cloudflare.com/vinext-nextjs-on-vite/ |
| 9/28 | Kitesurf の続報（AI エージェント用ブラウザ） | https://blog.cloudflare.com/kitesurf-update/ |
| 9/28 | EmDash 1.0（Astro ベースの CMS） | https://blog.cloudflare.com/emdash-cms-plugin-registry/ |
| 9/28 | BEACON（実ユーザー計測データの公開） | https://blog.cloudflare.com/how-fast-is-the-web/ |
| 9/28 | VoidZero の 4 か月報告（Vite / Vitest / Oxc など） | https://blog.cloudflare.com/voidzero-update/ |
| 9/28 | Rust Workers の Emscripten ターゲット対応 | https://blog.cloudflare.com/rust-workers-emscripten-target/ |
| 9/28 | The Cold Start（スタートアップのピッチイベント） | https://blog.cloudflare.com/introducing-the-cold-start/ |

![Birthday Week 2026の発表と、自分のサイトへの関係](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-29_cloudflare-map-birthday-week-2026/07_2026-09-29_cloudflare-map_fig4.png)

図は、発表を「4 つの棚のどこに入るか」と「自分の 3 サイトにどれくらい関係するか」で並べたものです。関係が一番近いのは cf で、それでも「今は待ち」。ほかは、今のサイトには直接効かないものが大半でした。

1 本ずつ見ていきます。

### cf。Wrangler の後継は、人間より先に AI のほうを向いている

発表記事の冒頭に、今回一番おもしろかった数字があります。Wrangler の利用のうち、AI エージェントによるものの割合です。2025 年は 1 桁パーセント、2026 年 3 月に 4 分の 1、そして発表の前の週に、

> Last week, agent usage reached 48%.

出典: https://blog.cloudflare.com/cloudflare-cf-cli-launch/

ほぼ半分です。自分も、Wrangler を打っている回数は、手で打つより Claude Code に打たせているほうが多い自覚があります。なので、この数字には妙に納得しました。

cf の中身を整理します。

- **Cloudflare の API 全体を 1 つの CLI で**: Wrangler が積み上げてきた約 280 の機能に対して、cf は 3,000 を超える API 操作をカバーします
- **JSON が既定の出力**: 人間向けには整形、エージェント向けには圧縮して出します。Wrangler では一部のコマンドしか `--json` に対応しておらず、エージェントが表をパースして苦労していた、という反省から来ているそうです
- **`cf cli search`**: やりたいことを自然言語で聞くと、API の説明とパラメータから作った小さな索引が、合いそうなコマンドを返します。3,000 のコマンドを全部コンテキストに入れずに済ませる工夫です
- **設定が TypeScript になる**: `wrangler.toml` や `wrangler.jsonc` の代わりに `cloudflare.config.ts` を書きます。型が付くので、エージェントが設定を書き間違えにくくなる、という狙いです
- **Vite が既定**: ローカル開発とビルドは Cloudflare Vite plugin を通すのが標準になります
- **インストール**: `npm i -g cf`

コマンドの命名も、揃えにいっています。4 月の技術プレビューの記事（https://blog.cloudflare.com/cf-cli-local-explorer/ ）には、こんな一文がありました。

> It's always get, never info.

Wrangler だと、D1 は `info`、Hyperdrive は `get`、Workflows は `describe` と、同じ「詳細を見る」でも動詞がばらばらでした。人間は慣れで覚えますが、エージェントは毎回迷います。そこを揃える、という話です。

経緯も押さえておきます。cf は 4 月 13 日の Agents Week で技術プレビューとして出ていて、npm の `cf` 0.0.4 も同じ日に公開されています。9 月 25 日に 1.0.0-beta.0、9 月 28 日に 1.0.0-beta.5 が出て、今回の open beta です（https://registry.npmjs.org/cf ）。

#### 自分のサイトの設定を、cf の形に書き直すとどうなるか

発表記事の書き方にならって、本サイトの設定を `cloudflare.config.ts` に置き換えるとしたら、たぶんこうなります。**cf で動かして確かめたものではなく、記事の説明から起こした下書き**です。

```ts
// 本サイトの wrangler.toml を cf の設定へ書き直した想像図（未検証）
import { bindings, defineConfig, triggers } from "cf/config";
import * as entrypoint from "./worker/index.js" with { type: "cf-worker" };

export default defineConfig(({ mode }) => ({
  worker: {
    name: "<worker-name>",
    entrypoint,
    compatibilityDate: "2026-09-27",
    env: {
      CACHE: bindings.kv({ id: "<kv-namespace-id>" }),
      DB: bindings.d1({ name: "<database-name>" }),
      GITHUB_TOKEN: bindings.secret(),
    },
    triggers: [triggers.scheduled({ schedule: "0 3 * * *" })],
  },
}));
```

`mode` を使えば、本番とステージングの値を三項演算子で書き分けられます。TOML の `[env.staging]` を何ブロックも並べるより読みやすいのは確かです。発表記事によると、Cloudflare 社内では 5,000 行を超えていた Wrangler の設定が、この形にして 40% 縮んだ例があるそうです（https://blog.cloudflare.com/cloudflare-cf-cli-launch/ ）。

#### 移行は `cf migrate` 一発、とは限らない

移行コマンドは `cf migrate` です。ただし、発表記事の説明をよく読むと条件があります。

- すでに Vite でビルドしている Worker は、`cloudflare.config.ts` へ変換される
- esbuild でのビルドを続ける必要がある JavaScript の Worker と、Rust や Python の Worker は、開発とデプロイを Wrangler に任せ続ける

つまり、Vite に乗っていない Worker は、cf を使っても中身は Wrangler が動きます。自分の本サイトの Worker 本体は Wrangler の esbuild でバンドルしているので、今 `cf migrate` しても実質的な変化は小さい、というのが自分の読みです。

Wrangler の今後についても、はっきり書いてありました。

> maintenance support for Wrangler for 18 months after the beta ends

ベータが終わったら、cf へ誘導する最後のメジャー版の Wrangler を出し、その後 18 か月は保守を続ける。ベータ終了の時期はまだ書かれていません。なので「Wrangler が廃止された」わけではないです。ここを取り違えると、慌てて本番の経路を替えることになります。

#### 先に移行した人、保留した人

発表から 1 日で、GitHub には移行の issue がいくつも立っていました。

保留を決めた例が skanehira/chatbook の issue です（https://github.com/skanehira/chatbook/issues/34 ）。ビルド、ローカル開発、dry-run のデプロイまでは通ったうえで、本番の経路が変わるのに依存が beta のままなので正式版まで待つ、という判断でした。詰まりどころとして、Node 26 でコマンドが止まったまま返らない（Node 24 では 1 秒で終わる）、`wrangler tail` に相当するログの流し見が見つからない、D1 の migration がデータベース名ではなく ID の指定になる、などが挙がっています。再開条件は「Vite plugin が 2.x の安定版になり、cf が beta でない 1.0 以上になったら」。この条件の立て方は、そのまま真似したいと思いました。

逆に、Wrangler を完全に外した例もあります（https://github.com/Nishfleet/siterep-public/issues/131 ）。CI では `CLOUDFLARE_API_TOKEN` と `CLOUDFLARE_ACCOUNT_ID` をそのまま読むので、シークレットは移し替え不要だったそうです。ただ、Workflows と Durable Object の migration の設定が TODO のまま残り、手で書き直しています。

この Workflows の件は、Cloudflare 側でもバグとして報告されていて（https://github.com/cloudflare/workers-sdk/issues/15925 ）、修正の PR も出ています。発表から 1 日の状態なので、これを読むころには直っているかもしれません。

移行して数字が出ている例もありました。Kotlin/JS の Worker を移した例では、バンドルが 204.7 KiB から 172.5 KiB に減っています（https://github.com/mataku/mataku.today/pull/281 ）。Vite と Rolldown の tree-shaking が効いた形です。

#### Hacker News と X の空気

Hacker News のスレッド（https://news.ycombinator.com/item?id=49879577 ）では、cf が TypeScript 製であることに議論が集中していました。依存関係を利用者に管理させるな、という立場から、

> Do write your cli in a compiled language.

という声がある一方で、1 ファイルにバンドルすれば済む、エージェントがコードを書いて動かす用途なら JS / TS はむしろ向いている、という反論も付いていました。設定ファイルが TypeScript（つまり実行されるコード）になることへの違和感も、複数の人が書いています。ここは自分も、エージェントが書いた設定ファイルを読み込んだ瞬間に任意のコードが動く、という点は頭の片隅に置いておきたいです。

日本語圏では、Mojofull（@furoku）さんが発表の翌朝に図解つきのまとめを出していました（https://x.com/furoku/status/2104647537852576183 ）。Wrangler からの世代交代に「一瞬ざわつきました」と書きつつ、読み込んだら AI と一緒に作る方向への本気の移行だった、という受け止め方です。自分も最初にざわついた側なので、この温度感はよく分かります。

4 月の技術プレビューのときにも反応はありました。Ashley Peacock さんは、Wrangler に入ったローカルのデータ閲覧機能と、Cloudflare 全体を扱う新しい CLI の予告を紹介していて（https://x.com/_ashleypeacock/status/2043707612907086041 ）、brandon（@burcs）さんは「Wrangler のコマンドに縛られない本物の Cloudflare CLI」が来たと喜んでいました（https://x.com/burcs/status/2043737356578992455 ）。半年前の時点で、もう期待は高かったわけです。

#### 自分の 3 サイトでの判断

| サイト | cf にすると何が変わるか | 判断 |
|---|---|---|
| ishizakahiroshi.com | Worker 本体が esbuild ビルドなので、中身は Wrangler に任されたまま。設定が TypeScript になる恩恵はある | 待つ |
| manabi-map | Pages を使っている。静的な Pages サイトは別のコマンドが要るという報告があり、自分では未確認 | 組み替えが終わるまで触らない |
| 地図の静的サイト | Direct Upload の 1 コマンドが置き換わるだけ | 待つ。先に CI デプロイ化 |

どれも「待つ」になりました。再開の条件は chatbook の issue にならって、cf が 1.0 の正式版になること、Vite plugin が安定版になること、自分が使う D1 と Pages の操作が名前で素直に打てること、の 3 つにしておきます。

### Forge。cf を生んだ工場ごと、オープンソースにした

Forge は、API の定義（OpenAPI）から SDK、CLI、ドキュメントなどを生成するパイプラインです。cf の中身は、この Forge で生成されています。

発表記事（https://blog.cloudflare.com/forge-open-source-generation-pipeline/ ）の説明では、各チームの API リポジトリの CI で動き、変換器（transformer）を差し替えたり連鎖させたりして、いろいろな出力を作れる作りです。現時点で確実に出せるのは cf の生成物で、API ドキュメントや SDK も今後数か月でこれに乗せていく、と書かれています。

オープンソースにした理由として、こんな一文がありました。

> You should own your SDKs, CLIs, and docs.

リポジトリ（https://github.com/cloudflare/forge ）は Apache-2.0 で、README に書かれている使い方は、TypeScript SDK の生成までです。

```bash
pnpm add --save-dev @cloudflare/forge-transformer-sdk-ts
pnpm exec forge openapi.json --out ./generated
```

README には、完全な生成には Docker が要るとも書かれています。「Terraform や MCP サーバーも生成できる」と紹介している記事もありますが、手順が書いてあるのは今のところ SDK だけで、そこから先は予定の範囲と読むのが安全です。

背景として、The New Stack は、Anthropic が買収した Stainless が SDK 生成サービスを終了したことと、Forge の公開を並べて報じています（https://thenewstack.io/cloudflare-forge-anthropic-stainless/ ）。自分が確認できたのは見出しまでで、本文は読めていません。

反対側の意見も 1 つ置いておきます。Forge への反応ではなく、5 月の投稿ですが、sam（@samgoodwin89）さんは、仕様書はどれも間違っていて不完全なので、SDK の生成は固い生成器よりコーディングエージェントに任せたほうがいい、と書いていました（https://x.com/samgoodwin89/status/2054408134752604599 ）。エラー時の振る舞いこそ仕様書に書かれていない、という指摘は、実務の感覚としてかなり分かります。

自分への関係は、今は薄いです。ただ、manabi-map には学校データを返す公開 API があるので、OpenAPI の定義を書けば、Forge で利用者向けの SDK やドキュメントを作れることになります。いつかやるかもしれない引き出しとして、しまっておきます。

### Vinext 1.0。1 週間の実験が、7 か月で本番用になった

Vinext は、Next.js のアプリを Vite の上で動かすフレームワークです。始まりは 2 月の実験で、1 人のエンジニアが AI を使って 1 週間で Next.js を作り直した、という記事でした（https://blog.cloudflare.com/vinext/ ）。

当時の反応は、驚きと疑いが半々だったと思います。syumai（@__syumai）さんは、ブログの「Next.js の drop-in replacement」という一文を引用して、「⁉️」だけを付けていました（https://x.com/__syumai/status/2026440223157256411 ）。Hacker News でも、その主張への疑いや、長く保守できるのかという声が出ています（https://news.ycombinator.com/item?id=47142156 ）。InfoQ は、Cloudflare 自身が「実験的で、まだ本格的なトラフィックで試されていない」と書いていたことを伝えています（https://infoq.com/news/2026/03/cloudflare-vinext-experimental ）。

保守の疑問には、1 か月後に Steve Faulkner さんが答えていました（https://x.com/southpolesteve/status/2036452002318692380 ）。トリアージも修正もレビューも AI で回していて、最初の 1 か月で PR 395 件をマージ、リリースは 27 日で 34 回。7 月には 1.0 のベータが出ています（https://x.com/James_Elicx/status/2073457440461303836 ）。

そして今回の 1.0 です（https://blog.cloudflare.com/vinext-nextjs-on-vite/ ）。App Router だけでなく Pages Router と、その混在にも対応し、主要な機能のテスト互換性について、

> test compatibility has risen to more than 99%, excluding cache components.

と書いています。`"use cache"` の Cache Components だけは、まだ限定的な対応です。新しい版を流量 0% でデプロイし、その版にリクエストしてキャッシュを温めてから切り替える「キャッシュウォーミング」も入りました。

自分の 3 サイトには Next.js を使っているものが 1 つもないので、Vinext そのものには出番がありません。ただ、同じ週に出た VoidZero の報告（https://blog.cloudflare.com/voidzero-update/ ）のほうは、manabi-map に直接効きます。VoidZero は Vite、Vitest、Rolldown、Oxc を作っているチームで、6 月に Cloudflare に加わりました。manabi-map は Vite で作っているので、Vite 8、Vitest 5、型情報を使う lint を ESLint より速く回す tsgolint あたりが、手元の開発を速くしてくれる候補です。

### Kitesurf と WebMCP。「人間が見る Web」と「エージェントが使う Web」

Kitesurf は、8 月の Agents Week で発表された、AI エージェント専用のブラウザです。Cloudflare Developers の公式投稿（https://x.com/CloudflareDev/status/2085394318005846411 ）では、Chromium はエージェント 1 体ずつに渡すには重すぎる、Kitesurf は Rust で書かれていて CPU とメモリの使用量が 3 分の 1 から 7 分の 1、リクエストごとに起動する、と紹介されていました。

今回の続報（https://blog.cloudflare.com/kitesurf-update/ ）では、Web 標準のテスト（WPT）の通過数が 73 万を超えたこと、ターミナルの中にページを描画するモードが付いたこと、そして WebMCP への対応が加わりました。提供状況はこうです。

> It's available for free while in beta, behind per-account limits.

WebMCP は、サイトの側が「この機能はこの関数で呼べます」とエージェントに公開する仕組みです。エージェントはボタンを探してクリックする代わりに、関数を直接呼べる。Cloudflare では 8 月から開発者プレビューとして、コードもオリジンも変えずにスイッチ 1 つで有効にできる形で提供されています（https://x.com/CloudflareDev/status/2085393979479380035 ）。まつにぃ（@yugen_matuni）さんの「WebMCP 一瞬で作れるのすごいな」という反応（https://x.com/yugen_matuni/status/2085377152216875264 ）が、手触りをよく表していると思います。

自分のサイトで考えると、manabi-map の学校検索は、WebMCP のツールとして出す意味がありそうです。「この住所から通える公立高校」をエージェントが聞きに来たとき、画面を操作させるより、検索の関数を直接呼ばせたほうが、こちらのサーバーにも優しい。後で書く創業者レターの数字を見ると、なおさらそう思います。

### EmDash 1.0。WordPress の後継を名乗る CMS が、安定版になった

EmDash は、4 月 1 日に v0.1.0 として公開された、Astro ベースの CMS です（https://blog.cloudflare.com/emdash-wordpress/ ）。Astro は 1 月に Cloudflare に加わっていて（https://blog.cloudflare.com/astro-joins-cloudflare/ ）、その記事では Astro はオープンソースで MIT ライセンスのまま、と約束しています。

今回の 1.0（https://blog.cloudflare.com/emdash-cms-plugin-registry/ ）の目玉は、プラグインの扱いです。プラグインは隔離された環境で動き、既定ではコンテンツにも秘密情報にもネットワークにも触れません。使う権限を宣言し、管理者が承認してから入れる。記事の表現を借りると、

> feels more like installing a mobile app than a traditional CMS plugin.

プラグインの配布も、AT Protocol のアカウントを使った分散型のレジストリになりました。発行者が署名したリリースの記録を、発行者自身のアカウントに置く形です。

4 月の公開時、X の反応はかなり割れていました。

- Cloudflare の Dane Knecht さんは、TypeScript、サーバーレス、MIT ライセンス、MCP サーバー内蔵、既存の WordPress サイトを数分で取り込める、と発表していました（https://x.com/dok2001/status/2039375273758490691 ）
- Yusuke Wada（@yusukebe）さんは「テクニカルな意味で非常に面白い」として、Dynamic Worker でのプラグイン実行、Astro ベース、パスキー認証などを挙げています（https://x.com/yusukebe/status/2039640468426997816 ）
- WordPress の YouTube で知られる Jamie Marsland さんは、よくできているが「aimed at the wrong problems」、つまり解くべき問題がずれている、と書いていました（https://x.com/pootlepress/status/2039603842946404531 ）
- しょうへい（@showheyohtaki）さんは、v0.1.0 のプレビュー版なので「今すぐ移行する段階ではない」と整理しています（https://x.com/showheyohtaki/status/2039632744507134064 ）
- 実際に使い込んだ黒神（@kokushing）さんは、4 月時点で「まだプロダクション運用はできない」、ただ個人ブログならあり、プラグインを使うには Sandbox のために月 5 ドルかかる、と具体的に書いています（https://x.com/kokushing/status/2043012810268098834 ）

1.0 はこの 4 月の声への回答、という読み方もできます。Cloudflare 自身のブログも EmDash へ移行したと InfoQ が報じていて（https://www.infoq.com/news/2026/09/cloudflare-emdash-migration/ ）、自社で使い始めたのは一つの区切りでしょう。

自分の場合は、入れる理由が見つかりませんでした。記事は Markdown で書いて GitHub に push し、Worker が取ってきて配る形で、もう回っています。Naoki Aoyama（@naa0yama）さんが、CMS の引っ越しや画像の置き換えで苦労したので GitHub と Hugo と Cloudflare で md を書く運用に慣れてしまった、と書いていたのですが（https://x.com/naa0yama/status/2039662388430131704 ）、まさに自分もこっち側です。管理画面が要る人、書き手が複数いる人には、選択肢に入ってくると思います。

### BEACON。自分のサイトが「世界の中で速いのか」を測る物差し

BEACON は、Cloudflare 上の大きなサイト 1 万件から集めた、実際の利用者の表示速度の計測データを、Google BigQuery で公開したものです（https://blog.cloudflare.com/how-fast-is-the-web/ ）。

> billions of real-world performance measurements across 10,000 of the largest websites.

BigQuery の `cf-open-web-performance` プロジェクトにある `rumarchive` データセットで、毎日更新されます。LCP、CLS、INP といった Core Web Vitals を、平均値ではなく分布のまま、国・OS・ブラウザ別に見られます。

記事の中の発見も面白かったです。iOS の WebKit は全体では一番速いのに、46 の国では Blink 系より LCP と INP が 1 割悪い。表示の遅さに効いているのは、画像やフォントのダウンロード時間そのものより、読み込みの発見が遅れることや、描画を止めるリソースのほう。ページ内の移動（ソフトナビゲーション）は、ページ全体の読み込みより 2 倍から 3 倍速く描画される。

自分のサイトの数字は、このデータセットには入っていません。自分の側の数字は、Cloudflare Web Analytics が集めている Core Web Vitals で見られます（https://developers.cloudflare.com/web-analytics/data-metrics/core-web-vitals/ ）。manabi-map の数字を、BEACON の日本のスマホの分布と並べれば、「速いつもり」が世界の中でどのあたりなのか分かるはずです。BigQuery を使うので Google Cloud のアカウントが要り、クエリの課金がどうなるかは記事に書かれていなかったので、試す前に確認します。

### 創業者レター。自動トラフィックが、人間を追い越した

Birthday Week の前日、9 月 27 日に出た創業者レター（https://blog.cloudflare.com/cloudflares-2026-annual-founders-letter/ ）で、Matthew Prince さんと Michelle Zatlyn さんは、自動化されたトラフィックが人間のトラフィックを上回る時期の見込みについて書いています。当初は 2027 年後半と予測していたものが、エージェントと AI のクローラーの伸びで、

> pulled that date forward to May of 2026.

さらに、今の傾向が続けば、5 年で自動化トラフィックは人間の 1,000 倍になる、とも。Prince さんは 6 月の時点で、予想よりずっと早く、ボットがインターネットの歴史で初めて人間のトラフィックを超えた、と X に書いていました（https://x.com/eastdakota/status/2062212701414187452 ）。

1,000 倍、です。

個人サイトの運営者として、これは他人事ではありません。無料枠の上限はリクエスト数で決まっていて、そのリクエストの大半が人間でなくなる、ということですから。実際、本サイトの KV の上限に当たったのも、人間の読者ではなくクローラーの巡回でした。Bot Fight Mode やレート制限を入れておくこと、`_routes.json` のように「そもそも重い処理を通さない」作りにしておくこと、そして WebMCP のように「エージェントには専用の入口を渡す」こと。3 つとも、この数字の前では少し意味が重くなります。

### そのほかの 3 本

- **VoidZero の 4 か月報告**（https://blog.cloudflare.com/voidzero-update/ ）: 加入から 4 か月で 80 を超えるリリース。Oxc の React Compiler、Vitest 5、tsgolint の安定版、Oxfmt、Vite+ 1.0 など。上の Vinext の節で書いたとおり、Vite で作っている人にはこちらのほうが効くかもしれません
- **Rust Workers の Emscripten ターゲット**（https://blog.cloudflare.com/rust-workers-emscripten-target/ ）: `wasm32-unknown-emscripten` の wasm-bindgen 対応を、実験的なプレビューとして出したものです。Tokio ベースの Rust アプリを Workers で動かす道が開きつつあります。cf の説明では Rust の Worker は Wrangler に任されるので、ここは Wrangler 側の話として追うことになりそうです
- **The Cold Start**（https://blog.cloudflare.com/introducing-the-cold-start/ ）: 10 月の Cloudflare Connect で、初期のスタートアップがピッチするイベントです。応募は米国かカナダの拠点が条件でした

## いつ cf に乗り換えるか。待つ条件と、待つ間にやること

最後に、自分の結論をまとめます。

**今は乗り換えません。** cf は open beta で、自分のサイトの構成（esbuild ビルドの Worker、Pages、Direct Upload）は、どれも cf の恩恵がまだ小さい側に入ります。Wrangler はベータ終了後 18 か月保守されるので、慌てる理由もありません。

**再開する条件**は 3 つにしました。

1. cf が beta でない 1.0 以上になる
2. Cloudflare Vite plugin が安定版になる
3. 自分が使う D1 と Pages の操作が、cf で素直に打てる（名前で指定できる、ログを流し見できる）

**待つ間にやること**のほうが、実は大事でした。

- 地図サイトのデプロイを GitHub Actions に載せて、CI の成果物をそのまま上げ、公開後の照合まで自動で回す
- manabi-map のデプロイ経路を 1 本に決めて、「push で本番に出る」と書いたままの手順書を直す
- 3 サイトとも Wrangler の版を `package.json` で固定する
- 本サイトの記事キャッシュの版上げを、手作業のまま忘れない仕組みにする

どれも cf が無くてもできることで、cf が来たときには、自動化した流れの中のコマンドを 1 行差し替えれば済むようになります。

## あわせて読みたい

- [Cloudflare Pages の SPA フォールバックが 308 になる。直しても既存ブラウザには届かない](https://qiita.com/ishizakahiroshi/items/0b4e0b146ec0cbeb00fe)（manabi-map で `_redirects` をやめて middleware に移した経緯です）
- [Cloudflareの無料枠を考える前に、公開データをSQLiteと静的配信へ分ける](https://qiita.com/ishizakahiroshi/items/0ae508ed2ee095761154)（manabi-map のデプロイを今組み替えている理由の、設計側の話です）
- [ダウンロード数が見たいだけだったのに、「勝手に課金」が怖くて無料クラウドを3つ比べた](https://note.com/ishizakahiroshi/n/n7cac773ccb67)（本サイトの統計部分を Cloudflare の無料枠で作ると決めたときの比較です）

## おわりに

「多すぎて訳がわからない」から始めて、4 万字近く書いてしまいました。

分かったことは、案外シンプルでした。棚は 4 つ。個人開発で触るのはその一部。新しい道具は AI のほうを向いていて、自分のサイトの手間を減らすのは、道具より先に「誰がデプロイを起動するか」を決めることだった。

cf が正式版になったら、今日書いた 3 つの条件を 1 つずつ確かめて、この記事の続きを書くつもりです。それまでは、地図サイトの配送を自動にするところから。小さく、1 サイトずつ片づけていきます。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-09-29_cloudflare-map-birthday-week-2026/

※ ヘッダー画像とインフォグラフィックの絵は AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
