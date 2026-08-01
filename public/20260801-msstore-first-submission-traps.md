---
title: "Tauri アプリの Microsoft Store 初回申請で踏んだ4つの罠と回避手順（MSIX / Partner Center）"
tags:
  - MicrosoftStore
  - MSIX
  - Tauri
  - Windows
  - 個人開発
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![Tauri アプリの Microsoft Store 初回申請で踏んだ4つの罠](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-31_msstore-traps_hero.png)

自作の Markdown エディタを Microsoft Store に申請しました。Tauri（Rust）で作った Windows の exe を MSIX で包む経路です。

申請そのものは半日で終わりました。ただ、その半日のうち体感 3 割くらいは「なんでこれ保存されないんだ」に溶けています。公式ドキュメントに書いてあるようで書いていない、書いてあっても画面と対応が付かない箇所が 4 つありました。

同じところで止まる人がいそうなので、罠と回避手順を先に並べます。

## 自作の Markdown エディタを Store に出しています

自作で [offline-md-editor-viewer](https://github.com/ishizakahiroshi/offline-md-editor-viewer) という Markdown エディタ／ビューアを作っています。オフラインで動く Markdown エディタ／ビューアで、単一 HTML と Windows 用ポータブル exe を配っています。

ブラウザ版はインストール不要で、下記で開けます。

```bash
npx offline-md-editor-viewer
```

リポジトリはこちらです（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/offline-md-editor-viewer

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-31_msstore-traps_infographic.png)

## 先に結論。踏んだのはこの4つ

- 「個人情報を利用しますか?」に **No を保存できない**（バグではなく仕様）
- restricted capability の説明欄に **約 500 字の上限**があり、超過分が無言で切り捨てられる
- **Notes for certification の入力欄が、そのページに無い**
- 値を変えてから Save を押す前にページを移動すると、**変更が消える**

以下、順に書きます。

## そもそもなぜ MSIX 経路にしたのか

Store への提出経路は MSIX と EXE/MSI の 2 つがあります。選んだのは MSIX です。決め手は 1 点だけで、**Store 側がパッケージを再署名してくれるのでコード署名証明書を買わなくていい**からでした。EXE/MSI 経路は自前の Authenticode 署名が必須で、個人の OSS には重すぎます。

なお Tauri v2 に MSIX bundler は統合されていません。公式ターゲットは msi と nsis だけです（https://v2.tauri.app/distribute/windows-installer/ ）。なので MSIX 化は `MakeAppx` を直に叩くスクリプトを自分で書きました。

もう 1 つ。個人開発者の登録料は**無料**になっています。以前の 19 ドルは免除されました（https://learn.microsoft.com/ja-jp/windows/apps/publish/whats-new-individual-developer ）。ここは古い情報が検索上位に残っているので注意です。

## 罠1: 「個人情報を利用しますか?」に No が保存できない

Properties ページに必須の設問があります。

> Does this product access, collect, or transmit personal information?

このアプリはネットワーク通信を一切しません。リリースビルドは Content-Security-Policy の `connect-src` を `'none'` に固定していて、アプリ自身が通信できないようになっています。当然 No です。

No を選んで Save。再読み込みしたら Yes に戻っていました。

操作ミスかと思ってもう一度やりました。Yes に戻ります。3 回目も Yes。

![同じ操作を3回繰り返して、3回とも同じ画面に戻される](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-31_msstore-traps_illustration-1.png)

ここで 30 分溶かしました。しかも面倒なのは、同じ Save で **Category や Website URL は正しく保存されている**ことです。だから「保存自体が壊れている」とも言い切れない。

答えは公式ドキュメントに書いてありました。

> If we detect that your packages declare capabilities that could allow personal information to be accessed, transmitted, or collected, **we will mark this question as Yes**, and you will be required to enter a privacy policy.
>
> （https://learn.microsoft.com/en-us/windows/apps/publish/publish-your-app/msix/support-info ）

capability を検出したら Microsoft 側がこの設問を Yes に塗り替える、と明記されています。つまり**バグではなく設計どおりの挙動**でした。

さらに Store Policy 10.5.1 にこうあります。

> Product types that inherently have access to Personal Information must always have privacy policies. These include, but are not limited to, **Desktop Bridge and Win32 products**.
>
> （https://learn.microsoft.com/en-us/windows/apps/publish/store-policies ）

Win32 と Desktop Bridge の製品は**常に**プライバシーポリシーが要る。実際に通信するかどうかは見ていません。製品種別と宣言 capability だけで機械的に決まります。

![runFullTrust の宣言からプライバシーポリシー必須に至る因果の連鎖](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-31_msstore-traps_fig1.png)

図のとおり、起点は `runFullTrust` です。Win32 の exe を MSIX で包む以上この capability は必須で、外すと `MakeAppx` のマニフェスト検証で落ちます（https://learn.microsoft.com/en-us/windows/uwp/packaging/app-capability-declarations ）。

**つまり MSIX 経路を選んだ時点で、プライバシーポリシーは回避不能です。**

対処はシンプルで、Yes を受け入れてプライバシーポリシー URL を入れるだけ。規約違反にもなりません。Microsoft 側が Yes にしているので虚偽申告の構図になりませんし、10.5.1 が求めているのは「ポリシー本文の内容が正確であること」であって「収集していると書け」ではないからです。本文には「一切収集しません」と正直に書けば要件を満たします。

むしろ危ないのは逆方向でした。No のまま URL を出さないと、審査で落ちる可能性があると公式に書かれています。

## 罠2: 説明が途中で切れているのに、エラーが出ない

