---
title: 作った PR が、リポジトリの持ち主にも 404。GitHub で bot アカウントが隠れたときの切り分けと回避手順
tags:
  - GitHub
  - 個人開発
  - Go
  - AIエージェント
  - bot
private: false
updated_at: '2026-09-06T13:14:27+09:00'
id: c9aea320ca2e76fa1fbe
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-04_invisible-bot-pr_hero.png)

朝の 6 時 24 分に、AI エージェントが Pull Request を作りました。テストは緑で、修正内容も筋が通っていました。18 分後、スマホの GitHub アプリで自分のリポジトリを開いたら、Pull Requests の件数が **0** と出ていました。

番号を直接叩くと `Could not resolve to an issue or pull request with the number of 5.` と返ってきます。リポジトリの持ち主は自分です。自分のリポジトリに、自分の bot が作った PR が、自分から見えない。ここから半日、Git でもコードでもない問題を追いかけることになりました。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-04_invisible-bot-pr_infographic.png)

## 先に結論。bot が隠れているかは 1 コマンドで分かります

同じことをやろうとしている方のために、判定と回避策だけ先に置きます。

```bash
# 未認証で bot のプロフィールを引く。404 なら「隠されている」
curl -s -o /dev/null -w '%{http_code}\n' https://github.com/<bot-account>

# 念のため親アカウント側も。こちらが 200 なら GitHub 全体の障害ではない
curl -s -o /dev/null -w '%{http_code}\n' https://github.com/<owner-account>
```

404 が返ってきたら、その bot が作る Pull Request はリポジトリの持ち主からも見えません。fork 経由でも、同じリポジトリ内でも同じです。回避策はこうなります。

- 作業ブランチは fork ではなく **本家リポジトリに直接 push する**（bot を Write 権限の collaborator にしておく）
- **bot では PR を作らない**。作っても誰にも見えないので、後始末が増えるだけです
- 本人（可視のアカウント）が `compare` 画面から PR を作る
- URL は `https://github.com/<owner>/<repo>/compare/<base>...<branch>?expand=1`

compare 画面とブランチ自体は未認証でも 200 で開けます。**見えなくなるのは「作者が隠れているアカウントである Pull Request」だけ**でした。ここが分かるまでに一番時間を使ったので、先に書いておきます。

## many-ai-cli を作っています

