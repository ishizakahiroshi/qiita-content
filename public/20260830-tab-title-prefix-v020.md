---
title: Firefox 拡張に URL ルールを足して v0.2.0 を出した。SPA 追随は pushState の差し替えでは効いていなかった
tags:
  - Firefox
  - ブラウザ拡張
  - WebExtensions
  - JavaScript
  - 個人開発
private: false
updated_at: '2026-08-30T17:54:00+09:00'
id: 4914f4c0514d3678d71d
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-30_tab-title-prefix-v020_hero.png)

同じサイトを 3 つのアカウントで開くと、タブが全部同じ名前になります。X でも管理画面でも同じです。7 月に公開した v0.1.0 は、そこに Firefox のコンテナ名を差し込むだけの拡張でした。

今日 v0.2.0 を Firefox Add-ons へ出しました。URL ルールを足し、SPA での追随を作り直し、コンテナ名の変更が開いているタブへ届くようにしています。書いてみると、直した 3 つのうち 2 つは「動いていると思っていたコードが、実は最初から効いていなかった」話でした。

## 何をする拡張か

自作で [tab-title-prefix](https://github.com/ishizakahiroshi/tab-title-prefix) というブラウザ拡張を作っています。タブのタイトルに、コンテナ（Firefox）や URL ルール（Chrome）ベースで接頭辞を付ける拡張です。似たタブが並んでも一目で見分けられます。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=tab-title-prefix
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/tab-title-prefix

Firefox なら Add-ons から入ります。

https://addons.mozilla.org/ja/firefox/addon/tab-title-prefix/

入れた直後から、Multi-Account Containers で開いたタブのタイトル先頭に `[コンテナ名] ` が付きます。設定ファイルを書く必要はありません。URL ルールを使いたいときだけ、`about:addons` の拡張機能カードから設定画面を開いてルールを 1 行足します。

Chrome 版もビルドできますが、ウェブストアには出していません。Chrome には Multi-Account Containers 相当の機能が無く、URL ルールだけの汎用タブリネーム拡張は既に定番が複数あるためです。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-30_tab-title-prefix-v020_infographic.png)

## 前回の記事

この拡張の初回公開（v0.1.0）と、そこに至るまでの話は Zenn に書きました。今回はその続きになります。

- 初めての Firefox 拡張、2 回落ちてから 3 日で公開された: https://zenn.dev/ishizakahiroshi/articles/20260706-amo-approved-after-two-rejections

## v0.2.0 で増えたのは「自分でルールを決められる」こと

v0.1.0 はコンテナ名を出すだけでした。コンテナを使っていない人には何も起きません。

v0.2.0 では URL パターンとプレフィックスの組を自分で作れます。`https://github.com/*` に `[GH] ` を割り当てると、GitHub のタブだけタイトルが `[GH] ` で始まります。Firefox ではコンテナ名の後ろに連結されるので、`[仕事] [GH] リポジトリ名` のようになります。

設定画面まわりで足したのは次の 5 つです。

- ルールの追加、編集、削除、並び替え
- ルール単位の有効と無効、それと全ルールの一括停止
- 保存前のタイトルプレビュー（付けた結果がその場で見えます）
- 現在のタブから URL パターンを起こすボタン
- マッチパターンが不正なときの、何が悪いか分かるエラー表示

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-30_tab-title-prefix-v020_fig1.png)

プレビューを付けたのは、マッチパターンの書き方より「結果がどう見えるか」の方が知りたかったからです。`https://github.com/*` と `*://github.com/*` の違いを説明するより、付いたタイトルを見せた方が早い。

## pushState を差し替えても、SPA には効いていなかった

ここからが本題です。

v0.1.0 の content script は、SPA のページ内遷移を拾うために `history.pushState` を自前の関数で包んでいました。よくあるやり方です。遷移したら URL を見てプレフィックスを付け直す、という意図でした。

