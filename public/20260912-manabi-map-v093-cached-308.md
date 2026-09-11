---
title: "Cloudflare Pages の SPA フォールバックが 308 になる。直し方と、直しても既存ブラウザに届かない理由"
tags:
  - CloudflarePages
  - SPA
  - CSP
  - 個人開発
  - リダイレクト
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/01_2026-09-12_cached-308_hero.png)

リリースから 1 か月が経ったので、アクセス解析でも眺めようと思っただけでした。Search Console と Cloudflare のダッシュボードを開いて、数字を台帳に書き写して、それで終わるはずの夜でした。

終わりませんでした。ついでに `curl` で叩いた `/map` が、308 を返してきたからです。

## manabi-map という進路サービスを作っています

自作で manabi-map という Web サービスを作っています。通える高校を地図で見ながら、親子で決める。そういうサービスです。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=manabi-map
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/manabi-map

全国 47 都道府県・5,096 校を収録していて、AGPL-3.0 の OSS です。地点を検索すると通える範囲の高校が地図に並び、学校ごとに家族でメモを残せます。

インストールは要りません。ブラウザで開くだけです。

https://manabi-map.app

スマートフォンでもデスクトップでも同じ URL で、アカウントを作らなくても地図と検索は使えます。お気に入りと家族メモだけログインが要ります。

技術構成はこの記事の前提になるので先に書いておきます。React + TypeScript + Vite で作った SPA を、ビルド時に SSR でプリレンダーして静的 HTML を吐き、Cloudflare Pages で配信しています。データは Supabase です。学校データはビルド時に JSON へ焼き込んでいるので、通常の閲覧では DB を触りません。

### 前回の記事

このサービスの記事は続きものになっていて、前回は保存する位置情報を減らした話を書きました。

- 学校の一覧に住所を出しました。そのかわり、あなたの住所は保存しなくしました: https://qiita.com/ishizakahiroshi/items/3b457ffedd146b8bb496

今回はその次のリリース、v0.9.3 の話です。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/02_2026-09-12_cached-308_infographic.png)

## `/map` を直接開くと、トップページが出る

きっかけは本当にただの確認でした。アクセス解析を見るついでに、主要な URL のステータスコードを並べて眺めようとしただけです。

```bash
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" https://manabi-map.app/map
```

返ってきたのは `308 https://manabi-map.app/` でした。

308 は恒久的なリダイレクトです。つまりサーバーは「`/map` というページは今後も無いので、`/` を見てください」と答えていました。地図のページを開こうとした人が、トップページに着地していたということです。

ブラウザで開くと、確かにトップが出ます。アドレスバーからは `/map` が消えています。

これは 1 か月以上、そのままでした。

## `_redirects` に書いた 8 行が、1 行も効いていなかった

Cloudflare Pages で SPA のフォールバックを書くときの定番は `_redirects` ファイルです（仕様: https://developers.cloudflare.com/pages/configuration/redirects/ ）。

うちの `_redirects` にはこう書いてありました。

```text
/map        /index.html   200
/search     /index.html   200
/favorites  /index.html   200
/compare    /index.html   200
/mypage     /index.html   200
/auth/*     /index.html   200
/family/*   /index.html   200
/dashboard  /index.html   200
```

`/* /index.html 200` のような全部入りのワイルドカードにしていないのは意図的です。全部入りにすると存在しない URL まで 200 と空のシェルで返してしまい、Search Console が「ソフト 404」として扱います。以前それでインデックスの判断がノイズだらけになったので、SPA のルートだけを列挙する形に変えていました。

その列挙が、1 行も効いていませんでした。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/03_2026-09-12_cached-308_fig1.png)

上の図のとおり、8 行のうち静的な 6 行はすべて 308 でトップへ、ワイルドカードの 2 行は 404 でした。「200 でシェルを返す」と書いたつもりの設定が、「トップへ飛ばす」と「そんなページは無い」の 2 種類に化けていたことになります。

実測は次のようになりました。

| URL | 応答 | Location |
|---|---|---|
| `/map` | 308 | `/` |
| `/map/` | 404 | 該当なし |
| `/map?school=1` | 308 | `/?school=1` |
| `/search` | 308 | `/` |
| `/favorites` | 308 | `/` |
| `/compare` | 308 | `/` |
| `/mypage` | 308 | `/` |
| `/dashboard` | 308 | `/` |
| `/auth/callback` | 404 | 該当なし |
| `/family/join` | 404 | 該当なし |
| `/schools/` | 200 | 該当なし |
| `/pref/fukuoka/` | 200 | 該当なし |
| `/nonexistent-xyz` | 404 | 該当なし |