自作で [many-ai-cli](https://github.com/ishizakahiroshi/many-ai-cli) という AI コーディング CLI 用のローカル Web ダッシュボードを作っています。複数の AI コーディング CLI を並列で走らせ、承認をブラウザ 1 タブに集約。スマホからでも。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/many-ai-cli

同じ悩みを持っている方は、下記で入ります。

```bash
pnpm add -g many-ai-cli
# npm 派なら npm install -g many-ai-cli
```

入れたら 1 回だけ `many-ai-cli setup` を実行します。グローバルの bin がシェルにまだ反映されていない場合は `pnpm exec many-ai-cli setup` でもショートカットが作られます。

起動後の見え方は OS で違います。

Windows では `setup` がデスクトップに「MANY-AI-CLI」というショートカットを 1 つ作り、同じものをスタートアップフォルダにも置きます。ダブルクリックするとタスクトレイに常駐し、トレイメニューの「Hub を開く」でブラウザが開きます。停止も同じトレイメニューの「Hub を停止」です。

macOS と Linux では「Many AI Hub Start」と「Many AI Hub Stop」の 2 つが作られます（`.command` と `.desktop`）。Start を叩くとコンソール窓とブラウザが一緒に開きます。

ターミナルから直接動かしたい場合は、全 OS 共通で `many-ai-cli serve --open` が使えます。止めるときは別のターミナルから `many-ai-cli stop`、または Hub 画面の右上にある電源ボタンでも止まります。

開いたら画面の左下にある「+ New Session」から、claude / codex / copilot / cursor-agent / opencode / grok のどれかを起動するところから始まります。

この記事は、その many-ai-cli 自身のバグを AI エージェントに直させようとして、GitHub の可視性にぶつかった記録です。

この話には前があります。借りているクラウドサンドボックスから、失うと痛いアカウントの認証情報を引き上げた、という話でした。

前回の記事: [AI エージェントに開発マシンの鍵を渡していいのか。借り物のサンドボックスを実測して、9 案を比べた](https://qiita.com/ishizakahiroshi/items/52030bd9ae36f77124a7)

## 鍵を抜いた次に来たのは、名義の問題でした

前回、サンドボックスから個人の認証を引き上げました。そこまでは良かったのです。

では、そのサンドボックスの中で動いている AI エージェントに、自分の公開リポジトリを直させたいとき、**誰の名前でコミットして、誰の名前で PR を出すのか**。個人の Personal Access Token を置き直したら、前回やったことが台無しになります。

だから作業専用のアカウントを 1 つ作りました。GitHub の言葉でいう machine user です。公式ドキュメントにも、これは明示的に認められていると書いてあります。

> Accounts registered by "bots" or other automated methods are not permitted. However, if you want to create a single machine user for automating tasks such as deploy scripts in your project or organization, that is totally cool.

出典: [Managing deploy keys - GitHub Docs](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys)

「bot が登録したアカウントは禁止。ただし、デプロイスクリプトのようなタスクを自動化するために machine user を **1 つ** 作るのは全く問題ない」。この条文は今回の使い方にちょうど当てはまります。人間が作った、1 つだけの、自分のリポジトリのための作業用アカウント。

アカウントの作成日は 2026-09-02 でした。その 2 日後に隠れました。

## 35 リポを設定して 31 個 fork しました。たぶんここが分水嶺でした

作業用アカウントを用意したあと、サンドボックス側の環境を一気に整えました。指示はこういう内容です。

- ローカルの git identity はリポジトリごとに設定する（`--global` は使わない）
- `origin` は bot の fork、`upstream` は本家
- 日常のベースブランチは `develop`
- `main` / `master` / `develop` へは直接 push しない
- 作業ブランチは `grok/fix-…` か `grok/feature-…`

エージェントの結果レポートはこうでした。

```
configured:    35
fork-created:  31
failed:         4   (empty repositories cannot be forked / HTTP 403)
```

**新規アカウントが、作成から 1 日ほどで 31 個の fork を作った。** 振り返るとここが一番あやしいところです。GitHub の Acceptable Use Policies には、こう書かれています。

> We do not allow content or activity on GitHub that is: automated excessive bulk activity and coordinated inauthentic activity, such as spamming

出典: [GitHub Acceptable Use Policies](https://docs.github.com/en/site-policy/acceptable-use-policies/github-acceptable-use-policies)

「過剰な自動一括アクティビティ」。31 個の fork がそれに当たるかどうかは GitHub 側の判断なので断定はできません。ただし、他に思い当たる節がない、というのが正直なところです。悪意はまったく無かったのですが、外形だけ見れば「作りたてのアカウントが短時間に大量 fork」であり、機械的な検知にはよく引っかかる形をしています。

なお失敗した 4 件は、リポジトリは存在するけれど Git の中身が空で、そもそも fork できないものでした。これは仕様です。

## 監査で出たのは medium 2 件でした

環境が整ったので、本題に入りました。「many-ai-cli のバグを深い深度でチェックしてほしい」という依頼です。

Cloud Agent が使えない状況だったので、サンドボックス上の既存クローンでローカル深掘り監査になりました。読み取り中心で、テスト実行とレポート作成以外は何も変更していません。監査時点の `develop` の HEAD は `ddbc1b1` です。

結果は **HIGH なし、medium 2 件**。この 2 件が、どちらも「最近入れた機能の穴」だったのが面白いところでした。

### git-turn の終了スナップショットが reattach で捨てられる

many-ai-cli には、AI の 1 ターンごとに Git の差分を撮って Review 画面にカードとして出す機能があります。ターンの開始時にツリーの状態を控え、DONE を検知したら終了時のツリーを撮って差分を出す、という作りです。

問題は、終了時のキャプチャが goroutine で走っている最中に、ラッパーが再接続（reattach）したときでした。

```go
if s.sessions[sessionID] != ses || ses.gitTurnCaptureDone != captureDone {
    close(captureDone)
    return  // ターンが append されないまま終わる
}
```

reattach すると Hub 側は新しい `*session` を作り直します。走っていたワーカーは古いポインタを握ったままなので、この**ポインタ一致の判定に落ちて、ターンを記録せずに帰ります**。

厄介なのは、同じファイル内の `finishGitTurnWorktreeTree` は既に reattach を考慮していて、`gitTurnIndexDir` が一致していれば「論理的には同じセッション」として扱っていたことです。Git のインデックスは新しいポインタに着地するのに、**ターンの記録だけが捨てられる**。整合していませんでした。

さらに reattach 時の状態引き継ぎは `gitTurnStartTree`（開始時のツリー）を残したまま、in-flight フラグを false に戻します。この組み合わせで何が起きるか。

- 次のターンが始まっても `gitTurnStartTree` が空でないので、新しい基準点を撮らずに帰る
- 終了キャプチャの再実行も走らない
- 結果として Review の git-turn カードが 1 枚欠け、次のターンが開かないまま、その次の DONE で 2 ターン分がまとめて 1 枚になる

Git の I/O はミリ秒では終わらないので、WebSocket の再接続と競合する窓は現実的な幅がありました。

修正は素直で、ポインタ一致ではなく「同じポインタ、または同じ空でない `gitTurnIndexDir`」で判定する `gitTurnSessionMatches` を用意し、reattach 時に in-flight フラグと完了チャネルごと引き継ぐようにしました。回帰テストは `TestGitTurnEndCaptureSurvivesReattach` として追加しています。

### spawn-confirm が deciding のまま固まる

もう 1 件はブラウザ側です。子セッションを起動するときに確認ダイアログが出るのですが、その `decide()` の作りにこういう穴がありました。

- POST が HTTP 200 を返したら、そのまま `return` して WebSocket の `spawn_confirmation_closed` だけを待つ
- 待っている間（`state === 'deciding'`）は cancel ハンドラが `preventDefault()` を呼ぶので **Escape が効かない**
- ボタンは「処理中」の表示に差し替わっているので **Close も無い**
- タイムアウトも無い

Hub 側は `handleSpawnConfirmation` の中で `{ok:true}` を書いてから `performSpawn` を呼びます。つまり **HTTP 200 は「受け付けた」であって「終わった」ではありません**。WebSocket が死んでいたり、ブロードキャストを取りこぼしたりすると、Hub は子プロセスを起動しているのに、画面だけがリロードするまで閉じられなくなります。

修正は、HTTP 200 を受けたら 4 秒のフォールバックタイマーを張り、close のブロードキャストが来なければ Close ボタンを出して Escape を戻す、というものです。**2 回目の POST はしません**（二重起動を避けるため）。遅れて WebSocket の close が来たら、そちらで本当の結果に上書きします。

判定ロジックは DOM に触らないストア側へ出したので、bun のテストで「POST は 200 だが WS の close が来ない」ケースを直接書けるようになりました。

### テスト結果

```
go test ./internal/hub/ -count=1 -timeout 90s
  ok  many-ai-cli/internal/hub  25.584s

go test ./internal/hub/ -race -count=1 -timeout 60s \
  -run 'TestGitTurnEndCaptureSurvivesReattach|TestReattachPreservedStateTransfersInFlightGitTurnCapture'
  ok  many-ai-cli/internal/hub  1.173s

cd web && bun test tests/
  26 pass / 0 fail   (修正前は 22。spawn-confirm の HTTP/WS フォールバック 4 ケースを追加)
```

`-race` を付けた全体実行で `TestWorkflowJournalTailAdvancesAcrossFourMiBResult` が 1 件落ちましたが、これはパーサーのオフセット自体は進んでおり、100ms × 8 回というテスト側の予算が race detector 下で足りていないだけでした。本番の `tailWorkflowJournal` はタイマーで回り続けるので、製品のバグには数えていません。**落ちたテストを「フレークです」で流さずに、オフセットが進んでいることまで確認してから外す**、というのはここで守れた方だと思います。

## PR を出して、スマホで見たら 0 件でした

修正コミットは `db5061b`、ブランチは `grok/fix-reattach-git-turn-spawn-confirm`。bot の fork へ push して、本家の `develop` 宛てに PR #5 を作りました。API 上の作成時刻は 2026-09-03 21:24:58 UTC、日本時間で 9 月 4 日の 6 時 24 分です。

その 18 分後にスマホで見た画面がこれでした。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/shot1-mobile-pr-count-zero.jpg)

リリースの数もスターの数も正しく出ているので、リポジトリのページ自体はちゃんと表示されています。Pull Requests の行だけが 0 です。

Pull Requests: 0。リリースは 14 件、Star は 7 と正しく出ているので、リポジトリ自体は普通に見えています。PR だけが無い。

一覧を開くと #4 までしかなく、番号を直接指定するとこうなりました。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/shot2-could-not-resolve-5.jpg)

`Could not resolve to an issue or pull request with the number of 5.`

「解決できない」。存在しないのではなく、解決できない。この時点ではまだ、エージェントが PR を作ったと嘘をついている可能性も疑っていました。

## 3 つの視点で同じものを見ると、食い違いが見えました

ここからは切り分けです。**同じ URL を、認証の状態を変えて叩く**というだけの作業ですが、これで一気にはっきりしました。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-04_invisible-bot-pr_fig1.png)

表にすると、落ちているのは上 2 行だけだと分かります。bot のプロフィールと bot が作った PR は未認証でも持ち主でも 404。一方でブランチと compare、そしてコミットの署名は、どの立場から見ても生きています。この色の付いた 2 行が、あとで回避経路になりました。

bot の認証で `gh` を叩くと、PR #5 は open として普通に返ってきます。ところが未認証の Web では PR ページが 404。bot 自身のプロフィールページも、未認証だと 404。

つまり「bot にログインしている自分」だけが見えている世界がありました。**エージェントが嘘をついていたのではなく、エージェントから見えている景色と、こちらから見えている景色が違っていた**わけです。

そしてこれが今回いちばん効いた確認で、**リポジトリの持ち主である本アカウントから見ても、bot の PR は見えませんでした**。所有者権限をもってしても見えない。設定ミスで説明できる範囲を越えています。

この記事を書いている時点（2026-09-04）でも、状態は変わっていません。手元から取り直した実測を貼ります。

```console
$ gh api repos/ishizakahiroshi/many-ai-cli/pulls/5
{ "message": "Not Found", "status": "404" }
gh: Not Found (HTTP 404)

$ gh pr list --repo ishizakahiroshi/many-ai-cli --state all --limit 12
4  CLOSED  ishizakahiroshi  fix(linux): Claude Hub 黒画面 ...
3  CLOSED  ngav1491         fix(hub): Grok TUI overlays ...
2  MERGED  ngav1491         docs(i18n): add Vietnamese manuals ...
1  MERGED  ishizakahiroshi  feat: usage-links registry に providers.json を追加
```

`gh` はリポジトリの持ち主として認証しています。それでも #5 と #6 は列に並ばず、直接叩けば 404 です。

Web UI では、もっと分かりやすい形で食い違いが出ていました。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/shot3-open-zero-badge-one.png)