`runFullTrust` は restricted capability なので、Submission options ページで「なぜ必要か」の説明を書かされます。公式には「できるだけ詳しく書け」としか書いていません。

なので詳しく書きました。700 字くらい。貼り付けて保存。

保存後に読み返したら、末尾が `files/folders the user expl` で切れていました。

**エラーは出ません。文字数カウンタもありません。**公式ドキュメントにも上限の記載が見つかりませんでした。

500 字以内に縮めたら通りました。

```
This is a Win32 desktop app (Tauri/Rust) packaged as MSIX. runFullTrust is required
because the package contains a full-trust Win32 app -- the standard way to package an
existing desktop app with MSIX (Desktop Bridge); without it, MakeAppx fails manifest
validation. The app makes no network requests (CSP connect-src 'none') and only accesses
files/folders the user explicitly opens via standard dialogs. No elevated privileges or
background services beyond a typical file editor.
```

481 字です。**保存したら必ず末尾まで読み返す**。これに尽きます。

## 罠3: Notes for certification の入力欄が、そのページに無い

Submission options ページに「Notes for certification」という見出しがあります。審査担当者向けの補足を書く欄です。

見出しはある。説明文もある。**入力欄が無い。**

よく読むと、説明文にこう書いてありました。

> Provide any additional details testers need to evaluate this submission **on the Additional Testing Information page**.

実体は左メニューの Supplemental info の下、**Additional Testing Information** ページの Description 欄でした。Submission options 側には見出しと案内文だけが置かれています。

親切といえば親切なのですが、見出しの直下に入力欄が無い画面は普通に迷います。

## 罠4: Save を押す前にページを移動すると、変更が消える

これは単純です。値を変えたあと、Save を押さずに別ページへ移動すると変更が失われます。

単体なら「まあそうか」で済む話です。ただ罠1 と重なると厄介でした。「No にしたのに Yes に戻っている」が、サーバー側の上書きなのか自分の保存漏れなのか区別が付かない。切り分けに余計な時間がかかりました。

**値を変えたら他の操作をせず即 Save。**Partner Center ではこれを徹底したほうがいいです。

## ついでに分かった小さいこと

罠というほどではないけれど、知っていると数十分助かるものを並べておきます。

- **掲載情報が必須なのは、MSIX の `<Resources>` が宣言している言語だけ**。このアプリの UI は 13 言語対応ですが、マニフェストの宣言は en-us と ja-jp の 2 つなので、掲載も 2 言語で提出できました。残りは任意で、公開後の更新でも足せます。13 言語ぶん手入力する覚悟をしていたので、これは効いた
- **Productivity カテゴリに subcategory は存在しない**。ドロップダウンが非活性のままなので一瞬バグを疑いますが、正常です。「Category and subcategory\*」のアスタリスクは category にだけ掛かっています
- **IARC の年齢レーティング質問票は App Type の 3 択から始まる**。Game / Social or Communication / All Other App Types で、ユーティリティ類は 3 番目。以降を全部 No で答えると、オフラインのテキストエディタは最低区分（IARC 3+、ESRB Everyone、PEGI 3、USK Everyone）になりました
- **画面の描画が不完全になることがある**。見出しだけ出てフォームが出ない状態に何度か遭遇しました。リロードで直ります

## 学んだこと

- **Store の判定は「アプリが何をするか」ではなく「パッケージが何を宣言しているか」で機械的に決まる。**CSP で通信を封じていても、`internetClient` を宣言していなくても、判定は変わりません
- **同じ操作を 2 回やって同じ結果になったら、それは操作ミスではない。**3 回目を試す前に調べたほうが早い。今回はこれを 3 回やってから調べました
- **公式ドキュメントは「読む」より「症状の文言で検索する」ほうが当たる。**罠1 の答えは、画面に出ていた英文をそのまま検索したら出てきました
- **初回提出は手動でしかできない。**Store submission API は「年齢レーティング質問票への回答を含めて、Partner Center で 1 回提出を作成済みであること」を前提にしています（https://learn.microsoft.com/en-us/windows/apps/publish/store-submission-api ）。しかも GitHub Actions 経由で更新する経路は、前提が「アプリが Store で公開済み（live）であること」とさらに厳しい（https://learn.microsoft.com/en-us/windows/apps/publish/msstore-dev-cli/github-actions ）。CI 化するにしても、最初の 1 回は手で通すしかありません

## offline-md-editor-viewer はこんなときに刺さります

- オフライン環境や、外部通信が制限された社内ネットワークで Markdown を書きたい人
- インストーラを入れられない PC や、USB に入れて持ち運びたい人（Windows 向けにポータブル exe があります）
- 書いているファイルを絶対にクラウドへ上げたくない人（依存ゼロ・ローカル保存です）
- Microsoft Store から入れたい人（現在審査中です。通ったら Store からも入ります）

`npx offline-md-editor-viewer` で 1 分で試せます。設定はゼロです。

- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/offline-md-editor-viewer
- npm: https://www.npmjs.com/package/offline-md-editor-viewer

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## おわりに

いま審査中です。通るかはまだ分かりません。

4 つの罠は、どれも「知っていれば 1 分、知らないと 30 分」の類でした。公式ドキュメントに書かれていないわけではなくて、**画面で困っている瞬間にそのページへ辿り着けない**のが本体だと思っています。

なので、こうやって症状の文言ごと書き残しておく。次に同じ画面で止まった人の検索に引っかかればいいなと思います。

小さく。踏んだ罠を、その日のうちに書き残していきます。

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