静的にプリレンダーしているページ（`/schools/` や県別ページ）は無傷です。壊れていたのは「JavaScript が描く画面」だけでした。

ここで一度、手が止まりました。じゃあ何で今まで誰も困らなかったんだ、と。

## なぜ「200」と書いたのに「308」が返るのか

答えは、右側に書いた `/index.html` の方でした。

Cloudflare Pages は静的ホスティングなので、`/index.html` というパスを `/` に正規化します。ディレクトリのインデックスを正準の URL 1 本に寄せる、よくある挙動です。ここまでは普通の話です。

問題は、この正規化が **rewrite の行き先にも適用される**ことでした。`/map` へのリクエストを `/index.html` の中身で返すつもりで書いた行が、「`/index.html` は `/` に正規化されるので、クライアントに 308 を返して `/` へ行かせる」という動作になっていました。

つまり `_redirects` の 3 列目に書いた `200` は、この組み合わせでは効きません。左の `/map` は正しく拾われているのに、右の `/index.html` が自分を `/` へ送り返してしまう。

ワイルドカードの `/auth/*` と `/family/*` が 404 だったのは別の理由で、こちらはルールとして拾われずに素通りし、Pages が `dist/404.html` を返していました。同じファイルに書いた 8 行が、書き方 2 種類で別々に壊れていたことになります。

この「`/index.html` は `/` に正規化される」は、後でもう一度出てきます。覚えておいてください。

## 誰が、どう困っていたのか

ステータスコードの話だけだと「見た目は動いているし、まあいいか」になりがちなので、実害を書き出しました。

- 検索結果や SNS から `/map` を直接開いた人が、地図ではなくトップに着地する
- 学校ページの「地図で見る」を新しいタブで開くと、地図ではなくトップが出る
- `/map?school=1` のような共有リンクは `/?school=1` に化けるので、どの学校の話だったかが消える
- OAuth の戻り先 `/auth/callback` が 404
- 家族招待リンクの着地先 `/family/join` が 404

最後の 2 つを見つけたときは、正直かなり焦りました。ログインも家族招待も 1 か月以上壊れていたのでは、という顔になります。

実際には壊れていませんでした。Pages は 404 のときに `dist/404.html` を返しますが、うちの `404.html` は SPA のシェルそのものなので、**HTTP ステータスが 404 のまま JavaScript が起動して、正しい画面を描いていた**のです。ブラウザで `/family/join` を開くと、招待ページがちゃんと出ます。無効なトークンなら「参加できませんでした」と正しく表示されます。

見た目は動く。ステータスコードだけが嘘をついている。いちばん気づきにくい壊れ方でした。

とはいえ、これはこれで良くありません。404 を返している URL をクローラーは拾いませんし、`/map` の 308 は往復を 1 回増やします。何より、開こうとしたルートが失われるのが致命的です。

## 1 か月気づかなかった理由を、数字で見る

「1 か月以上そのまま」と書きましたが、なぜ誰も言ってこなかったのかは、同じ夜に見ていたアクセス解析でだいたい説明がつきました。

Search Console 側はこうでした。

- インデックス登録済み 5,710 から 6,410（sitemap の 6,419 に対して 99.9%）
- 「検出 - インデックス未登録」893 から 103
- 直近 28 日でクリック 181 から 250、表示 4.94 万から 7.25 万、平均掲載順位 21.3 から 19.7

伸びているのは県別ページと学校ページです。どちらも**静的にプリレンダーしている側**なので、今回壊れていた SPA ルートとは無関係でした。検索から来る人は `/map` を直接踏みません。

Cloudflare 側はもっと分かりやすい数字でした。ゾーンの 30 日間のリクエストが 306.65k、ユニークビジターが 22.02k。それに対して beacon 計測（ボット除外）の実ユーザーは 790 visits。**生ログの 3.6% しか人間ではない**ということです。

この手の数字は、見るたびに少ししょんぼりします。が、今回に限っては役に立ちました。実ユーザーが少なく、その人たちもトップか検索結果からの学校ページ経由で入ってくる。`/map` を直接叩くのは、開発者である自分がアプリ内リンクではなく URL を手で打ったときくらいです。そしてアプリ内リンクはクライアント側のルーティングなので、壊れていても素通りできてしまう。

**利用者が少ないうちは、壊れたことが症状として返ってこない。** だから定期的に自分で叩いて確かめるしかない、という結論になりました。1 か月放置したのは運が悪かったからではなく、確かめる手順を持っていなかったからです。

## Pages Functions で 200 を返す

直し方は 2 つ考えました。

1. `_redirects` の行き先を `/index.html` ではない書き方に変える
2. 列挙そのものを Pages Functions（middleware）へ移して、シェルの本文を 200 で返す

