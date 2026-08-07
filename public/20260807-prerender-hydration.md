---
title: SEO 用のプリレンダー HTML が初期表示を壊していた。createRoot から hydrateRoot へ移すまで
tags:
  - React
  - ssr
  - hydration
  - vite
  - SEO
private: false
updated_at: '2026-08-07T13:30:04+09:00'
id: 0bfa08174025670c0b00
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![SEO 用のプリレンダー HTML が初期表示を壊していた](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-07_prerender-hydration_hero.png)

自分のサービスを開いたら、一瞬だけ知らない画面が出た。黒い背景に、飾りのない文字がずらっと並んでいる。まばたきの間に消えて、いつもの画面に戻った。

見間違いかと思ってもう一度開いた。また出た。

## 何を作っているか

自作で **manabi-map** という進路検討サービスを作っています。住所を起点に通える範囲の高校を地図で見ながら、親子で比較・記録・検討できる進路管理サービスです。全国 47 都道府県・5,096 校を収録して、OSS として公開しています。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=manabi-map
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/manabi-map

サービス本体は https://manabi-map.app で動いています。

前回、このサービスの SEO 衛生をまとめて直しました（[全国5,095校のサイトマップを、今日1日で作り直した話](https://note.com/ishizakahiroshi/n/n3427e64c1d90)）。今回は、そのとき整えたはずのプリレンダー HTML が、実はユーザー体験の方を壊していた、という続きです。

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-07_prerender-hydration_infographic.png)

## 実測してみたら、CSS は間に合っていた

最初に疑ったのは CSS の読み込み遅延、いわゆる FOUC です。スタイルが当たる前に生の HTML が見えてしまう、あれ。

以下の数値は、すべて本番 https://manabi-map.app に対してブラウザの Performance API（https://developer.mozilla.org/ja/docs/Web/API/Performance_API ）で測った実測値です。

```
fonts.googleapis.com/css2                 | start=88  end=123
manabi-map.app/assets/index-*.css         | start=89  end=127
manabi-map.app/assets/index-*.js          | start=89  end=128
```

CSS は 127ms で到着しています。速い。これで「スタイルが間に合っていない」は消えました。

代わりに気になったのが JS のサイズでした。

```
index-*.js  decoded=833,140 B（単一チャンク）
domInteractive=51ms  DCL=92ms  load=207ms
```

これはディスクキャッシュが効いた状態の数字です。初回訪問なら、この JS をダウンロードして、パースして、実行し終えるまでの時間がまるごと乗ります。

そしてもうひとつ。本番の HTML を取得して中を見たら、`#root` の中にこう入っていました。

```html
<div id="root">
  <main>
    <h1>親子で使う、学校選びの地図ノート。</h1>
    <p>住所を入れると、通える高校が地図に表示されます。</p>
    <nav aria-label="一覧から探す">...</nav>
  </main>
</div>
```

クラスが 1 つも付いていません。Tailwind はクラスベースなので、CSS がロード済みでも preflight（リセット）しか当たらない。つまり **CSS の到着が速かろうが、この HTML は最初から「崩れた見た目」でしか表示されない**。

念のため、CSS 適用済みの本番ページで `#root` をこの HTML に差し替えてスクリーンショットを撮りました。最初に見た「知らない画面」と、文言も改行位置も一致しました。

## そもそも、なぜ静的な HTML を置いていたか

先に前提を書きます。React の SPA が配信する HTML は、素のままだと中身が空です。

```html
<body>
  <div id="root"></div>
  <script type="module" src="/assets/index-xxxxxxxx.js"></script>
</body>
```

ブラウザが JS をダウンロードして実行し、`#root` の中に画面を描く。人間が見るぶんにはこれで困りません。

困るのはクローラーです。Googlebot は JavaScript を実行してからインデックスしますが、GPTBot や ClaudeBot のような AI クローラーの多くは実行しません。空の `<div>` しか読めないので、学校ページが全部「中身の無いページ」として扱われます。

そこで、**ビルド時にクローラー向けの HTML をあらかじめ作って `#root` の中に入れておく**。これがプリレンダーです。ページを配信する前に中身を焼き込んでおく、というだけの話です。

```
vite build        → dist/index.html は #root が空のまま
gen-seo-pages.mjs → dist の各 HTML の #root に中身を注入
                    （学校ページ・県ページ・ハブ・法務・ガイド・404）
```

