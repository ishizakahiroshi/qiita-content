---
title: >-
  Cloudflare Pages から Workers へ移したら管理画面だけ
  404。静的ファイルの照合では見えない秘密の持ち越し漏れを、配備の手前で止める
tags:
  - cloudflare
  - CloudflareWorkers
  - Wrangler
  - Supabase
  - 個人開発
private: false
updated_at: '2026-09-30T07:33:28+09:00'
id: 3744dd94411fc10c1dde
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![照合は全部通った、なのに管理画面だけ 404](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-29_workers-migration-missing-secrets/01_2026-09-29_workers-missing-secrets_hero.png)

Cloudflare Pages で動いていたサイトを、Workers の static assets へ移しました。1 万件を超える静的ファイルを 1 件ずつ突き合わせて、全部一致。ヘッダも同じ。

それなのに、管理者がログインすると管理画面から追い出される。Pages Functions が読んでいた秘密（Supabase の URL と鍵）を、Worker へ持ち越していなかったのが原因でした。

この記事は、Pages から Workers へ移すときに秘密の持ち越し漏れをどう見つけたかと、次から配備の手前で止めるために足した検査の記録です。

題材は、自作の学校検索サービス「まなびマップ」です。通える高校を地図で見ながら、親子で決める。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=manabi-map
- リポジトリ: https://github.com/ishizakahiroshi/manabi-map

Web版は https://manabi-map.app から開けます。地点を検索し、周辺の高校を地図で探すところから試せます。Starをいただけると励みになります。

この話には前があります。前回は Cloudflare のサービスを棚に分けて、個人サイト 3 つが実際にどの部品で動いているかを設定ファイルから棚卸ししました。今回はそのうち 1 つを、Pages から Workers へ実際に移したときの話です。

前回の記事: [Cloudflareのサービス全体地図2026。Workers・Pages・D1から新CLI「cf」まで、個人サイト3つの実構成で読み解く](https://qiita.com/ishizakahiroshi/items/5b83591f7e6e90ca53b6)

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-29_workers-migration-missing-secrets/02_2026-09-29_workers-missing-secrets_infographic.png)

## 先に結論。移す前に「コードが読む env 名」を数える

- 静的ファイルの照合（中身とヘッダの一致）は、Worker に何の binding が付いているかを見ていません。秘密が無くても全部合格します
- 移行元の設定画面を写すのではなく、**Functions のコードが読んでいる `env.XXX` の名前**を先に数えます
- 数えた名前は「登録する秘密」か「わざと設定しない（理由つき）」のどちらかとしてファイルに宣言し、コードとずれたらテストで落とします
- 本番化の直前に `wrangler versions view <version-id> --json` で binding を読み、宣言と完全一致しなければ止めます

## 1 万件が全部一致した夜

移行作業は、範囲を決めて AI エージェントに夜のうちに任せていました。

手順はこうです。新しい Worker を作り、`wrangler versions upload` で Preview 版として送る。Preview の URL で全ファイルを取り直して、候補と bytes 単位で突き合わせる。一致したら、同じ version を `wrangler versions deploy` で 100% にする。最後に独自ドメインを付けて、もう一度全件。

Preview、workers.dev、独自ドメインの 3 か所で、全部合格。報告は全部緑でした。

## 「ログインできたけど、管理画面が見られない」

その夜のうちに、自分で管理者としてログインしてみて気づきました。ログインはできる。でも管理画面を開くと、トップへ戻される。

フロントの作りを確認すると、管理者かどうかは専用の API に聞いていて、**200 が返ったときだけ**管理者として画面を出します。それ以外はトップへリダイレクト。つまり「戻される」は、その API が 200 以外を返しているということです。

推測で直す前に、Worker のログを流しながらもう一度開いてみました。

```bash
npx wrangler tail <worker-name> --format json
```

管理者判定の API が、404 を返していました。

## 404 を返していたのは、管理者判定の API だった

その API は、失敗をすべて同じ 404 で返す作りです。管理者でないときも、ログインの確認に失敗したときも、Supabase の URL か鍵が環境に無いときも、全部 404。

```ts
// 管理者判定（要点だけ抜粋）
if (!token || !env.SUPABASE_URL || !env.SUPABASE_ANON_KEY) return notFound()
```