v0.2.0 を作りながら実機で確かめたら、この差し替えは最初から一度も効いていませんでした。content script はページとは別の isolated world で動くので、そこで `history.pushState` を書き換えても、ページ側の JavaScript が呼ぶ `history.pushState` は元のままです。Firefox でも Chrome でも同じです。

つまり「SPA に追随できている」ように見えていたのは、遷移のときに `document.title` も一緒に書き換わり、title の MutationObserver が反応していたからでした。タイトルを変えない遷移では、そもそも追随していなかったことになります。

直し方は地味です。差し替えをやめて、`lastUrl` に前回の URL を持ち、`location.href` と比較するだけにしました。比較のきっかけは head の MutationObserver です。

```js
// 差し替えではなく、変化に気づいたら比較する
function refreshIfUrlChanged() {
  if (location.href === lastUrl) return;
  lastUrl = location.href;
  applyPrefix();
}
```

行数は減りました。動く範囲は広がりました。効いていないコードを消しただけなので当然ではあります。

## Firefox でも `chrome` は定義されている

もう 1 つ、効いていなかったコードがあります。

Chrome 版では URL ルールを追加するときに、そのホストへの権限をユーザーへリクエストする必要があります。Firefox 版では manifest 側で権限を持っているので、リクエストは不要です。そこで「Chrome ならリクエストする」分岐を書いていました。

その判定が `typeof chrome !== "undefined"` でした。これが間違いです。Firefox は互換のために `chrome` グローバルも定義しているので、Firefox でもこの分岐に入ります。結果として Firefox でルールを追加すると、不要な origin 権限のリクエスト経路へ入っていました。

正しくは `browser` の側を見ます。

```js
// Firefox は chrome も browser も定義する。Chrome は browser を定義しない
const isFirefox = typeof browser !== "undefined";
```

同じ間違いを service worker 側にも書いていたので、2 箇所直しています。ブラウザ判定を書くときは「どちらが定義されていないか」で分けるのが安全です。

## `strict_min_version` を入れると lint が警告する

manifest まわりで 1 つ判断がありました。

`browser_specific_settings.gecko.strict_min_version` を入れるべきか。入れておくと古い Firefox にインストールされる事故を防げます。ただ、2026 年から必須になった `data_collection_permissions` と噛み合わず、`web-ext lint` が警告を出します。手元で測ったのは次の 3 通りです。

- `strict_min_version: "115.0"` … warnings 2
- `strict_min_version: "140.0"` … warnings 1
- 指定しない … warnings 0

警告を残したまま出しても審査は通るはずですが、0 件で出せる状態があるなら 0 件で出したい。今回は指定しないことにしました。

ちなみに `data_collection_permissions` は、データ収集をしない拡張でも明示が要ります。

```json
"browser_specific_settings": {
  "gecko": {
    "id": "tab-title-prefix@ishizakahiroshi.dev",
    "data_collection_permissions": { "required": ["none"] }
  }
}
```

これを書いておくと、AMO の製品ページに「開発者によると、この拡張機能はデータ収集を必要としません。」と自動で表示されます。プライバシーポリシーの欄は空のままで大丈夫でした。

## 設定は上書きせず、足すだけにする

v0.2.0 で設定のスキーマを v2 に上げました。移行処理を書くとき、v1 のキーを消して v2 のキーに置き換える書き方をしたくなります。

やめました。v1 が読むキーを残したまま、v2 のキーを足す形にしています。理由は単純で、開発版を一時読み込みして試したあと AMO 版の v0.1.0 に戻したときに、設定が壊れていると自分が困るからです。拡張 ID が同じなので storage は共有されます。

移行の分岐で気をつけたのは、自分より新しい `schemaVersion` を見つけたときに何もしないことです。将来の版が書いた設定を、古い版が読んで壊すのが一番まずい。

## 出したら、審査待ちなしで公開された