Cloudflare Pages は静的ファイルを SPA fallback より優先して返すので、`/school/xxxx/` を直接叩いた人にもクローラーにも、中身の入った HTML がそのまま届きます。

この仕組み自体は正しく動いていました。壊れていたのは、その HTML を **誰が書いていたか** の方です。

## 原因は「React が描くものと別物の HTML を置いていた」こと

コードを追うと 2 行で決着しました。

```js
// web/scripts/gen-seo-pages.mjs
/** #root に静的コンテンツ（クローラー向けの初期 HTML。mount 時に React が置き換える）を注入。 */
function withRootContent(html, content) { ... }
```

```tsx
// web/src/main.tsx
createRoot(document.getElementById('root')!).render(...)
```

クローラー向けの HTML をビルド時に手書きして `#root` に入れる。React は `createRoot` で起動するので、**マウントした瞬間に `#root` の子を全部捨てて描き直す**。

設計としては意図どおりです。コメントにも「mount 時に React が置き換える」と書いてある。問題は、置き換わるまでの数百ミリ秒、その手書き HTML が**そのままユーザーに見えていた**ことでした。

バンドルが大きいので、その窓が長い。回線が細ければもっと長い。

![旧方式と新方式で、初期 HTML と React 出力の関係がどう変わるか](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-07_prerender-hydration_fig1.png)

旧方式は「クローラー向けの HTML」と「ユーザー向けの画面」を別々に作って、後者が前者を上書きする形でした。新方式では前者を廃止して、後者をビルド時に文字列化したものを置きます。

## 対症療法を出したら、却下された

ここで自分は 3 つの案を出しました。

1. `index.html` の head に最小の inline CSS を入れて、プリレンダー HTML 自体を「読める見た目」にする
2. プリレンダー HTML を初期状態で非表示にして、React マウント時に置き換える
3. 巨大な単一チャンクを分割してマウントを速くする

返ってきたのは 1 行でした。

> そのあたりじゃなくて 根治 してください

これは正しい。1 は見た目を取り繕うだけで「初期表示と最終表示が別物」という構造は残るし、2 は隠したコンテンツを検索エンジンが軽視する可能性がある上に、そもそも SEO のために入れているものを隠すのは目的と矛盾します。3 は窓が短くなるだけで 0 にはならない。

根治は 1 つしかありません。**初期 HTML を React の実出力そのものにして、`hydrateRoot` で継ぐ。**

`createRoot` と `hydrateRoot` の違いはここです。`createRoot` は DOM を自分で作り直す。`hydrateRoot` は **既にある DOM をそのまま使い、そこへイベントハンドラと内部状態だけを載せる**。同じものが既に置いてあるなら、捨てて描き直す必要がない。だから置き換えが起きず、ちらつく瞬間が消えます。

## 隠すことは、原理的にできない

作業を始めてから気づいたのですが、この選択には見えていなかった帰結がありました。

hydration は「サーバーが書いた HTML」と「ブラウザで React が最初に描くもの」が一致していることが前提です。1 文字でも違うと React はその部分を捨てて描き直します。

つまり「HTML には情報 X を載せず、JS 起動後に X を出す」ができない。それをやると、まさに X の差分でツリーが描き直される。**今直そうとしているちらつきそのものです。**

このプロジェクトには「偏差値の数値をプリレンダー HTML に載せない」という運用がありました。ビルド時に禁止語チェックが走って、静的 HTML の `<main>` に「偏差値」が入るとビルドが落ちる。

```js
const FORBIDDEN_WORDS = ['ランキング', 'TOP', '狙い目', 'おすすめ', '偏差値', '通学時間']
```

ところが React 側の学校詳細ページは、見出しを `校名（県共：58〜62）` のような形式で出しています。括弧の中の数字が偏差値レンジです。**画面には元から出ている**。ただ HTML ファイルには書かれておらず、JS が動いて初めて出るものでした。

SSR に統一するなら、これが HTML に載ります。載せないためには UI から偏差値表示を消すしかない。それは製品の仕様変更で、本末転倒です。

結論として、方針をこう決めました。

- **静的 HTML には載せる**（画面に出している以上、隠すのは原理的に無理）
- **公開 API には出さない**（こちらで線を守る）

