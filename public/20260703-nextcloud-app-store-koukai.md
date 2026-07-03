---
title: Nextcloud App Store に個人アプリを公開するときにハマったポイントまとめ
tags:
  - PHP
  - Security
  - OSS
  - nextcloud
  - GitHubActions
private: false
updated_at: '2026-07-03T14:29:21+09:00'
id: a32b3affeffd49acb2b4
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

Nextcloud 上に置いた `.html` ファイルを、安全にプレビューするだけの小さなアプリを作って、Nextcloud App Store（公式のアプリストア）に公開しました。作業自体は 1 日で終わったのですが、途中でハマりどころがいくつかあったので、同じことをやる人向けに残しておきます。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260703-nextcloud-app-store-koukai_infographic.png)

## 作ったもの

[Safe HTML Viewer](https://apps.nextcloud.com/apps/safe_html_viewer) という Nextcloud アプリです。ソースは GitHub に置いています。

https://github.com/ishizakahiroshi/nextcloud-safe-html-viewer

やることはシンプルで、ファイル一覧の `.html` に「Safe HTML preview」という file action を追加し、クリックすると別タブで開きます。ポイントはその開き方です。

- `Content-Security-Policy: sandbox allow-scripts allow-popups` を付けて配信する。`allow-same-origin` は付けない
- なので中の JavaScript は動くが、Nextcloud のログイン Cookie や同一オリジン API には触れない
- 表示直前に、メールアドレスや電話番号らしき文字列、プライベート IP、`api_key=` のような認証情報っぽい文字列をベストエフォートで `[REDACTED-...]` に置き換える。元ファイルは一切書き換えない

社内資料の HTML を Nextcloud で共有したいけど、うっかり生きた認証情報が張り付いたまま送ってしまう事故を減らしたい、という動機で作りました。完全な漏洩防止ではなく「ベストエフォートの安全プレビュー」と割り切っています。

## 証明書の申請、実は通っていた

Nextcloud App Store にアプリを出すには、まず自分の証明書を発行してもらう必要があります。手順はこうです。

```sh
openssl req -nodes -newkey rsa:4096 \
  -keyout safe_html_viewer.key \
  -out safe_html_viewer.csr \
  -subj "/CN=safe_html_viewer"
```

この CSR を `nextcloud/app-certificate-requests` リポジトリへ PR として送ると、Nextcloud 側の担当者が署名して `.crt` を返してくれます。

ここで最初にやらかしました。PR を出した後、GitHub の通知を見落としていて「申請したのに音沙汰がない」と何日か放置していたのですが、実際は 3 日で承認・マージされていました。`gh pr list --author <自分> --state all` で自分の PR を確認すればすぐ分かる話で、通知に頼らず能動的に見に行くべきだったと反省しています。

## アプリ登録 API でハマった話

証明書が手に入ったら、次は App Store 側にアプリを登録します。API は 1 本だけです。

```
POST https://apps.nextcloud.com/api/v1/apps
```

ボディに証明書と、app id に対する署名を JSON で入れて送ります。署名はこう作ります。

```sh
echo -n "safe_html_viewer" \
  | openssl dgst -sha512 -sign safe_html_viewer.key \
  | openssl base64
```

ここで気づかず 1 回失敗しました。`openssl dgst -sign` の出力はバイナリです。そのまま JSON の文字列フィールドに突っ込むと、エラーメッセージは `"No data provided"` としか返ってきません。原因が分かりづらい上に、`openssl base64` を挟むのを忘れていただけ、というオチでした。ドキュメントのコマンド例をそのまま実行すれば防げたはずなので、コピペの精度を上げるべき典型例です。

## description は HTML じゃなくて Markdown

登録が通ったので、`info.xml` の description を張り切って HTML で書きました。

```xml
<description>
  <![CDATA[
    <p>Safely view <code>.html</code> files...</p>
    <ul><li>...</li></ul>
  ]]>
</description>
```

公開されたページを見ると、`<p>` や `<li>` のタグがそのまま生テキストで表示されていました。App Store の description 欄は Markdown 記法で扱われる仕様で、HTML タグを書くとエスケープされてしまうようです。

```xml
<description>
  <![CDATA[
Adds a **"Safe HTML preview"** file action to `.html` files.

**Features**

- One-click sandboxed preview
- Best-effort redaction of secrets
  ]]>
</description>
```

書き直して事なきを得ましたが、地味に刺さるポイントでした。CDATA の中で行頭にインデントを入れるとコードブロック扱いされる（4 スペースや Tab は Markdown 的にコードブロック）ので、列 0 から書くのもコツです。

## screenshot の順序も XML スキーマ違反だった

`info.xml` にはスクリーンショットの URL も書けます。最初、`<screenshot>` を `<dependencies>` の後ろに置いていたのですが、これは XSD（スキーマ定義）上は順序違反でした。コメントアウトしていた間は検証が走らず気づかず、有効化した瞬間に発覚しました。要素の順序はスキーマ通りにする、という当たり前の話ですが、コメントアウトで隠れていたバグは見つけにくいという教訓です。

## file action の表示名が日本語ハードコードだった

これは公開前に見つけた自分のバグですが、file action の表示名がこう書かれていました。

```js
registerFileAction({
  id: 'safe-html-viewer',
  displayName: () => 'HTML プレビュー',
  // ...
})
```

グローバルに配る OSS アプリなのに、表示名が日本語固定です。`@nextcloud/l10n` の `translate()` に差し替えて、英語を既定にしつつ日本語だけ `l10n/ja.js` で翻訳を提供する形に直しました。

```js
import { translate as t } from '@nextcloud/l10n'
// ...
displayName: () => t('safe_html_viewer', 'Safe HTML preview'),
```

自作アプリを最初から国際化前提で書いていないと、こういう取りこぼしが出ます。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260703-nextcloud-app-store-koukai_hero.png)