提出は Firefox Add-ons の Developer Hub から、既存アドオンの New Version として行いました。自動検証は errors 0 / warnings 0。

驚いたのはこの後です。提出した時刻と、API が返す `last_updated` が同じでした。

```
current_version: 0.2.0
file status: public
addon status: public
```

v0.1.0 のときは 2 回落ちて、Awaiting Review を経由して 3 日かかっています。同じアドオンへのバージョン追加は、審査キューに入らずそのまま公開されました。Mozilla の Add-on Policies には「When an add-on is given human review or otherwise assessed by Mozilla」という書き方があり、人手レビューが全件ではないことは読み取れます（https://extensionworkshop.com/documentation/publish/add-on-policies/）。ただ「初回だけキューに入る」と明文で書かれたページは、探した範囲では見つけられませんでした。今回はそうだった、というところまでが確認できたことです。

公開後もいつでも人手レビューの対象になり得る、とは提出完了画面にも書かれています。出して終わりではないようです。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-30_tab-title-prefix-v020_illustration-submit.png)

タグを打ったのは公開を確認した後にしました。AMO は手元で再現できない要件を随時足してくるので、タグを先に打つと manifest を直すたびに打ち直しになります。v0.1.0 のときに実際そうなりました。

## tab-title-prefix はこんなときに刺さります

- 同じサイトを複数アカウントで開いていて、タブが全部同じ名前になる人
- Firefox の Multi-Account Containers を使っていて、どのタブがどのコンテナか一目で分からない人
- コンテナは使っていないが、特定のサイトのタブだけ目印を付けたい人
- SPA のダッシュボードを開きっぱなしにしていて、ページ内遷移のたびに目印が消えるのが気になっていた人（v0.2.0 で直りました）

いずれかに心当たりがあれば、Firefox Add-ons から 1 クリックで入ります。設定ファイルを書く必要はありません。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=tab-title-prefix
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/tab-title-prefix

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## あわせて読みたい

- 初めての Firefox 拡張、2 回落ちてから 3 日で公開された: https://zenn.dev/ishizakahiroshi/articles/20260706-amo-approved-after-two-rejections （この拡張の初回公開の記録。今回の v0.2.0 はその続きです）
- 初めての Firefox 拡張を AMO に出したら、公開前に 2 回落ちた: https://zenn.dev/ishizakahiroshi/articles/20260703-amo-first-submission-rejected-twice （落ちた理由と、`data_collection_permissions` を足すまでの話）
- X の複数アカウント運用: Chrome 拡張で挫折 → Firefox Multi-Account Containers に流れ着いた記録: https://zenn.dev/ishizakahiroshi/articles/20260702-x-multi-account-firefox-container （そもそもこの拡張を作ることになった発端）

技術寄りではない、拡張そのものの紹介は note にも書きました: https://note.com/ishizakahiroshi/n/n8ac8e2ad786a

## おわりに

今回直した 3 つのうち、2 つは「効いていないコードを消しただけ」でした。動いているように見えていたのは別の理由で、そこに気づけたのは実機で 1 つずつ確かめたからです。

lint の警告 0 件は、たぶんそれ自体に意味はありません。ただ、0 件で出せる状態を保っておくと、次に何か警告が出たときに「増えた」とすぐ分かる。そういう地味な効き方をするものだと思っています。

小さく。動いていないコードを見つけたら消す、を続けていきます。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-08-30_tab-title-prefix-v020-url-rules/

※ ヘッダー画像とインフォグラフィックと本文の挿絵は AI（画像生成）で作成しています。

書いた人: ishizakahiroshi

システムエンジニア歴 18 年。バックエンドとインフラ、最近は AI 連携が中心です。業務委託を受け付けています（フルリモート対応）。

- サイト: https://ishizakahiroshi.com/
- GitHub: https://github.com/ishizakahiroshi
- X: https://x.com/ishizakahiroshi