Worker の binding を確かめると、付いていたのは静的ファイル用の `ASSETS` だけ。秘密は 1 つもありませんでした。

「管理者ではない」と「設定が無い」が、外から見ると同じ 404。ここが盲点でした。

## 移行元の Pages にも、環境変数は 1 つも無かった

なぜ誰も気づかなかったのか。移行元を見に行って、納得しました。

Pages 側の高校版のプロジェクトにも、環境変数が 1 つも無かったんです。Worker は移行元の Pages の設定に合わせて作ったので、秘密なしで生まれました。移行元の時点で、もう欠けていたわけです。

そして、1 万件の照合はこれを拾えません。照合が見ているのは、配ったファイルの中身とヘッダです。Worker が実行時に何を読めるかは、どこにも写っていません。

![荷物は全部そろったのに、裏口の鍵だけ旧居に置いてきた](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-29_workers-migration-missing-secrets/03_2026-09-29_workers-missing-secrets_illustration-left-key.png)

引っ越しで、段ボールは 1 箱残らず新居に届いて検品も済んだ。なのに、事務所の裏口の鍵だけが旧居に掛かったまま。今回の状態はそれに近いです。

## 秘密を足したのに、最初の 1 回はまた 404

秘密は Worker 側にだけ置くことにしました。リポジトリには値を書きません。

`wrangler versions secret bulk` を使うと、今の version のコードはそのままに、秘密だけを足した新しい version ができます。この時点では本番に出ないので、Preview の URL でもう一度全件を照合してから本番化できます（コマンドの一覧は https://developers.cloudflare.com/workers/wrangler/commands/workers/ ）。

```bash
# 秘密の値を書いた一時ファイルは、読ませたらすぐ消す
npx wrangler versions secret bulk secrets.json --name <worker-name>
# できた version を Preview で照合してから本番化
npx wrangler versions deploy <version-id>@100% --name <worker-name>
```

これで直った、と思って開くと、ログの 1 行目はまた 404。10 秒後の 2 行目が 200 でした。

直ったのか、直っていないのか。ログの状態コードだけでは分かりません。

Supabase のログで、2 つの要求がどのアカウントから来たかを突き合わせると、答えは単純でした。**1 回目は Google でログインしたアカウント、2 回目は LINE でログインしたアカウント**。同じ人でも Supabase 上は別のユーザーで、管理者として登録してあったのは片方だけ。1 回目の 404 は、正しい 404 だったわけです。

ちなみに、Supabase の Management API で古いログ取得口を叩いたら 410 が返ってきました。今は `/v1/projects/{ref}/analytics/endpoints/logs` に ClickHouse の SQL を送る形で、ログの種類は `source` 列で絞ります（https://supabase.com/docs/reference/api/v1-get-project-logs ）。短い間隔で続けて呼ぶと、今度はレート制限で弾かれます。地味に足止めされました。

## 配備の手前に、関所を 2 つ足した

同じ穴を次に踏まないように、配備に使っている手元のスクリプトへ検査を足しました。

### 1. コードが読む env 名を、ファイルに宣言してテストで固定する

Worker ごとに、登録する秘密と、わざと設定しない名前（理由つき）を JSON に書きます。値は書きません。

```json
{
  "secrets": ["SUPABASE_ANON_KEY", "SUPABASE_SERVICE_ROLE_KEY", "SUPABASE_URL"],
  "unset": {
    "MAINTENANCE_MODE": "未設定で通常運転。\"1\" のときだけサイト全体を止める"
  }
}
```

テストは Functions のソースから `env.XXX` を拾い、宣言と両方向で突き合わせます。

```js
// Functions のソースから、読んでいる env 名を集める（binding の ASSETS は除く）
for (const m of source.matchAll(/\benv\s*(?:\?\.|\.)\s*([A-Z][A-Z0-9_]*)\b/g)) read.add(m[1])

// コードが読むのに、宣言していない名前
assert.deepEqual([...read].filter((n) => !declared.has(n)), [])
// 宣言しているのに、もう読まれていない名前
assert.deepEqual([...declared].filter((n) => !read.has(n)), [])
```

