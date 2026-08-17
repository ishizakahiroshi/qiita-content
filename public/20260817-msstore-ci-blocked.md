---
title: Microsoft Store の更新を CI 化しようとしたら、公式手順の画面が存在しなかった
tags:
  - MicrosoftStore
  - msix
  - PartnerCenter
  - Tauri
  - CI
private: false
updated_at: '2026-08-17T15:27:18+09:00'
id: 2598b42565bd2fa4ccac
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-17_msstore-ci_hero.png)

Microsoft Store に出しているアプリの掲載版が、6 日間 0.3.1 のまま止まっていました。修正版の 0.3.2 は GitHub Release にも npm にも出ているのに、Store にだけ届いていない。理由は「2 回目以降の提出を自動化しようとして、その下準備が終わらなかった」からです。

今日その続きをやって、結論から言うと**自動化は完成しませんでした**。公式手順が指している画面が、私のアカウントの Partner Center には存在しなかったためです。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-17_msstore-ci_infographic.png)

## offline-md-editor-viewer を作っています

自作で [offline-md-editor-viewer](https://github.com/ishizakahiroshi/offline-md-editor-viewer) という Markdown エディタ／ビューアを作っています。ネット不要・単一 HTML で完結する Markdown エディタ／ビューアです。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=offline-md-editor-viewer
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/offline-md-editor-viewer

Windows デスクトップ版は Microsoft Store から入ります。

- https://apps.microsoft.com/detail/9N9FDS8BB2F6

前回の記事はこれです。同じアプリの初回申請が 2 日で通った話でした。

- 身構えて出したアプリが、2日でMicrosoft Storeの審査を通った話: https://note.com/ishizakahiroshi/n/n06dfb6919bb2

初回が通ったので、次は自動化だと思ったわけです。そこが今日の出発点でした。

## 先に結論

Microsoft Store の更新を GitHub Actions に載せるには、Store submission API を叩くための資格情報が 4 つ要ります。公式ドキュメントにも手順が載っています。

https://learn.microsoft.com/en-us/windows/apps/publish/msstore-dev-cli/github-actions

その手順の 2 番目がこれです。

> Entra ID でアプリを登録し、Account settings → User management → Microsoft Entra applications から追加する。このとき Manager ロールを割り当てる

**この「Microsoft Entra applications」タブが、私の Partner Center には無いです。**

Entra 資格情報で Partner Center にサインインした状態でユーザー管理を開き、ページ内の `[role="tab"]` を全部列挙しても、返ってくるのは「ユーザー」の 1 つだけでした。「ユーザーの追加」を押して出るダイアログも、選択肢は「新しいユーザーを作成する」のみ。アプリケーションを追加する項目がどこにもない。

Manager ロールを割り当てられないと submission API は使えない、と公式手順にも書いてあります。つまりここで詰みです。個人の Store 開発者アカウントで出ている UI と、ドキュメントが想定している UI が違う。企業アカウントなら出るのかもしれませんが、そこは確認していません。

というわけで今回は**手動アップロードで更新を出しました**。掲載版を直すことの方が優先だったからです。

## そこに至るまでに溶かした時間

詰んだと分かるまでに 1 時間ちょっとかかりました。途中で踏んだものを並べておきます。全部、次に同じことをやる人が引っかかるやつだと思います。

### テナント管理のページが 404 になる

公式手順は「Account settings → Tenants」と書いています。左メニューにもその項目があります。ただし URL が違いました。

```
/dashboard/account/v3/tenants          → notfound へリダイレクト
/dashboard/account/v3/tenantmanagement → 正解
```

左メニューのリンク自体は `tenantmanagement` を指しているので、メニューから辿れば問題ありません。URL を推測して直打ちすると外します。ついでに言うと、新しい UI（`/dashboard/v2/account-settings/*`）には Tenants の項目そのものがありません。

### 「作成」ボタンが押せない、と思ったら読み込み中だった

テナント一覧の上に「Microsoft Entra ID の作成」というボタンがあります。これがグレーアウトして押せない。

DOM を見たら `disabled` は付いておらず、opacity が 0.46 でした。一覧テーブルのヘッダ行が描画されるまでの間だけ非活性になる作りです。数秒待てば押せます。押せないと思って別の導線を探し始めると遠回りになります。

### 「無料で作成できる」テナントに、支払い方法の登録が要る

ここが一番の想定外でした。

Entra テナントを持っていない場合、Partner Center 内から新規作成できる、というのが公式の説明です。有料の Azure サブスクリプションも要らない、と。

実際にボタンを押すと `signup.microsoft.com` の「Microsoft for your business」に飛びます。ステップは 3 段です。

```
Account details → Sign-in details → Payment setup and finish
```

3 段目が支払い設定です。画面には「購入するまで課金されません」と明記されていますが、**カード番号とセキュリティコードの入力は必須で、スキップする導線がありません**。ページ内のボタンとリンクを全部列挙して探しましたが、あるのは Save だけでした。

課金はされないので実害はないのですが、「無料で作れる」という説明から想像する体験とは違います。

### バーチャルカードが 2 回弾かれた

実カードを預けたくなかったのでバーチャルカードを使ったところ、2 回失敗しました。

```
1 回目: Enter your payment data again or try a different way to pay.
2 回目: Check that the details in all fields are correct or try a different card.
```

3 回目に 3D セキュア（カード発行元アプリでの承認）を通る経路で成功しました。承認画面に出ていた金額は 0 円です。オーソリだけ立てて可否を見ている形なので、0 円オーソリを通さないカードだと弾かれるのだと思います。

### 住所フォームで Next が死ぬ

これは完全に UI のバグ寄りの挙動です。

請求先の組織情報フォームは、既定で Country/Region が United States になっています。氏名や住所を入れてから Country を Japan に切り替えると、フォームの構成が日本式に組み替わります（Zip が Postal Code になり、氏名の並びが姓・名の順になる）。

その組み替えのタイミングで、Next ボタンの上にある規約文言の枠が灰色のスケルトンのまま読み込まれなくなり、**全項目を埋めても Next が押せなくなります**。待っても直りません。

回避策は単純で、**空のフォームの状態で先に Country を Japan にしてから、他の欄を埋める**。これだけです。順番を変えるだけで規約文言が正しく出て、Next も青いままでした。

### 作ったばかりのテナントに拒否される

テナントの作成が Sign-in details まで進んだ時点で、実体は既にできています。ところが Partner Center から関連付けようとすると、こう返ってきました。

```
AADSTS5000228: Access to '<tenant-id>' tenant is denied.
```

このテナント ID は、作成完了画面に出ていたものと同じでした。つまり存在はしている。signup フローを最後まで完走してプロビジョニングが終わると、このエラーは消えます。**途中離脱した状態のテナントは、存在するけれど使えない**という中間状態があるようです。

### Partner Center のアカウント切り替えが見つからない

ここで一番時間を溶かしました。

Entra 資格情報で Partner Center にサインインすると、**「アプリとゲーム」ワークスペースが消えます**。見えるのは「マイ アクセス」だけ。Store の提出画面に入れません。逆に元の Microsoft アカウントに戻すと、今度はテナント管理が `notauthorized` になる。2 つの資格情報を行き来する必要があります。

まず試したのがこれです。

```
https://login.microsoftonline.com/common/oauth2/v2.0/logout
```

Entra アカウントだけを選んでサインアウトしました。Microsoft 側のセッションは確かに切れます。しかし Partner Center に戻ると、ユーザー名は Entra のままでした。**Partner Center は独自のセッションを持っていて、Microsoft のサインアウトでは切り替わりません**。

正解はこれでした。

```
ホーム画面 → ヘッダ右上のアカウントアイコン → 「別のアカウントでサインインする」
```

そして、ここが引っかかった理由です。**このアカウントアイコンは、ページによって描画されません**。私はずっと `account/v3/overview` で探していて、そこには出ないのです。ホーム（`/dashboard/home`）に行くと出ます。しかもヘッダの描画自体に数秒かかるので、ページを開いた直後は空です。

アクセシビリティツリーに要素が無いことを「存在しない」と読んで、一度は代行を諦めると宣言しました。実際には「そのページでは描画されていない」だけでした。

## 手動での更新提出

CI を諦めて手動に切り替えたら、ここは拍子抜けするほど簡単でした。初回申請と違って、掲載情報も年齢区分もプロパティも前回分が引き継がれます。触るのはパッケージだけです。

1. アプリの概要 →「製品の更新」の「更新の開始」で新しい Submission の下書きを作る
2. パッケージ画面で新しい MSIX をアップロードする
3. `Validating...` が終わって Validated になるのを待つ
4. Device family availability で新版がランク 1 に入り、旧版に取り消し線が付く。**ここで Save を押さないと旧版が残る**
5. 公開されるサポート連絡先に private なアドレスが混ざっていないか目視する
6. 「送信して認定を受ける」

提出後は概要画面のバッジが「更新プログラムの認定中」になり、ステータスが 申請 → 前処理中 → 認定 と進みます。画面には「通常数時間ですが、場合によっては最大 3 営業日かかることがあります」と出ました。前回は丸 2 日でした。

1 つ注意点があって、**「Microsoft Store のプレゼンス」欄は認定に合格するまで旧 Submission のまま**です。提出した直後にここを見て「反映されていない」と誤読しそうになりました。

### 提出物は必ず CI の成果物から作る

MSIX を作るとき、ローカルでビルドした exe をそのまま使わないようにしています。GitHub Release に上がっている exe をダウンロードし直して、そこから MSIX を作ります。

```powershell
gh release download v0.3.3 --pattern "offline-md-editor-viewer.exe" `
  --dir apps/desktop/src-tauri/target/release --clobber
.\scripts\release\build-msix.ps1
```

理由は単純で、**同じソースから作ってもローカルビルドと CI ビルドの exe はバイト列が違う**からです。今回も 9,058,304 バイトと 9,045,504 バイトで差がありました。Store に出したものと GitHub で配っているものが違うバイナリになると、後から追跡できなくなります。

生成後は MSIX を展開し直して、中の exe の sha256 が Release のものと一致するかを確認しています。MSIX 自体の sha256 は zip のタイムスタンプで毎回変わるので、同一性の根拠には使えません。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-17_msstore-ci_fig1.png)

図にしたとおりで、ローカルビルドは「動作確認用」、CI の成果物は「配布用」と割り切っています。この線を引いておかないと、手動提出のときに手近な exe を掴んでしまいます。

## v0.3.3 で中身も直しています

Store の話ばかりになりましたが、0.3.3 では監査で見つかった実装の穴も直しました。地味ですが、どれも「静かに壊れる」種類のものです。

### 64 MiB を超えるファイルを、読み込む前に断る

ブラウザ版でファイルを開くとき、`arrayBuffer()` でファイル全体をメモリに載せてから中身を判定していました。これだと巨大なファイルを開いた瞬間にメモリを持っていかれます。

デスクトップ版には元々 64 MiB の上限がありました。ブラウザ版だけ無防備だったので、**サイズを見てから確保する**順序に変えています。

```js
if (file.size > MAX_FILE_BYTES) {
  // ここで断る。arrayBuffer() を呼ぶ前
}
```

順序の問題なので変更自体は小さいのですが、上限を持っている側と持っていない側が同じアプリの中に同居していたのが良くなかったです。

### localStorage が例外を投げてもアプリを止めない

このアプリは UI 設定を localStorage に置いています。テーマ、文字サイズ、カードの表示状態、並び順など。

問題は、`localStorage.getItem` が例外を投げる環境があることです。プライベートウィンドウやストレージが無効化された環境では、アクセスしただけで throw します。起動時の設定読み込みでそれを踏むと、**インライン実行中の初期化が丸ごと止まる**。エディタ本体が動かなくなります。

読み書きを 3 つのヘルパーに寄せて、読めなかったら既定値へ倒すようにしました。

```js
function safeLocalStorageGet(key) {
  try { return localStorage.getItem(key); } catch { return null; }
}
```

保存キーも JSON の形も変えていないので、既存の設定はそのまま引き継がれます。

### フォルダの rename が、実装はあるのに呼べなかった

これが一番「あるある」でした。

デスクトップ版にはフォルダ名を変更する処理が実装されていて、バックエンドも動きます。ところが**画面のどこからも呼べませんでした**。右クリックメニューにフォルダ用の rename が無く、`F2` もファイルにしか効かない。

実装があることを知っている作者だけが「対応済み」と思っていて、使う側からは存在しない機能でした。context menu と `F2` の両方から呼べるように導線を足しています。

ついでに、README の英語版と日本語版で「フォルダ名の変更に対応」と書いてある箇所が、ブラウザ版でも対応しているかのように読める書き方になっていたので直しました。ブラウザ版は File System Access API の制約でフォルダの rename ができません。

### 直したことを CI で固定する

上の 3 つは、直しただけだと次のリファクタで戻ります。ビルドを伴わない回帰チェックを 1 本足しました。

```
scripts/ci/check-browser-runtime-safety.mjs
```

単一 HTML からストレージ系とファイル読み込み系のヘルパーを抜き出して実行し、上限の判定と例外時のフォールバックが生きているかを検査します。rename の導線がブラウザ版とデスクトップ版で正しく分かれているかも見ます。

このアプリはビルドツールを使っていない単一 HTML なので、テストランナーを持ち込むと構成が重くなります。必要な関数だけ切り出して Node で走らせる形に落ち着きました。

## 学んだこと

- **公式手順の画面が実在するかは、手順を読む前に見に行った方が早い。** 今回は下準備を 3 段進めてから「そもそもタブが無い」と分かりました。最初にタブの有無だけ確認していれば、テナント作成も支払い登録も要らなかった可能性があります
- **「見つからない」は「存在しない」ではない。** ページによって描画される要素が違う UI では、探す場所を変えるだけで解決することがあります。1 か所で見つからないことを根拠に打ち切り判断を出すのは早すぎました
- **同じソースでも、ビルドした場所が違えばバイナリは違う。** 配布物の出どころを 1 本に決めておかないと、後から「Store に出したのはどれか」が追えなくなります
- **実装があることと、使えることは別。** フォルダの rename は 1 行も新規実装していません。呼べるようにしただけです

## あわせて読みたい

- [Tauri アプリの Microsoft Store 初回申請で踏んだ4つの罠と回避手順（MSIX / Partner Center）](https://qiita.com/ishizakahiroshi/items/57e9b7933fe375fbb1e8): 同じアプリの初回申請の記録です。今回の記事は、その次の「2 回目の提出」で詰まった話になります
- [身構えて出したアプリが、2日でMicrosoft Storeの審査を通った話](https://note.com/ishizakahiroshi/n/n06dfb6919bb2): 初回が一発合格したときの記録です。この結果を見て「じゃあ自動化しよう」と思ったのが今回の出発点でした
- [Microsoft Store でアプリ名が4パターン全滅したので、npm と crates.io の名前も全部取り直します](https://qiita.com/ishizakahiroshi/items/6659a2afe2fdaad911ba): 別アプリですが、同じ Partner Center で名前の予約に苦戦した話です

## offline-md-editor-viewer はこんなときに刺さります

- ネット接続がない場所で Markdown を書きたい人。単一 HTML なので、ファイルを 1 つ持っていれば動きます
- 環境を汚さずに使いたい人。Windows 向けにはポータブル exe もあり、USB に入れて持ち歩けます
- 書いたものを手元に置いておきたい人。依存ゼロ・ローカル保存で、どこへも送信しません
- フォルダ操作までエディタ内で済ませたい人。0.3.3 でフォルダ名の変更を右クリックと `F2` から呼べるようにしました

いずれかに心当たりがあれば、Microsoft Store から 1 分で試せます。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=offline-md-editor-viewer
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/offline-md-editor-viewer
- Microsoft Store: https://apps.microsoft.com/detail/9N9FDS8BB2F6

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## おわりに

自動化しようとして、自動化できないことが分かっただけの日でした。

ただ、テナントもアプリ登録も残っています。Manager ロールを割り当てる導線さえ見つかれば、シークレットを発行するところから再開できます。今日やったことが全部無駄になったわけではない、と思うことにしています。

次に同じところで止まる人のために、踏んだ順に書き残しておきました。どれか 1 つでも時間の節約になれば。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-08-17_msstore-ci-blocked/

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