タブのバッジは **1**。一覧は **0 Open**。「There aren't any open pull requests.」

件数を数えているコードと、一覧を引いてくるコードで、可視性フィルタの掛かり方が違うのだと思います。GitHub の内部実装は分からないので推測ですが、少なくとも**同じページの中で 2 つの数字が食い違っている**のは観測できた事実です。数字が合わないのを見て、ようやく「これはこちらの設定ミスではない」と腹を決めました。

## コミットの署名は生きているのに、ユーザーは存在しない

記事を書くにあたって事実を取り直していて、いちばん奇妙な食い違いに気づきました。

```console
$ gh api users/ishizakahiroshi-bot
{ "message": "Not Found", "status": "404" }
gh: Not Found (HTTP 404)

$ gh api repos/ishizakahiroshi/many-ai-cli/commits/db5061b
{
  "sha":    "db5061b4120af34fed352aa4eb433df9db331c7d",
  "author": "ishizakahiroshi-bot",
  "login":  "ishizakahiroshi-bot",
  "date":   "2026-09-03T21:24:42Z"
}
```

**ユーザーとしては 404 なのに、コミットの author としては login まで解決して返ってきます。**

同じ API、同じ認証、同じアカウント名です。`/users/<name>` は「そんなユーザーはいない」と言い、`/repos/.../commits/<sha>` は「このコミットの作者は `ishizakahiroshi-bot` です」と言う。