2 を選びました。理由は 3 つあります。

- `_redirects` は宣言だけなので、どう解釈されるかを自分で確かめられない。今回のように「書いたつもり」と「返る応答」がずれても、テストが書けない
- middleware なら `isSpaRoute()` を関数として切り出せて、ユニットテストでルート表を固定できる
- うちの Pages プロジェクトにはもともとメンテナンスモード用の middleware があり、SPA フォールバックもそこに寄せた方が、判定の順番（メンテ中は `/map` も 503）を明示的に書ける

書いたのはこれだけです。

```ts
/** index.html の本文を 200 で返すルート(末尾スラッシュは正規化して比較) */
const SPA_ROUTES = new Set([
  "/map",
  "/search",
  "/favorites",
  "/compare",
  "/mypage",
  "/dashboard",
  "/auth/callback", // OAuth の戻り先
  "/family/join",   // 家族招待リンクの着地先
]);

// index.html は `/` として取得する。`/index.html` を直接 ASSETS から取ると
// Pages のアセット正規化で 308(`/` へ) が返り、本文が得られない。
const SPA_SHELL_PATH = "/";

export const isSpaRoute = (pathname: string): boolean =>
  SPA_ROUTES.has(pathname.replace(/\/+$/, "") || "/");
```

本体はこうです。

```ts
// GET / HEAD 以外(POST 等)を HTML 200 にすり替えない。
const isReadRequest = request.method === "GET" || request.method === "HEAD";
if (isReadRequest && isSpaRoute(pathname)) {
  const shell = await env.ASSETS.fetch(new URL(SPA_SHELL_PATH, request.url));
  // シェルが取れないときは従来どおりの応答へ落とす(勝手に 200 を作らない)。
  if (!shell.ok) {
    return next();
  }
  // _headers 由来のヘッダ(CSP 等)を落とさないよう、本文と一緒にそのまま引き継ぐ。
  const headers = new Headers(shell.headers);
  headers.set("content-type", "text/html; charset=utf-8");
  return new Response(shell.body, { status: 200, headers });
}

return next();
```

`env.ASSETS.fetch()` は Pages Functions からビルド済みの静的アセットを取るための口です（入門: https://developers.cloudflare.com/pages/functions/get-started/ ）。

コメントに書いたとおり、ここでも `/index.html` の正規化に一度ぶつかりました。middleware の中から `env.ASSETS.fetch("/index.html")` すると、やはり 308 が返ってきて本文が取れません。`/` を取りに行く形にして解決しています。**同じ仕様に、外側（`_redirects`）と内側（Functions）の 2 回踏まれた**ことになります。

細かいところでは次の 3 つを守りました。

- **GET と HEAD 以外は触らない**。POST を HTML 200 ですり替えると、API の失敗が静かに成功に化ける
- **シェルが取れなければ `next()` に落とす**。勝手に 200 を作らない
- **列挙外の URL は素通し**。`/nonexistent-xyz` が 404 のままであることは、直したあとに必ず確認する

最後の 1 つは、以前「全部入りのワイルドカード」をやめてソフト 404 を解消した成果を、今回の修正で巻き戻さないための防波堤です。テストも 5 本から 14 本に増やして、`/map/deeper` や `/auth/` のような「SPA ルートの子」「ワイルドカードの親」が 200 にならないことを固定しました。

`_redirects` からは 8 行を消して、コメントだけ残しました。

```text
# SPA フォールバックのルート列挙は、このファイルではなく functions/_middleware.ts が正典。
# ここに SPA fallback 行を書き足さないこと（二重管理になり、308 正規化の罠も再発する）。
```

同じ設定が 2 か所にあると、次に触る人（半年後の自分を含む）がどちらを直せばいいか分からなくなります。消した場所に理由を置いておくのは、コードより先に読まれる場所を 1 つ作るということだと思っています。

## ルート表を、テストで固定する

`_redirects` から middleware へ移した理由のうち、実際に効いたのは「テストが書けること」でした。

設定ファイルは宣言なので、書いた内容が期待どおり解釈されるかを手元で確かめられません。今回はまさにそこで転んでいます。関数にしてしまえば、ルート表は普通のユニットテストで固定できます。

`isSpaRoute` を export しているのはそのためです。テストはこういう形で書きました。

```ts
test('isSpaRoute matches the App.tsx routes and nothing else', () => {
  for (const pathname of SPA_ROUTE_PATHS) {
    assert.equal(isSpaRoute(pathname), true, pathname)
    assert.equal(isSpaRoute(`${pathname}/`), true, `${pathname}/`)   // 末尾スラッシュも同じ扱い
  }
  for (const pathname of [...PRERENDERED_PATHS, '/nonexistent-xyz', '/auth', '/family']) {
    assert.equal(isSpaRoute(pathname), false, pathname)
  }
})
```