宣言から秘密 3 つを消すと落ち、戻すと通ることを確かめました。新しい秘密をコードに足した人は、宣言を書かないとテストが通りません。

### 2. 本番化の直前に、version の binding を読む

照合では binding が見えないので、本番化の直前に読みます。

```js
const view = JSON.parse((await run(['versions', 'view', versionId, '--name', worker, '--json'])).stdout)
const actual = view.resources.bindings.map((b) => `${b.name}:${b.type}`).sort()
const expected = ['ASSETS:assets', ...secrets.map((n) => `${n}:secret_text`)].sort()
if (JSON.stringify(actual) !== JSON.stringify(expected)) throw new Error('binding が宣言と一致しない')
```

足りない名前と余分な名前は出しますが、値は出しません。今回と同じ「ASSETS だけの version」を本番化しようとすると、ここで止まります。

![静的ファイルの照合が見ていた範囲と、足した 2 つの関所](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-29_workers-migration-missing-secrets/04_2026-09-29_workers-missing-secrets_fig1.png)

図にすると、ファイルの中身とヘッダは照合で守れていて、抜けていたのは Worker に付く binding だけでした。宣言とテストで「何を持つべきか」を固定し、本番化の直前に「実際に持っているか」を確かめる形にしています。

あわせて、新しい Worker でも最初の本番化より前に秘密を足せるよう、秘密を足す手順の基準を「本番化済みの version」から「アップロード済みの version」へ広げました。順番は upload、秘密、Preview で照合、本番化です。

書いている途中で、`wrangler versions upload` に秘密を同時に渡す `--secrets-file` があると公式ドキュメントで知りました（https://developers.cloudflare.com/workers/configuration/secrets/ ）。手順を次に見直すときの候補にしています。

## 学んだこと

- **中身が全部一致しても、動く保証ではない。** 照合が何を見ていないかを、一度書き出しておく
- **移行元の設定は正本ではない。** コードが読む名前のほうを数える。移行元が最初から欠けていることもある
- **失敗を全部同じ 404 で返す作りは、設定漏れも見分けられない。** 本番化の前に、binding を機械で読む
- **「入れない」の報告は、どのアカウントから来たかを分けてから直す。** ログイン方法ごとに別ユーザーになる構成では特に

## まなびマップはこんなときに刺さります

- 通える範囲の高校を、地図で見比べたい人
- 文化祭や説明会で見聞きしたことを、学校ごとに家族で残したい人
- 偏差値の数字が、どこまで根拠を確認できたものかを気にする人
- 設定する地点について、保存される個人データを最小限にしたい人

いずれかに心当たりがあれば、ブラウザで https://manabi-map.app を開くだけで試せます。インストールや設定は要りません。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=manabi-map
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/manabi-map

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## あわせて読みたい

- [Cloudflare Pages の SPA フォールバックが 308 になる。直し方と、直しても既存ブラウザに届かない理由](https://qiita.com/ishizakahiroshi/items/0b4e0b146ec0cbeb00fe)（検収に使った経路だけが最後まで問題を見せなかった話。照合の守備範囲という意味で同じ穴です）
- [PostgreSQL の migration が「全部成功」したのに本番が壊れる。plpgsql の遅延解決と、カタログを見る検証の限界](https://qiita.com/ishizakahiroshi/items/257b0fc493007e64db4b)（全部緑なのに壊れる、の DB 版。実際に呼んで確かめる検証へ切り替えました）
- [個人開発の Web 進路サービスに Supabase + LINE + Cloudflare Pages を組み合わせて起きた落とし穴の記録](https://zenn.dev/ishizakahiroshi/articles/20260706-manabi-map-supabase-line-cloudflare)（同じサービスの認証まわりの落とし穴。LINE ログインの構成はこちら）

## 次に移すときは、まず数えるところから

全部緑の報告は、その照合が見ている範囲の中だけの緑でした。見ていない範囲を先に書き出しておけば、今回の穴は移す前に見えていたと思います。

次に何かを移すときは、移行元の画面を開く前に、コードの `env.` を数える。宣言を書いて、テストを通してから移す。小さく、これを手順の最初に置いていきます。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-09-29_workers-migration-missing-secrets/

---

※ ヘッダー画像とインフォグラフィックの絵は AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