隠されているのはアカウントの**エンティティ**であって、そのアカウントが過去に残した**署名**ではない、ということだと理解しています。だからコミットの帰属は保たれるし、`git log` にも普通に出る。一方で、ユーザーを参照して組み立てられるもの（プロフィール、collaborator 検索、そして Pull Request）は軒並み落ちます。

Pull Request は「作者」を必ず持つオブジェクトなので、作者が引けなくなった瞬間、オブジェクトごと解決できなくなる。`Could not resolve` という文言は、たぶんそのままの意味でした。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-04_invisible-bot-pr_illust1.png)

未認証のプロフィールも並べておきます。

```console
$ curl -s -o /dev/null -w '%{http_code}\n' https://github.com/ishizakahiroshi-bot
404
$ curl -s -o /dev/null -w '%{http_code}\n' https://github.com/ishizakahiroshi
200
```

親アカウントは 200 です。GitHub 全体が落ちているわけでも、ネットワークの問題でもありません。

同じ症状の報告は GitHub Community にもあります。プロフィールが公開ビューから消え、Google や GitHub の検索結果にも出なくなる、という内容です（[My GitHub account has been suddenly "flagged" and hidden from public view](https://github.com/orgs/community/discussions/27294)）。「flagged」という言葉が使われていますが、当人には通知が来ないケースが多いようです。

こちらにも、制限を知らせるメールは 1 通も届きませんでした。bot 用の Gmail の受信箱には、GitHub からの Welcome メールと sudo 確認コード、招待の通知はちゃんと届いています。制限に関するものだけが無い。

## collaborator に入れようとしたら、ユーザー名で検索できませんでした

「fork 経由が駄目なら、同じリポジトリの中で PR を作ればいいのでは」と考えました。そのためには bot を collaborator にする必要があります。

ユーザー名で招待しようとしたら、こうなりました。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/shot4-collaborator-not-found.png)