`PRERENDERED_PATHS` には `/`、`/schools/`、`/pref/fukuoka/`、`/legal/terms/`、`/press/` のような**静的 HTML を出しているパス**を並べてあります。ここを横取りしたら、せっかくのプリレンダー結果を捨てて空のシェルを返すことになるので、いちばん壊してはいけない側です。

そして下半分のほうが大事です。**直した結果、前に直したものが壊れていないか**を見ています。

以前、存在しない URL まで 200 で返していたのをやめて、Search Console のソフト 404 を解消しました。今回の修正は「SPA ルートだけ 200 にする」なので、油断すると SPA ルートの子や、ワイルドカードの親まで 200 にしてしまい、その成果を巻き戻します。middleware を実際に通す側のテストでも、そこを名指しで押さえました。

```ts
test('unknown URLs stay 404 instead of getting the SPA shell', async () => {
  for (const pathname of [
    '/nonexistent-xyz',
    '/map/extra',           // SPA ルートの子
    '/auth/nonexistent',
    '/family/nonexistent',
    '/mypage-typo',         // 1 文字違い
  ]) {
    // ... 404 のまま素通しされ、シェルを取りに行っていないこと
  }
})
```

最後の `assert` で「アセットを取りに行っていない」ことまで見ているのがポイントです。ステータスが 404 でも、内部でシェルを取ってしまっていたら意図とは違う実装になっています。

テストは 5 本から 14 本に増えました。増えたぶんはほぼ全部「200 にしてはいけないもの」と「メンテナンスモードが SPA フォールバックより先に勝つこと」です。

正しく動くことのテストより、**やりすぎないことのテスト**のほうを厚くする。今回みたいに「ワイルドカードで全部拾えばいいや」に流れやすい変更では、ここが効くと思っています。

## セキュリティヘッダは引き継がれるのか。コードを読んでも分からなかった

この修正でいちばん怖かったのはここです。

Cloudflare Pages では `_headers` ファイルでレスポンスヘッダを足せます。うちはそこに `X-Frame-Options` や `Content-Security-Policy-Report-Only` を書いています。

問題は、`env.ASSETS.fetch()` で取った応答に、この `_headers` のルールが適用されているのかどうかです。適用されていなければ、`/map` だけセキュリティヘッダが全部落ちた状態で配信されることになります。iframe への埋め込みも CSP も効かない穴が 1 つ空く。

ドキュメントを読んでも、コードを読んでも、確定できませんでした。「たぶん適用される」で本番に出す類の話ではありません。

なので Preview に出して、実際のヘッダを並べました。

```bash
curl -sI https://<preview>.pages.dev/map \
  | grep -iE "x-frame-options|x-content-type|referrer-policy|permissions-policy|content-security-policy"
```

結果、`/` と `/map` と `/auth/callback` の 3 つで **完全に同じヘッダ集合**が返りました。

```text
x-frame-options: DENY
x-content-type-options: nosniff
referrer-policy: strict-origin-when-cross-origin
permissions-policy: camera=(), microphone=(), payment=(), usb=()
content-security-policy-report-only: default-src 'self'; ...
cache-control: public, max-age=0, must-revalidate
```

`_headers` は「配信するアセットに対して」適用されるので、Function が `ASSETS` から取った応答にも乗ってくる、という理解でよさそうです。ただしこれは実測から言えることで、仕様として保証されているかまでは確かめていません。次に Pages の挙動が変わったときに気づけるよう、この確認は検収項目として手順書に残しました。

分からないものは、分かる形にしてから出す。当たり前のことなのに、締め切りが近いとここを飛ばしたくなります。

## ついでに見つかった、ページを開くたびに 1 件出ていた CSP 違反

ヘッダを並べているときに、CSP の Report-Only 違反が毎回 1 件飛んでいるのに気づきました。

原因は `connect-src` でした。

```text
connect-src 'self' https://*.supabase.co https://nominatim.openstreetmap.org ...
```

Supabase の HTTPS は許可してあります。ところがメンテナンス状態を画面へ即時反映するために Realtime を使っていて、こちらは WebSocket、つまり `wss://` です。

CSP のソース表現はスキームを含めて照合するので、`https://*.supabase.co` を許可しても `wss://<project>.supabase.co` は別物として扱われます（`connect-src` の仕様: https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Content-Security-Policy/connect-src ）。同じホストなのに通らない。ここは頭では分かっていても、設定を書くときに落とします。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/06_2026-09-12_cached-308_fig3.png)