API 側は既に検査がありました。`/api/v1/` 配下の全 JSON を走査して、`deviation` を含むキーが 1 つでもあればビルドを落とす関数です。これは今回も一切緩めていません。序列化を禁じる語（ランキング / TOP / 狙い目 / おすすめ）も残しました。

「偏差値サイト化しない」の本質はランキング化の禁止であって、数値を物理的に隠すことではない、と整理し直した形です。

![1 枚めくった下から、まったく同じものが出てくる](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-07_prerender-hydration_illustration-lifting-sheet.png)

hydration でやっていることは、要するにこれです。上に載っている HTML をめくると、その下から React が描いた同じものが出てくる。同じでなかった瞬間に、React は上の 1 枚を捨てて描き直します。

## hydration を成立させるために消したもの

実装で一番手間だったのは、SSR 基盤そのものではなく「初回 render を一致させる」作業でした。

### localStorage を初期値に使っていた箇所

```tsx
// 変更前
const [locale, setLocaleState] = useState<Locale>(readStoredLocale)
```

これだと、言語を en に設定している人だけプリレンダー（ja 固定）と食い違います。React error #418 が出て、ツリーが描き直される（https://react.dev/errors/418 ）。

```tsx
// 変更後
const [locale, setLocaleState] = useState<Locale>(DEFAULT_LOCALE)
useEffect(() => { setLocaleState(readStoredLocale()) }, [])
```

初回 render は既定値に固定して、hydration 後に復元する。同値なら React が bail out するので余計な再描画も起きません。

自宅地点を保持している Context も同じ形でした。こちらは既存の `[session]` effect が未ログイン時に `localStorage` を読む経路を持っていたので、初期値を `null` に変えるだけで済みました。

### fetch したデータで初期表示が変わるページ

学校詳細ページは `/school-data/<id>.json` を fetch して描画します。SSR ではそのファイルを読んで描けますが、ブラウザ側の初回 render は「読み込み中」から始まる。ここが食い違う。

解決は、**SSR で使った payload をそのまま HTML に埋め込む**ことでした。この形は `hydrateRoot` の公式ドキュメント（https://react.dev/reference/react-dom/client/hydrateRoot ）でも前提として書かれている作法です。

```html
<div id="root">...</div>
<script type="application/json" id="__MM_INITIAL__">{"schools":[...]}</script>
```

`type="application/json"` なので実行されません。クライアント側はこれを `JSON.parse` して初回 render から使います。埋め込みは必ず `#root` の**外**に置きます。中に入れると React の管理外の要素が紛れて hydration が壊れる。

実データで確かめると、15.6KB の payload を渡した場合は 26,385 文字の完全な HTML が返り、渡さない場合は 4,447 文字（読み込み中の姿）でした。差がそのまま「クローラーに見せられる情報量」です。

法務ページとガイドは生の Markdown を、公開データのページは収録件数を、同じ仕組みで埋め込みました。ここで大事なのは **node 側で HTML 化して流し込まない**ことです。React の `react-markdown` が出す HTML と 1 文字でも違えば、そこで hydration が壊れます。生の Markdown を渡して、描くのは React に任せる。

## SSR にしたら、UI の欠陥が芋づる式に出てきた

ここからが今回いちばんの収穫でした。

出力を React に切り替えた瞬間、ビルド時の検査が次々に落ち始めました。落ちた理由を追うと、どれも**アプリ側の実際の欠陥**でした。

- **学校詳細ページに `<main>` が無かった。** データが揃うと `<main>` が消える作りで、skip-link（`#main-content`）のリンク先が存在しない状態だった
- **学校詳細ページに `<h1>` が無かった。** 見出しが `<h3>` だけだった
- **学校詳細内のリンクが `/school/<id>`（末尾スラッシュ無し）だった。** canonical は `/school/<id>/` なので、内部リンクが正規 URL を指さず 308 リダイレクトを 1 回挟んでいた
- **公開データのページが、SSR 時に収録件数を出せなかった**

どれも、JS が動いた後の画面を見ている限り気づけません。プリレンダーを手書きしていた間は、手書き側が正しい HTML を出していたので、UI 側の不備が表に出なかったわけです。

**手書きの静的 HTML が、UI の欠陥を隠すカバーになっていた。** これは事前に想像していませんでした。

もうひとつ、検査スクリプト側の教訓もあります。