`Could not find a GitHub account matching ishizakahiroshi-bot`

自分が作ったアカウントで、今この瞬間もそのアカウントで `git push` が通っているのに、招待画面からは見つかりません。ユーザー検索も、ユーザーを参照する機能なので落ちる、という理解と一致します。

回避策はメールアドレスでの招待でした。ユーザー名ではなくメールアドレスを入れると Pending Invite が作られ、bot 側でログインして accept できます。ここは通りました。Write 権限の collaborator になっています。

なお途中で `Repository owner cannot be a collaborator` というエラーも見ましたが、これは本アカウント自身を collaborator に入れようとしたときの別のエラーで、今回の症状とは関係ありませんでした。**症状を追っているときは、関係ないエラーメッセージほど意味ありげに見えます。** これは違う、と切り分けるのに少し時間を使いました。

セキュリティの整理としては、自分が所有する公開リポジトリに、自分の作業用アカウントを Write で入れる、という形です。第三者のサービスに権限を渡しているわけではないので、この範囲なら許容と判断しました。

## 同じリポジトリ内の PR も見えませんでした

collaborator になったので、fork ではなく本家に直接ブランチを push して、same-repo の PR #6 を作りました。

これも未認証で 404 でした。

fork 経由かどうかは関係なかった、ということです。**Pull Request の作者が隠れているアカウントであれば、同じリポジトリの中にあっても見えない。** ここで方針を変えました。