上の図のとおり、許可リストに書いた `https://` は `wss://` を含みません。ホスト名が同じでも、スキームが違えば CSP は別の宛先として扱います。

Report-Only なので画面は壊れていませんでした。だから 1 か月以上、誰にも気づかれずレポートだけが積もっていたわけです。

ただし、これを enforce（実際にブロックする設定）へ上げていたら、**メンテナンス状態の即時反映が静かに止まっていました**。新しく開いたタブは REST で初期値を読むので正しく判定されます。壊れるのは「すでに開いている画面」だけで、しかもリロードすれば直る。バグ報告が上がってこない種類の故障です。

`connect-src` に `wss://*.supabase.co` を足して解決しました。粒度は既存の `https://*.supabase.co` に揃えています。

## 「違反 0 件」を、どうやって「0 件だ」と言い切るか

ここが今回いちばん勉強になったところかもしれません。

`wss://` を足したあと、当然「違反が止まったか」を確かめたくなります。ヘッダに文字列が入ったことは `curl` で見えますが、違反が実際に止まったかはブラウザでしか分かりません。

最初はコンソールを読もうとしました。Report-Only の違反はコンソールにも出るはずです。ところが、使っているブラウザ操作ツールのコンソール読み取りが、このタブで 1 件も返してきませんでした。リロードしても同じです。

ここで「0 件でした」と書くのは危険です。**違反が無いのか、読み取れていないだけなのかが区別できていない**からです。

なので、イベントを直接購読して、対照実験を同時に走らせました。

```js
window.__v = [];
document.addEventListener('securitypolicyviolation', e =>
  window.__v.push({ d: e.violatedDirective, u: e.blockedURI, dispo: e.disposition }));

new WebSocket('wss://<project>.supabase.co/realtime/v1/websocket'); // 対象
new WebSocket('wss://echo.websocket.invalidtest/');                 // 対照（許可していない宛先）
```

`securitypolicyviolation` イベントは、CSP 違反が起きたときにドキュメントへ飛びます（仕様: https://developer.mozilla.org/ja/docs/Web/API/Document/securitypolicyviolation_event ）。Report-Only の場合は `disposition` が `report` になります。

結果は違反 1 件。その 1 件は対照側の `wss://echo.websocket.invalidtest/` でした。対象の Supabase は 1 件も出ていません。

```json
{ "count": 1,
  "violations": [
    { "d": "connect-src", "dispo": "report", "u": "wss://echo.websocket.invalidtest/" }
  ] }
```

これで初めて「0 件」と言えます。対照が 1 件出ているので、リスナーが動いていないわけではないと分かるからです。

**「何も起きなかった」を成果として報告するときは、同じ経路で 1 件は起こしてみせる。** 検査が動いていないことを、検査に合格したことと取り違えないための、いちばん安い保険だと思います。

これは前に別のところでも踏んでいて、そのときは「検査スクリプトが 1 度も実行されていないのに合格と表示されていた」という間抜けな話でした。形は違っても根っこは同じです。

## 一瞬しか見えない地図タイルを、毎回 20 枚

ここからは同じリリースに入れたもう 1 つの変更です。毛色が違うので分けて書きます。

前提として、このサービスの利用者は中高生と保護者です。とくに子ども側はモバイル回線の残量が少ない。なので「見た目のために通信量を使わない」を製品ルールに明記しています。

この 1 か月で 2 つ減らしました。

- **Web フォントを全廃**（v0.9.1）。日本語フォントは `unicode-range` で細かく分割されるので（仕様: https://developer.mozilla.org/ja/docs/Web/CSS/%40font-face/unicode-range ）、県ページを 1 枚開くだけで実測およそ 2.8MB を取りに行っていました。サイト本体（HTML と JS と CSS の合計およそ 300KB）の 9 倍です。読み込み完了時に文字幅が変わってレイアウトがずれる問題（CLS）の主因でもありました
- **地図が読むデータを分割**（v0.9.2）。全国 5,096 校ぶんを入試実績や出典まで含めて一度に読んでいたのを、地図と一覧に必要な分だけに絞りました。3,790,673 バイトから 812,905 バイトへ、およそ 78.6% 減です

その結果、`/map` の通信量に占める地図タイルの割合が 12.0% から 44.5% に上がりました。分母が減ったので、残ったものが目立ってきた形です。

そこで測ったら、こうなっていました。

地図を開くと、まず全国表示（ズーム 5）を描いて、それから設定した地点の通学圏へ寄せていました。つまり **一瞬しか見えない全国のタイルを、毎回 20 枚ほど取っていた**ことになります。ローカルの実測でおよそ 263KB。ユーザーの目には 1 秒も映りません。

