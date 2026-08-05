---
title: Microsoft Store でアプリ名が4パターン全滅したので、npm と crates.io の名前も全部取り直します
tags:
  - Windows
  - Rust
  - MicrosoftStore
  - msix
  - 個人開発
private: false
updated_at: '2026-08-05T16:45:46+09:00'
id: 6659a2afe2fdaad911ba
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![Remote Audio ヒーロー画像](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-05_msstore-name-conflict_hero.png)

10 日ほど前に、「まだ 1 行もコードを書いていないのに、名前だけ npm と crates.io と GitHub の 3 か所で押さえた」という記事を書きました（https://qiita.com/ishizakahiroshi/items/be424fcb61b18e33ee7f ）。あのとき決めた名前は `audioremote` です。

その名前が、Microsoft Store では使えませんでした。しかも 1 回ではありません。表記を変えて 4 回試して、4 回とも同じところで止まりました。

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-05_msstore-name-conflict_infographic.png)

## Remote Audio という Windows 11 用の OSS を作っています

自作で [Remote Audio](https://github.com/ishizakahiroshi/RemoteAudio) という Windows 11 用の常駐ツールを作っています。Windows 11 ホストの既定音声出力を、同じ LAN 上のブラウザから切り替える単一 exe のローカルサービスです。ホストの前に戻らず、ゲスト Win11・スマホ・VM から出力先を変えられます。

同じ悩みを持っている方は、下記で入ります。

```powershell
npm i -g audioremote
```

パッケージ名が製品名と食い違っているのは、まさにこの記事の題材です。次のリリースで `remoteaudio` へ移します。

リポジトリはこちらです（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/RemoteAudio

## 「この名前は既に使われています」を 4 回見た

Partner Center のアプリ名予約は、拍子抜けするほど簡単な画面です。新規作成 → MSIX or PWA app → 名前を入れて **Check availability** を押す。空いていれば緑のチェックが出ます。手順は公式に画像つきで載っています（https://learn.microsoft.com/en-us/windows/apps/publish/publish-your-app/msix/reserve-your-apps-name ）。

緑のチェックは出ませんでした。試した順に並べます。

- `audioremote`
- `AudioRemote`
- `audio remote`
- `Audio Remote`

4 つとも「既に使われています」でした。

ここで妙なことに気づきます。Microsoft Store をどう検索しても、その名前のアプリが出てこないのです。使われているのに、どこにもない。

これは公式ドキュメントの Note にそのまま書いてありました。

> You might find that you cannot reserve a name, even though you do not see any apps listed by that name in the Microsoft Store. This is usually because another developer has reserved the name for their app but has not submitted it yet.
> （出典: 上記 Microsoft Learn のページ）

誰かが予約だけして、寝かせている。同じページには「予約は 3 ヶ月使わないと解除される」とも書いてあります。3 ヶ月待つ、という選択肢は、まあ、ないです。

![試した 4 表記がどこで潰れたか](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-05_msstore-name-conflict_fig1.png)

図にすると、試した 4 表記はすべて同じ 1 か所で止まっています。大文字化もスペース挿入も、予約判定を通り抜ける役には立ちませんでした。Store の検索結果が空なのに予約だけ埋まっている、というねじれもここに出ています。

## 大文字とスペースをいじっても逃げられない

ここが今回いちばん実務的な学びでした。

先に断っておくと、**公式ドキュメントには「表記ゆれを正規化して同一視する」とは書かれていません**。書いてあるのは "All apps on the Microsoft Store must have a unique name." の 1 文だけです。ですから以下は仕様の説明ではなく、4 回叩いた結果の観測にすぎません。

観測としては、こうでした。

- 大文字小文字の違いは効かない（`audioremote` と `AudioRemote` が同じ扱い）
- スペースの有無も効かない（`audioremote` と `audio remote` が同じ扱い）

つまり「表記をちょっとずらして回避する」という発想は、この画面には通用しません。逃げ道は 2 つだけです。**別の語を足すか、語順を変えるか。**

語順を変えました。`Remote Audio` です。

もっともらしく言えば、README の 1 行目が "Switch your Windows 11 host's default audio output device" なので、英語として自然なのは Remote Audio の方だった、という話になります。でも正直に書くと、順番が逆です。通ったから決めました。理屈は後から付きました。

## 名前を 1 つ変えると、どこまで巻き添えになるか

![1 枚だけ差し替えたつもりが下の段まで連鎖する様子](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-05_msstore-name-conflict_illustration-1.png)

差し替えたのは上の 1 枚だけのつもりでした。実際には、その下に積んであったものが一緒に動きます。Store の表示名を 1 つ変えるだけのつもりが、その日のうちにここまで動きました。

- Store 表示名: `Remote Audio`
- MSIX の `Identity/Name`: `ishizakahiroshi.RemoteAudio`
- GitHub のリポジトリ名: `audioremote` → `RemoteAudio`
- ビルドされる実行ファイル: `audioremote.exe` → `RemoteAudio.exe`
- Web UI とタスクトレイの表示（日本語・英語とも）
- winget マニフェストの表示名

一方で、この時点で `audioremote` のまま残ったものもあります。

- npm のパッケージ名
- crates.io のクレート名
- 設定ファイルの場所 `%APPDATA%\audioremote\config.toml`
- HTTP API のパスとトークンの接頭辞
- winget の `PackageIdentifier`（`ishizakahiroshi.AudioRemote`）

表示名と識別子が割れた状態です。ここで手が止まりました。

## 分裂を抱えるか、全部寄せるか

一般論としては、据え置きが正解ということになっています。

npm も crates.io も、一度公開した名前は返せません。crates.io は yank しても名前を永久に占有します。設定ファイルのパスを変えれば、既に入れている人のトークンが消えます。winget の `PackageIdentifier` に至っては、そもそも改名という操作が存在しません。だから普通は「表示名だけ変えて、識別子は互換のために据え置く」に落ち着きます。

一度はそう書きかけました。理屈は通っています。

ただ、自分の場合はその理屈が効きませんでした。**壊れて困る既存ユーザーが、まだ 1 人もいないからです。**

v0.1.0 を出したのは 2026 年 7 月 31 日で、この記事の 5 日前です。npm と crates.io と GitHub Releases に置いてはありますが、実質使っているのは自分だけです。守るべき互換性が無いところで互換性を理由に分裂を抱え込むのは、面倒を将来へ送っているだけでした。

なので、寄せます。**v0.2 で npm と crates.io の名前も取り直して、全部 RemoteAudio に統一します。**

![統一できるかどうかを分けるのは技術ではなくユーザー数](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-05_msstore-name-conflict_fig2-v2.png)

図にしてみると、分岐しているのは技術的な難しさではありません。まったく同じ操作が、既に使っている人がいるかどうかで「やってはいけないこと」と「今しかできないこと」に反転します。

移行先はこうなります。

- npm: `remoteaudio`（空きを確認済み）
- crates.io: `remoteaudio`（同上）
- 設定ファイル: `%APPDATA%\RemoteAudio\config.toml`
- winget: `PackageIdentifier` も `ishizakahiroshi.RemoteAudio` へ
- 旧 `audioremote` は最終版を 1 つだけ出して「新しい名前へ移ってください」と案内し、以後は更新しない

winget にだけ締切がある、と書くつもりでした。初回 PR（https://github.com/microsoft/winget-pkgs/pull/411865 ）が open のうちなら `PackageIdentifier` ごと差し替えられて、マージされた瞬間に永久識別子になるからです。

**ところが公開したあとに PR を見に行ったら、マージ待ちではありませんでした。** バリデーションに失敗して止まっていました。

```
labels: Internal-Error-PR, Needs-Attention, Validation-Guide
```

原因はマニフェストの置き場所でした。バージョンフォルダがありません。

```
出したパス
  manifests/i/ishizakahiroshi/AudioRemote/ishizakahiroshi.AudioRemote.installer.yaml

正しい構造（同じアカウントの別パッケージは 8 バージョンすべてこの形）
  manifests/i/ishizakahiroshi/many-ai-cli/0.3.0/ishizakahiroshi.many-ai-cli.installer.yaml
                                          ~~~~~
```

マニフェスト本体の `PackageVersion` は `0.1.0` で正しく入っていたので、中身ではなく**配置だけ**の問題です。どのみち作り直しなので、この PR は閉じて v0.2.0 のリリース時に新しい識別子で出し直すことにしました。

**締切だと思っていたものは、最初から存在しませんでした。** 期限に追われて識別子を決めかけていたわけで、そこは調べ直してよかったです。

## それでも、完全には揃いません

寄せると決めた後で、揃わないものが 2 つ残ることに気づきました。

**npm は新規パッケージ名に大文字を使えません。** ですから `RemoteAudio` そのものは npm 名になりません。統一後も npm と crates.io では `remoteaudio` という小文字表記です。表示名は `Remote Audio`、実行ファイルは `RemoteAudio.exe`、パッケージは `remoteaudio`。読みは同じですが、字面は 3 通りあります。

**そして旧 `audioremote` は消えません。** crates.io は yank しても名前を永久に占有するので、新しい名前へ移った後も古い名前は墓標として残り続けます。検索に 2 つ出る状態は、統一しても完全には解消できません。

つまり「全部統一する」は、正確には「これから増える分は揃える」でした。すでに出したものは、出した時点で名前空間に残ります。ここは受け入れます。

## で、この Remote Audio は何をするアプリなのか

ここまで名前の話しかしていないので、中身も書いておきます。

机の上の Windows 11（ホスト）に、スピーカーと Nest Hub Max と有線イヤホンがぶら下がっています。でも普段作業しているのは、その隣にある別の Win11（ゲスト）です。会議が始まるたびにホストの前まで歩いて、設定を開いて、出力デバイスを選び直す。1 日に何度も起きるこれをやめたくて作りました。

やっていることは単純です。ホスト側で Rust 製の小さな HTTP サーバーが常駐して、同じ LAN に Web ページを配ります。手元のどの端末のブラウザからでもそれを開けば、ホストの再生デバイス一覧が出てきます。押せば、その場で Windows の既定になります。

![ゲスト側で開く Web UI](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-05_msstore-name-conflict_screenshot-guest.png)

ゲスト側で見えるのはこれだけです。上がマスター音量、下がホストに繋がっている再生デバイスの一覧で、緑の丸が今の既定。行をタップすると Console / Multimedia / Communications がまとめて切り替わります。右上の「LAN 露出中」は、ホスト以外からも繋がる設定になっていることの注意表示です。

ホスト側は窓を持ちません。通知領域のアイコンが入口で、右クリックするとこうなります。

![ホスト側のトレイメニュー](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-05_msstore-name-conflict_screenshot-host.png)

「共有 URL（接続用トークンを含む）をコピー」がゲストへの受け渡し口です。トークンを含むことをメニューの文言に書いてあるのは、これが実質パスワードだからで、起動時に黙ってクリップボードへ入れる作りにはしていません。

設計で外さなかったのは 4 点です。

- **Console / Multimedia / Communications の 3 役割を毎回まとめて切り替える。** 前 2 つだけ変えると、会議アプリだけが昨日のヘッドセットにつながったままになります。`IPolicyConfig` は 1 役割ずつしか設定できないので、切り替えた後に 3 つを再列挙して照合し、割れていたら 409 を返します
- **デバイスの指定は表示名ではなく Windows のデバイス ID。** Bluetooth 機器が少し違う名前で再接続しても、切り替えが壊れません
- **ゲスト側にインストールするものはゼロ。** 共有 URL を 1 回開けばトークンが `localStorage` に入るので、以後はブックマークだけでつながります。ネイティブのゲストアプリを配る案は恒久的に却下しました。exe を配ると導入手順が 0 から 1 に増えるうえ、スマホや Linux が同じ URL で繋がる汎用性まで失うためです
- **Web UI は vanilla JS。** npm も bundler も使っていないので、フロントエンドの依存が構造的にゼロです

コマンドはこのあたりです。

```powershell
RemoteAudio                  # トレイに常駐する（既定）
RemoteAudio list             # 再生デバイスと現在の既定を一覧
RemoteAudio set <id>         # 既定の出力デバイスを切り替える
RemoteAudio share            # LAN 用の共有 URL をトークン込みで表示
RemoteAudio token revoke <name>   # 再起動なしで即座に失効
```

## Remote Audio はこんなときに刺さります

- 音の出し先を変えるためだけに、別の PC の前まで歩いている人
- 会議アプリだけ古いヘッドセットにつながったままになった経験がある人
- ゲスト側に何かをインストールさせたくない人（スマホからも同じ URL で繋がります）
- npm / crates.io / GitHub Releases のどれで入れても、中身は同じ zip がいい人

いずれかに心当たりがあれば、`npm i -g audioremote` で試せます。設定ゼロで動きます。**次のリリースからは `npm i -g remoteaudio` になります**（旧名は最終版で案内を出して止めます）。

- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/RemoteAudio
- npm: https://www.npmjs.com/package/audioremote
- crates.io: https://crates.io/crates/audioremote

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## おわりに

Microsoft Store への提出は、**まだしていません**。名前が取れて、MSIX が作れて、プライバシーポリシーを書いて、掲載文と審査担当者向けのテスト手順を用意したところまでです。

別のアプリで一度だけ Store を通したことがあって、そのときは提出から約 2 日で、指摘ゼロで公開されました（https://note.com/ishizakahiroshi/n/n06dfb6919bb2 ）。そのときに踏んだ罠は別記事にまとめてあります（https://qiita.com/ishizakahiroshi/items/57e9b7933fe375fbb1e8 ）。ただ、あれは通信をしないローカルアプリでした。今回は LAN でポートを開けるサーバーです。同じようにいくとは思っていません。

受理されたら、また書きます。落ちたら、落ちた理由の方を書きます。たぶんそっちの方が役に立つので。

名前の統一も v0.2 と同時なので、まだ終わっていません。今の時点で言えるのは、**名前を揃え直せる期間は思っているより短い**ということくらいです。ユーザーが 1 人でも付いたら、今回やろうとしていることはもう選べません。

次に何か作るときは、npm を押さえる前に Partner Center の Check availability を叩こうと思います。3 分で済む話でした。

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