- ゴーストになった #5 と #6 は、bot 側から close する（見えないまま open で残しても誰も触れないので）
- ブランチ `grok/fix-reattach-git-turn-spawn-confirm`（`db5061b`）は本家に残す
- 本アカウントで `compare` から PR を作り直す

closed の一覧を本アカウントで開いても、やはり #1 から #4 までしか並びません。close したことすら見えない、という状態です。

## 回避策は「ブランチだけ置いて、見える人が PR を作る」でした

compare の URL は未認証でも 200 で開けます。

```
https://github.com/ishizakahiroshi/many-ai-cli/compare/develop...grok/fix-reattach-git-turn-spawn-confirm?expand=1
```

ブランチもツリーも普通に見えます。**隠れているのは「作者」であって「コード」ではない**ので、コードが載っている場所は全部生きているわけです。

ここが分かってからは早かったです。本アカウントでこの compare を開き、Create pull request を押すだけ。作られる PR の作者は本アカウントになるので、当然ながら見えます。中身は bot が書いた `db5061b` そのままです。

コードレビューは手元の開発マシンで走らせている Grok に任せました。このとき出した指示で 1 つ大事だったのが、**「ローカルの dirty な HEAD と混同するな」**という念押しです。

```bash
git fetch
git diff develop...grok/fix-reattach-git-turn-spawn-confirm
```

3 点リーダの `...` を使う差分（マージベースからの差分）を見てほしい、ということを明示しました。ローカルの `develop` は既に先へ進んでいたので、`..` で撮ると余計なものが混ざります。AI に差分レビューを頼むときは、**どの 2 点の間を見るのかを人間の側が指定する**ほうが確実でした。

レビューが通って、本アカウントで squash merge。

```console
$ gh api repos/ishizakahiroshi/many-ai-cli/commits/ced34e0
{
  "sha":    "ced34e05454701ecd7f336f64d36db2afc50bbae",
  "author": "ishizakahiroshi",
  "date":   "2026-09-03T22:47:17Z",
  "msg":    "fix(hub,web): reattach 中の git-turn 終了キャプチャ欠落と spawn-confirm の deciding 固まりを直す"
}
```

日本時間で 9 月 4 日 7 時 47 分。PR #5 を作ってから 1 時間 23 分後です。**修正そのものは 6 時 24 分の時点で完成していたので、1 時間 23 分は丸ごと「見えない」への対処に使ったことになります。**

作業ブランチはリモートから削除済み、ローカルへの取り込みと CI が通っていることも確認しています。

面白かったのは、手元の Grok からのレポートに「紐づく PR が無い」と書かれていたことです。ブランチだけがあって PR が無いように見える。**bot が作った PR が本アカウントから見えないので、そう見えるのが正しい**わけですが、レポートだけ読むと状況が分かりません。可視性の問題は、こういうふうに下流のレポートを静かに歪めます。

## 規約はどう書いてあるのか、読みに行きました

「自分が悪かったのか」を知りたかったので、条文を当たりました。

machine user については前述のとおり、**1 つなら明示的に OK** と書かれています。今回作ったのは 1 つです。