タイルは OpenStreetMap の公式タイルを使っているので、これは自分の回線だけの話ではありません。利用ポリシーにも「不要な一括取得をしないこと」が明記されています（https://operations.osmfoundation.org/policies/tiles/ ）。見られずに捨てるぶんは、先方の帯域も使っています。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/07_2026-09-12_cached-308_fig4.png)

上の図のとおり、全国表示ぶんのタイルがまるごと消えます。通学圏のタイルは変わりません。地図に出る学校も 1 校も減っていません。

直し方はごく単純で、前回その端末で通学圏に寄せたときのズーム段階を覚えておき、最初からその範囲で地図を初期化するようにしました。

```ts
const initialView = resolveInitialMapView(
  initialSharedSchoolIdRef.current ? null : loadLocalHome(),
  loadStoredHomeZoom(),
)
const map = L.map(node, { zoomControl: false })
  .setView([initialView.lat, initialView.lng], initialView.zoom)
```

気をつけたのは次の点です。

- **保存値の検証は純粋関数に切り出す**。壊れた JSON、範囲外のズーム、座標として成立しない値は、全部「全国表示」へ落とす。`localStorage` を読む処理はすべて `try/catch` で囲む（プライベートウィンドウで例外になる環境がある）
- **共有 URL で開いたときは従来どおり全国表示から始める**。`?school=<id>` 付きで開いた人には、その学校の位置が優先されるべきなので
- **設定地点をまだ保存していない人の表示は変えない**。覚えていないズームを使うことはできないので、初回訪問は今までどおり

結果はこうなりました。同じウィンドウサイズで、ズームの記憶だけを変えて測っています。

| 条件 | タイル総数 | ズーム 5（全国） | 通学圏 |
|---|---|---|---|
| 記憶なし（初回訪問・変更前と同じ経路） | 47 | 20 | 27 |
| 記憶あり（2 回目以降） | 27 | 0 | 27 |

本番でも測り直して、38 枚から 18 枚（全国ぶん 20 枚が消える）でした。総数が違うのは画面サイズの差で、減る枚数は同じです。

得をするのは 2 回目以降に地図を開く人だけです。ふだん使う人ほど効く、という形になりました。

## 減らしたぶんが静かに戻らないよう、CI で止める

通信量の話で厄介なのは、減らした直後がいちばん軽くて、そこからじわじわ戻ることです。機能を 1 つ足すたびに数十 KB ずつ増えて、半年後に測ったら元に戻っている。これは前にも見ました。

なので v0.9.2 のときに、ビルド成果物の大きさを CI で検査するようにしました。gzip をかけた後のサイズに上限を置いて、超えたら公開前に落とします。

```js
const FILE_BUDGETS = [
  { key: 'app-js',       budget: 290_000 },  // 実測 253,290
  { key: 'map-js',       budget:  75_000 },  // 実測  57,733
  { key: 'map-payload',  budget: 1_000_000 },// 実測 812,905
  // ...
]

const COMPOSITE_BUDGETS = [
  { key: 'pref-page-initial', budget:   380_000 }, // 実測   317,283
  { key: 'map-initial',       budget: 1_300_000 }, // 実測 1,138,018
]
```

個別ファイルだけでなく、**1 画面を開くときに実際に落ちてくる合計**にも上限を置いているのがポイントです。ファイルを分割しただけで合計は変わっていない、という改善もどきを弾けます。

もう 1 つ、サイズと同じくらい効いているのが**外部ホストの許可リスト**です。

```js
const ALLOWED_HOSTS = [
  'static.cloudflareinsights.com',
  'cloudflareinsights.com',
  'tile.openstreetmap.org',
  'nominatim.openstreetmap.org',
  'msearch.gsi.go.jp',
  '*.supabase.co',
]
```

ビルド成果物の中に、ここに無いホストへの参照が入ったら落ちます。Web フォントで 2.8MB 持っていかれた経験から入れました。検査スクリプトの実物はこれです（https://github.com/ishizakahiroshi/manabi-map/blob/main/web/scripts/check-payload-budget.mjs ）。外から何かを読む変更は、たいてい「CSS を 1 行足すだけ」の顔をしてやってきます。そのときに人間の気づきに頼らない仕組みが 1 つ要る、という判断です。

書いたあとに、わざと上限を超えさせて落ちることを確認しました。**検査は、落ちるところを見るまで動いていると思わないほうがいい。** これは前に痛い目を見ています。

## リリースして、検収して、全部通った

ここまでを v0.9.3 としてまとめて本番に出しました。同じ日に 3 本目のリリースです（v0.9.1 が Web フォント撤去、v0.9.2 が地図データ分割）。

本番の機械検収はこうです。