## 公開前の実機検証は使い捨てコンテナで

App Store に出す前に、実際に Nextcloud で動くか確認したかったので、手元の検証用サーバーに Docker で使い捨ての Nextcloud を立てました。

```sh
docker run -d --name nc-test -p 127.0.0.1:8080:80 \
  -e SQLITE_DATABASE=nextcloud \
  -e NEXTCLOUD_ADMIN_USER=admin \
  -e NEXTCLOUD_ADMIN_PASSWORD=xxxxxxxxxxxx \
  nextcloud:33
```

このコンテナに `occ app:enable safe_html_viewer` でアプリを入れ、架空のサンプル HTML（メールアドレスや API キーっぽい文字列を含む）をアップロードして動作確認しました。

- 未認証ユーザーからは 401
- 他ユーザーのファイルからは 404（ACL がちゃんと効いている）
- レスポンスヘッダーに `Content-Security-Policy: sandbox allow-scripts allow-popups` が入っている
- 本文中の `alice@example.com` のような文字列が `[REDACTED-EMAIL]` に変わっている
- 元ファイルを WebDAV で取り直すと、変換前の中身がそのまま残っている

一通り確認できたところで、検証コンテナは丸ごと削除しました。本番相当の環境を汚さず、使い捨てで試せるのが Docker の良いところです。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260703-nextcloud-app-store-koukai_fig1.png)

上の図は、公開前に確認した項目をまとめたものです。CSP ヘッダー・redaction・ACL・原本不変の 4 点を、それぞれ別の観点（ブラウザの開発者ツール、grep、WebDAV での再取得）で確認しています。1 つの手段だけに頼らず、複数の角度から裏を取るようにしました。

## タグを打つだけで公開まで自動化する

`.github/workflows/release.yml` をタグ駆動にしておくと、あとは `git tag vX.Y.Z && git push --tags` だけで、GitHub Release の作成から App Store への公開まで全部走ります。

```yaml
on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  build:
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - name: Prepare tarball
        run: |
          git archive --format=tar --prefix=safe_html_viewer/ HEAD | tar -x -C dist
          tar -czf safe_html_viewer-${GITHUB_REF_NAME}.tar.gz -C dist safe_html_viewer/
      - uses: softprops/action-gh-release@v2
        with:
          files: safe_html_viewer-${{ github.ref_name }}.tar.gz
      - uses: R0Wi/nextcloud-appstore-push-action@<pinned-sha>
        if: ${{ env.APPSTORE_TOKEN != '' && env.APP_PRIVATE_KEY != '' }}
        with:
          app_name: safe_html_viewer
          appstore_token: ${{ secrets.APPSTORE_TOKEN }}
          download_url: https://github.com/${{ github.repository }}/releases/download/${{ github.ref_name }}/safe_html_viewer-${{ github.ref_name }}.tar.gz
          app_private_key: ${{ secrets.APP_PRIVATE_KEY }}
```

必要な secret は `APPSTORE_TOKEN`（App Store の設定画面から発行）と `APP_PRIVATE_KEY`（証明書申請時に作った秘密鍵）の 2 つだけです。ここまで整えておけば、次のバージョンからは version bump してタグを打つだけで公開が終わります。実際、この日のうちに説明文の修正だけで 2 回リリースを打ち直しましたが、両方とも数分で反映されました。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260703-nextcloud-app-store-koukai_fig2.png)

上の図は、タグを push してから公開までの自動フローです。ビルド・GitHub Release 作成・App Store への署名付き公開まで、人間が手を動かすのは最初のタグ push だけです。

## まとめ

- 証明書申請の結果は通知任せにせず `gh pr list --author` で自分から見に行く
- アプリ登録 API の signature は必ず base64 化する（生バイナリだと "No data provided" としか出ない）
- `info.xml` の description は Markdown 記法で書く（HTML タグはそのまま表示される）
- `<screenshot>` はスキーマの順序（`<dependencies>` より前）を守る。コメントアウト中は検証されないので注意
- 公開前の実機検証は使い捨て Docker コンテナが手軽で、後片付けもコンテナごと消すだけで済む

小さなアプリですが、公開までの手順を一通り経験できたのは収穫でした。次は別の Nextcloud アプリも同じ手順で出してみようと思います。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