```js
// 旧: 手書き HTML の形に依存していた
if (!main.includes(`<h1>${escapedName}</h1>`)) throw new Error(...)

// 新: 実装非依存にした
const h1 = main.match(/<h1[^>]*>([\s\S]*?)<\/h1>/)
if (!h1 || !h1[1].includes(escapedName)) throw new Error(...)
```

「`<h1>校名</h1>` と完全一致するか」は、出力の作り方を変えた瞬間に無意味になります。検査したいのは「h1 があって、そこに校名が入っているか」であって、タグの書式ではない。ここを完全一致で書いていたせいで、1 回 2 分前後のフルビルドを 6 回まわす羽目になりました。

## 結果

県ページ 47 枚を除く全ページ（トップ・学校詳細 5,096 枚・各種ハブ・法務・ガイド）が SSR + hydration に切り替わりました。

- `pnpm build` は 139 秒で完走。生成 5,096 ページ、sitemap 5,154 URL
- `pnpm preview` で主要 7 ルートを開いて、**console のメッセージ 0 件**。旧方式では同じ操作で React error #418 が必ず出ていました

県ページだけ残っているのは、県内全校のデータを埋め込む必要があるためです。ここから先の数値も本番ビルドの実測で、生成物は https://github.com/ishizakahiroshi/manabi-map のビルドから誰でも再現できます。県別 JSON は全項目を持つと北海道 6.5MB・東京 2.8MB あって、HTML には入りません。一覧表示に使うフィールドだけの軽量版を別に作っているところです（現状で東京 167KB。キー名が長いのがそのまま効いているので、`recruitment_status_code` のような名前を 1 文字に潰す作業が残っています）。

## 学んだこと

- **`createRoot` は既存の `#root` を捨てて描き直す。** クローラー向けの HTML を置いても、React が起動するまでそれが見えている。CSS の到着が速いかどうかは関係ない
- **hydration を前提にすると「初期表示だけ隠す」ができなくなる。** 隠すこと自体が食い違いになる。画面に出しているものは HTML にも載る、と受け入れたうえで、別の層（今回は公開 API）で線を引く
- **静的出力の検査を文字列の完全一致で書かない。** 「タグの有無 + 中身の包含」で判定すれば、出力の作り方を変えても壊れない
- **手書きの静的 HTML は、UI の欠陥を隠すカバーになる。** 剥がすと `<main>` が無い、`<h1>` が無い、内部リンクが canonical と違う、が一気に出てくる。剥がした方が健康です

## manabi-map はこんなときに刺さります

- 中学生の子どもの進路を、親子で一緒に考えたい人
- 通学できる範囲にどんな高校があるのか、まず地図で把握したい人
- 学校ごとの見学メモや通学メモを、家族で共有しながら残したい人
- 地図を開かずに、都道府県ごとの一覧から探したい人

登録なしで使えます。設定も要りません。

- 紹介ページ（スクリーンショットと機能一覧）: https://ishizakahiroshi.com/work.html?id=manabi-map
- サービス本体: https://manabi-map.app
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/manabi-map

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## あわせて読みたい

- [全国5,095校のサイトマップを、今日1日で作り直した話](https://note.com/ishizakahiroshi/n/n3427e64c1d90): 今回壊れていたプリレンダー HTML を整えた回。SEO 衛生の側から見た同じ場所です
- [地図を開かずに学校を探せるようにしました。あと、偏差値の説明を書き直しました](https://note.com/ishizakahiroshi/n/n7411dc73c390): 今回「静的 HTML に載せる」と決めた偏差値まわりの、表示ポリシー側の話
- [PostgreSQL の migration が「全部成功」したのに本番が壊れる](https://qiita.com/ishizakahiroshi/items/257b0fc493007e64db4b): 同じサービスで踏んだ別の「検証が通ったのに壊れている」パターン

## おわりに

一瞬のちらつきなので、直さなくても誰も文句は言わなかったと思います。ただ、初めて開いた人が最初に見る 0.3 秒がそれだと、たぶんもう戻ってこない。

対症療法を出して却下されたのは、正直ありがたかったです。あのまま inline CSS を当てていたら、崩れた画面が「読める崩れた画面」になっただけで、UI 側に隠れていた欠陥も見つからないままでした。

構造が原因だと分かったら、その層で直す。当たり前のことを、また 1 回覚え直しました。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