- SPA ルート 10 本すべて 200。308 は 1 件も無し
- 404 であるべき 5 本すべて 404（`/nonexistent-xyz` `/map/deeper` `/auth/` `/family/` `/school/xxxxxxxx`）
- 静的 HTML 5 本すべて 200 で、本文量も従来どおり
- `/map` のヘッダが `/` と同一集合で、CSP に `wss://*.supabase.co` が入っている
- sitemap 6,419 件、地図用データ 812,905 バイト
- 画面下の版数が `v0.9.3` と完全一致

7 項目、全部期待どおり。CI も 3 本とも success。

いい気分で完了報告を書きました。

## 本番で開いたら、トップページが出た

報告した直後、動作確認をしてもらったら「地図は出てないんじゃないか」と返ってきました。

アドレスバーに `https://manabi-map.app/map` を入れて開くと、トップページが出る、と。

一瞬、頭が真っ白になりました。さっき `curl` で 10 本ぜんぶ 200 を確認したところです。ヘッダも見た。テストも通っている。

こういうとき、いちばんやってはいけないのが「ブラウザのキャッシュでは？」と言って済ませることだと思っています。当たっていたとしても、確かめずに言えばただの言い訳です。

なので、まず同じ URL を開いて再現させました。再現しました。`location.href` は `https://manabi-map.app/` になっていて、`/map` は消えています。

次に、どうやってそこへ移動したのかを見ました。

```js
const n = performance.getEntriesByType('navigation')[0];
({ name: n.name, redirectCount: n.redirectCount,
   redirectStart: n.redirectStart, redirectEnd: n.redirectEnd })
```

返ってきたのはこれです。

```json
{ "name": "https://manabi-map.app/",
  "redirectCount": 1,
  "redirectStart": 1,
  "redirectEnd": 3.2999999970197678 }
```

リダイレクトが 1 回起きていて、その所要時間は **2.3 ミリ秒**でした（`PerformanceNavigationTiming` の仕様: https://developer.mozilla.org/ja/docs/Web/API/PerformanceNavigationTiming ）。

ネットワークに出ていたら、どんなに速くても数十ミリ秒はかかります。2.3 ミリ秒は「外に出ていない」ということです。

念のため、キャッシュを外して同じ URL を取りました。

```js
const r = await fetch('/map', { cache: 'reload' });
({ status: r.status, redirected: r.redirected, url: r.url })
// → { status: 200, redirected: false, url: "https://manabi-map.app/map" }
```

サーバーは 200 を返しています。リダイレクトもしていません。`cache: 'reload'` は HTTP キャッシュを無視して取り直すオプションです（https://developer.mozilla.org/ja/docs/Web/API/Request/cache ）。

つまり、直っていました。直っているのに、ブラウザが聞きに行かなかった。

## 308 は、消しても消えない

308 Permanent Redirect は、既定でキャッシュ可能と定義されています（RFC 7538: https://datatracker.ietf.org/doc/html/rfc7538 ）。301 と同じ扱いです。

「恒久的」と言っているのだから、ブラウザが覚えて次から聞きに来ないのは、仕様どおりの正しい動作です。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/05_2026-09-12_cached-308_fig2.png)

上の図のとおり、修正後もサーバーには届かない経路が 1 本だけ残ります。ブラウザが自分の中で完結させてしまうので、サーバー側で何を直しても、その人には届きません。

これはつまり、こういうことです。

- **今回より前に `/map` を直接開いたことがあるブラウザは、直したあとも同じ挙動のまま**
- その人たちは修正前からトップに飛ばされていたので、今回で悪化したわけではない。ただし自動では直らない
- 新しく訪れる人と、今日以降に初めて直接開く人には、正しく地図が出る
- アプリ内のリンク（下のタブの「地図」など）はクライアント側のルーティングなので、最初から影響を受けていない

サーバーから解消する手段はありません。ブラウザが聞きに来ないので、ヘッダも応答も届けようがない。利用者側でキャッシュを消すか、クエリ付きの別 URL で開くか、そのどちらかです。

対処のしようがない、という結論は気持ちが悪いです。でも「対処できません」と書くのと「たぶん大丈夫でしょう」と書かないのとでは、次に同じ設計をするときの重みが違います。

301 と 308 を軽く打たないこと。これが今回いちばん高くついた学びでした。

## 自分の検証の抜け: `?probe=1` がキャッシュを外していた

もう 1 つ、自分の手順の話を書いておきます。

本番の検収で、地図のタイル枚数を測るときに `https://manabi-map.app/map?probe=1` という URL を使いました。測定のたびに `performance` のエントリをリセットしたかったので、クエリを付けて別ページ扱いにしたのです。