一方で、Acceptable Use Policies には「automated excessive bulk activity（過剰な自動一括アクティビティ）」が禁止されている、とあります（[GitHub Acceptable Use Policies](https://docs.github.com/en/site-policy/acceptable-use-policies/github-acceptable-use-policies)）。31 個の fork がここに当たるかどうかは、GitHub 側の閾値を知らないので分かりません。ただ **「アカウント自体は規約に沿っているのに、その使い方が自動検知に引っかかった」** という形なのだろう、と理解しています。

規約違反かどうかと、検知に引っかかるかどうかは別、という当たり前のことを久しぶりに実感しました。

## reinstatement を出しました。回答は待っています

見えるように戻してもらうには、GitHub に頼むしかありません。公式の窓口は Appeal and Reinstatement のフォームです。

- 説明: [GitHub Appeal and Reinstatement - GitHub Docs](https://docs.github.com/en/site-policy/acceptable-use-policies/github-appeal-and-reinstatement)
- フォーム: https://support.github.com/contact/reinstatement

ドキュメントでは Reinstatement と Appeal が明確に区別されています。

> A "Reinstatement" is where a user wishes to regain access to their account or content and is willing to make any necessary changes to address the violation.

> An "Appeal" is where a user disputes that a violation has occurred and can provide additional information to show that a different decision should have been reached.

今回は「違反していない」と争いたいわけではなく、「もし過剰な一括操作と見なされたなら、以後はやらないので戻してほしい」という立場なので Reinstatement を選びました。**この 2 つを取り違えると、たぶん審査の筋が変わります。**

審査はすべて人間が行い、初回判定をした担当者とは別の人が独立に見る、とも書かれています（`All decisions on Appeal are made by humans and not by any automated means.`）。

実際に送った本文がこれです。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/shot5-reinstatement-request.png)

書くときに意識したのは 2 点だけです。

1. **主張ではなく観測を並べる。** 「bot でログインすれば動く」「ログアウトすると 404」「公開 API も 404」「持ち主から PR が見えない」「collaborator 検索で見つからない」「制限メールは受け取っていない」。全部、再現手順の形にしました
2. **用途を 1 行で言い切る。** 「自分の公開リポジトリのための個人開発 bot であり、第三者向けサービスでもスパムでもない」

回答はまだ来ていません。GitHub Community の書き込みを見ると数日から、長い人では数か月という話も出ているので、気長に待つつもりです。

なお、返事を待てずに何度もチケットを立てるのは逆効果だという話も複数出ていました（[How long do I have to wait for support to answer my reinstatement request?](https://github.com/orgs/community/discussions/111304)）。1 通出したら黙って待ちます。

## 手順を Skill にしました

同じ沼にもう一度落ちないように、手順をエージェント用の Skill に落としました。肝は **「作業を始める前に可視性をチェックして、Path A か Path B かを選ぶ」** という 1 点です。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-04_invisible-bot-pr_fig2.png)

図にすると、判断のポイントは 1 か所しかありません。未認証で bot のプロフィールが開けるかどうか。そこで分かれた先は、どちらも「最後は持ち主が取り込む」に合流します。

Path A は bot が見えているときの普通の流れです。fork に push して、bot が PR を作る。

Path B は隠れているときで、本家にブランチを push し、bot では PR を作らず、compare の URL を人間に渡して本アカウントで作ってもらう。今回はこちらでした。

Skill の中身で効いているのは、実は分岐そのものより **「bot では PR を作らない」を明文化したこと**です。今回は #5 と #6 という見えない PR を 2 つ作ってしまい、その後始末に手間が増えました。見えないものを close する作業は、画面上では何も起きていないように見えるので、やっていて不安になります。最初から作らないのが一番でした。

Skill そのものが消えると困るので、private リポジトリにバックアップも置いています。サンドボックスの中に本アカウントでログインはしたくなかったので、リポジトリの作成は手元のマシンで行い、bot を collaborator として招く形にしました。**前回の記事で決めた「借り物の環境に個人の鍵を置かない」という方針は、ここでも崩していません。**

## 学んだこと

1. **新しい作業用アカウントで大量に fork すると、他者から見えなくなることがある。** 通知は来ないことがあります。作るのは 1 つでも、その 1 つが短時間に何十回も同じ操作をすれば、外形は自動化された一括操作です
2. **「存在するのに見えない」は、認証の状態を変えて同じ URL を叩くと切り分けられる。** 未認証、当該アカウント、リポジトリの持ち主。この 3 つで景色が違ったら、こちらの設定ミスではありません
3. **隠されるのはアカウントであって、コミットの署名ではない。** `/users/<name>` が 404 でも `/repos/.../commits/<sha>` は login を返します。だからブランチと compare は生きていて、そこが回避経路になります
4. **同じページの中で数字が食い違うことがある。** タブのバッジが 1 で一覧が 0。片方だけ見て「無い」と判断していたら、もっと長く迷っていたはずです
5. **AI エージェントのレポートは、可視性の影響を受けたまま上がってくる。** 「紐づく PR が無い」という報告は、エージェントの視点では正しくて、状況の説明にはなっていませんでした。**AI の報告が変なときは、AI を疑う前に「その AI から何が見えているか」を疑う**ほうが早いことがあります
6. **Cloud Agent が使えなくても、既存クローン上のローカル深掘りで medium クラスまでは取れる。** 環境が理想的でないことは、監査をやらない理由にはなりませんでした

## あわせて読みたい

- [Grok Bot のクラウド PC に自作 CLI を入れた。Linux で Claude Code だけ真っ黒になる原因は CSI 6n の無応答だった](https://qiita.com/ishizakahiroshi/items/9818ab40e7a30e42cba7)（今回作業した同じサンドボックスに、many-ai-cli を最初に入れたときの話です）
- [Grok Bot のサンドボックス内でしか再現しないバグを、手元から詰める。Tailscale SSH を通すまで人力の伝書鳩をしていた話](https://qiita.com/ishizakahiroshi/items/6a9edc768efea629bf07)（サンドボックスと手元のマシンの間をどうつないだか。今回のレビュー往復もこの経路の上でやっています）
- [AI エージェントに開発マシンの鍵を渡していいのか。借り物のサンドボックスを実測して、9 案を比べた](https://qiita.com/ishizakahiroshi/items/52030bd9ae36f77124a7)（前回の記事。作業用アカウントを作ることになった動機はここにあります）

## many-ai-cli はこんなときに刺さります

- Claude Code や Codex、Copilot、Cursor、Grok を同時に走らせていて、どれが今止まっているのか分からなくなる人
- 各 CLI の承認待ちを捌くために、ターミナルのタブを行ったり来たりしている人
- 離席中もスマホから承認だけ返して、AI を止めずに動かしておきたい人
- 今回直した Review 画面の git-turn カードのように、AI が 1 ターンで何を変えたのかを後から追いたい人

いずれかに心当たりがあれば、`pnpm add -g many-ai-cli` と `many-ai-cli setup` の 2 コマンドで試せます。設定ファイルを書く必要はありません。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/many-ai-cli
- npm: https://www.npmjs.com/package/many-ai-cli

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## おわりに

バグ修正そのものは、正直たいした話ではありませんでした。ポインタ比較を 1 つ緩めて、タイマーを 1 本張っただけです。

難しかったのは、Git でもコードでもなく「存在するのに見えない」でした。しかも見えないことを教えてくれる仕組みが無いので、こちらから 3 通りの見え方を突き合わせるまで、何が起きているのか分からない。エージェントは「PR を作りました」と正直に報告していて、それも嘘ではなかったわけです。

`ced34e0` は無事 `develop` に入っています。Path A に戻れる日が来るのかは、GitHub からの返事次第です。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-04_invisible-bot-pr_illust2.png)

見えないものを追いかけるときは、まず視点を 3 つに増やす。それだけを覚えて帰ろうと思います。

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