このクエリが、リダイレクトのキャッシュを外していました。

リダイレクトのキャッシュは URL 単位です。`?probe=1` が付いた瞬間、ブラウザにとっては初めて見る URL なので、素直にサーバーへ聞きに行きます。だから地図がちゃんと出て、タイルも数えられて、「本番も問題なし」になりました。

`curl` はそもそもキャッシュを持たないので、同じ理由で気づけません。

**修正前の URL を、修正前から使っているブラウザで開く。** これをやっていないのに「直った」と報告していたわけです。指摘されなければ、そのまま見落としていました。

恒久リダイレクトを廃止する修正では、検収項目にこれを入れないといけない。`curl` が 200 を返すことと、利用者の画面が直ることは、別の話です。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/04_2026-09-12_cached-308_illustration-loop.png)

## 今回学んだこと

4 つにまとめます。

**1. `_redirects` の rewrite 先に `/index.html` を書かない。**

Cloudflare Pages は `/index.html` を `/` に正規化し、その正規化が rewrite 先にも効きます。3 列目に `200` と書いても 308 になります。Pages Functions の中から `env.ASSETS.fetch("/index.html")` しても同じです。同じ仕様に外側と内側で 2 回ぶつかりました。

**2. 「何も起きなかった」を報告するときは、同じ経路で 1 件起こしてみせる。**

CSP 違反 0 件を確認するときに、許可していない宛先への接続を対照として同時に張りました。そちらが違反として捕まったので、0 件がリスナー不動作でないと言えます。検査が動いていないことを、検査に合格したことと取り違えないため。

**3. 反映の判定は「今回直したもの」そのもので見る。**

本番デプロイの完了を、`/map` が 308 から 200 に変わった瞬間で判定しました。ビルドの生成時刻やファイル件数は、1 つ前のビルドでも同じ値になり得ます。変わるはずのものではなく、**今回変えたもの**を見るのがいちばん確実でした。

**4. 恒久リダイレクトを廃止する修正は、`curl` と新規 URL だけで検収しない。**

301 と 308 はブラウザに残ります。サーバーを直しても、既にその URL を踏んだブラウザには届きません。検収には「修正前の URL を、修正前から使っているブラウザで開く」を必ず入れる。クエリを付けた瞬間、その検証は無効になります。

ついでに 5 つ目を挙げるなら、**見た目が動いていることは、ステータスコードが正しいことを意味しない**。今回いちばん長く放置されていたのは、`404.html` が SPA のシェルだったおかげで画面だけは正しく描けていた部分でした。壊れているのに誰も困らない状態は、直す動機が生まれないぶん厄介です。

## manabi-map はこんなときに刺さります

- 通える範囲にどんな高校があるのか、地図で一度に見たい人
- 文化祭や説明会で見聞きしたことを、学校ごとに家族で残しておきたい人
- 偏差値の数字を鵜呑みにせず、根拠を確認できた範囲だけ見たい人
- 進路サービスに住所を預けたくない人（設定地点は住所の文字列を保存せず、座標はおよそ 100m 単位まで丸めて持ちます）

インストールは不要で、ブラウザで開くだけです。アカウントを作らなくても地図と検索は使えます。

- 紹介ページ（スクリーンショットと機能一覧）: https://ishizakahiroshi.com/work.html?id=manabi-map
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/manabi-map

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## あわせて読みたい

- SEO 用のプリレンダー HTML が初期表示を壊していた。createRoot から hydrateRoot へ移すまで: https://qiita.com/ishizakahiroshi/items/0bfa08174025670c0b00
- Cloudflare の「10,000PV突破おめでとう」が 9 割 bot だったので、開き直って公開 API を作った話: https://qiita.com/ishizakahiroshi/items/f779e632b06a7c54a903
- 学校の一覧に住所を出しました。そのかわり、あなたの住所は保存しなくしました: https://qiita.com/ishizakahiroshi/items/3b457ffedd146b8bb496

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/08_2026-09-12_cached-308_illustration-memory.png)

## おわりに

アクセス解析を眺めるだけのつもりで開いた夜に、1 か月放置されていた壊れ方が 2 つ出てきて、ついでに地図の無駄も 1 つ見つかりました。

どれも「画面は動いている」ものばかりでした。だから誰も報告しないし、自分でも気づかない。今回見つかったのは、たまたまステータスコードを並べて眺めたからです。

正しく動いていることと、正しく答えていることは違う。次から `curl` で一覧を取るのを、リリース後の習慣に入れておこうと思います。

あと、301 と 308 は本当に軽い気持ちで打たないようにします。消しても消えないので。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-09-12_manabi-map-v093-cached-308/

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
