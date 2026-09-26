---
title: >-
  Codex CLI 0.157.0 が Windows で「path must be shorter than SUN_LEN」と言って起動しない。原因は
  CODEX_HOME のパスの長さだった
tags:
  - CodexCLI
  - Windows
  - AIエージェント
  - CLI
  - 個人開発
private: false
updated_at: '2026-09-26T14:21:20+09:00'
id: e1185939731120eb54dd
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/01_2026-09-26_codex-sun-len_hero.png)

朝、Codex CLI を 0.157.0 に上げたら、自作のダッシュボードから起動した Codex だけが動かなくなりました。普通のターミナルから起動すると、同じバージョンなのに何事もなく動きます。なんだそれ、と思いました。

出ていたのはこのエラーです。

```text
Error: app server did not become ready on <CODEX_HOME>\app-server-control\app-server-control.sock
...
Error: path must be shorter than SUN_LEN
To work without the background server, rerun the same command with --no-daemon (including resume or fork and its arguments).
```

先に結論です。**Windows で `CODEX_HOME` を長いパスに向けていると、0.157.0 の Codex は起動できません。目安は 64 文字以内です。** ユーザー名が極端に長くなければ、既定の `~/.codex` を使っている限り起きません。自分のように `CODEX_HOME` を切り替えて、アカウントごとに別の場所へ向けている人だけが踏みます。

## many-ai-cli について

自作で [many-ai-cli](https://github.com/ishizakahiroshi/many-ai-cli) という AIツール / Webダッシュボードを作っています。複数の AI コーディング CLI を並列で走らせ、承認をブラウザ 1 タブに集約。スマホからでも。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/many-ai-cli

同じ悩みを持っている方は、下記で入ります。

```powershell
pnpm add -g many-ai-cli
many-ai-cli setup
```

入れたら 1 回だけ `many-ai-cli setup` を実行します。PATH に載っていないときは `pnpm exec many-ai-cli setup` でも同じです。対応する AI CLI（Claude Code や Codex など）は、使うものを別に入れておいてください。

起動後の見え方は OS で違います。

Windows では、デスクトップに「MANY-AI-CLI」のショートカットができます。ダブルクリックするとトレイにアイコンが出るので、それを開いて「Hub を開く」を選ぶと、ブラウザで Hub が開きます。止めるときはトレイメニューの「Hub を停止」です。

macOS と Linux では「Many AI Hub Start」と「Many AI Hub Stop」ができます。Start を開くと、ブラウザと一緒にコンソールの窓が出ます。**そのコンソールが Hub 本体なので、× で閉じると Hub が止まります。** 邪魔なら最小化してください。

ターミナルから直接起動するなら `many-ai-cli serve --open` です（全 OS 共通）。Hub が開いたら、左下の「+ New Session」から Codex などを起動します。

README には、実機で確認できているのは Windows のローカル Hub で、macOS と Linux のネイティブ環境は十分には確認できていない、と書いてあります。この記事で扱うのは Windows での話です。

この話には前があります。複数の契約（アカウント）を切り替えて使えるようにしたのが v0.8.0 で、今回の `CODEX_HOME` の切り替えはそのときに足した仕組みです。

前回の記事: [AI CLI を 7 本並べたら、次に要るのは「契約」と「引き継ぎ」と「委譲」だった。many-ai-cli v0.8.0](https://qiita.com/ishizakahiroshi/items/49f6a29a43bf64168c1a)

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/02_2026-09-26_codex-sun-len_infographic.png)

## 朝に Codex を上げたら、Hub から起動した分だけ動かなくなった

many-ai-cli は、ChatGPT の契約を複数持っている人が使い分けられるように、契約ごとに `CODEX_HOME` を別のフォルダへ向けて Codex を起動します。ログインがフォルダごとに別々に保たれるので、契約が混ざりません。

更新のあと、この起動が全部だめになりました。同じ Codex を普通の PowerShell から `codex` と打つと、ちゃんと立ち上がります。違いは `CODEX_HOME` だけです。

## 「パスが長い」という見立ては、半分しか合っていなかった

AI に調べてもらうと、最初に返ってきたのは「Unix ドメインソケットのパスが長すぎる。`CODEX_HOME` が深いのが原因」という説明でした。エラー文とは合っています。

でも、引っかかりました。前回まで、同じフォルダで普通に動いていたからです。パスは何も変えていない。「長すぎる」だけでは、昨日まで動いていたことの説明がつきません。それで「前回は普通に動いてたよ。切り分けがちゃんとできてないんじゃないの」と突っ込みました。

そこで、ファイルの時刻を並べてもらいました。

- Codex 本体の `package.json` が書き換わった時刻は、朝 9 時 9 分台
- そのアカウント用フォルダの中の `packages` と `app-server-daemon` が**作られた**時刻は、その約 40 秒後
- 同じフォルダの `sessions` が作られたのは、1 か月以上前

つまり、バックグラウンドサーバー用のフォルダは、更新の直後に初めて作られていました。それまで存在していなかったものが、更新後の最初の起動で生えている。引き金は 0.157.0 への更新です。パスの長さは、その引き金が引かれる条件のほうでした。

補足すると、`codex app-server daemon` というコマンド自体は前からあります。上流の Issue には、2026 年 6 月に上がった macOS での報告があります。`CODEX_HOME` が長いと、同じ `path must be shorter than SUN_LEN` で落ちる、という内容です。macOS のソケットのパス上限は 104 バイトで、報告されたバージョンは 0.137.0 と 0.139.0 でした。ソケットの場所は `$CODEX_HOME/app-server-control/app-server-control.sock` に固定で、変える設定は無い、とも書かれています。9 月 26 日の時点で、この Issue はまだ open のままです。

https://github.com/openai/codex/issues/27765

自分の環境で起きたことを見る限り、通常の起動でもこのサーバーを使い始めたのが 0.157.0 だ、と読んでいます。ここは Codex のソースを読んで確かめたわけではなく、フォルダの作成時刻からの推測です。

## 何文字までなら通るのかを、実際に数えた

ソケットのパスは、`CODEX_HOME` の後ろに固定の 43 文字が付いたものです。

```text
<CODEX_HOME>\app-server-control\app-server-control.sock
            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  この部分が固定の 43 文字
```

Windows の Unix ドメインソケットは、パスの上限が 108 バイト（終端の NUL を含む）です。パスは 107 文字まで。そこから 43 を引くと、`CODEX_HOME` は **64 文字まで**になります。この 108 という値は Windows の仕様として知られているものですが、出典は確認していません。手元では 110 文字で落ちて、93 文字で通りました。境界そのものは試していません。

自分の環境で数えるなら、これで足ります。

```powershell
$suffix    = '\app-server-control\app-server-control.sock'
$codexHome = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $env:USERPROFILE '.codex' }
($codexHome + $suffix).Length   # 107 以下なら通る目安
```

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/03_2026-09-26_codex-sun-len_illustration-narrow-slot.png)

住所が長すぎて、細い投入口に入らない。今回起きたことを絵にするとこうなります。ソケットのパスはただの住所のようなもので、Windows 側の投入口が 107 文字分しか開いていませんでした。

実際に数えた例が、この 3 つです。ユーザー名は `alice` に置き換えています。

- 既定の `%USERPROFILE%\.codex` は、ソケットのパスが 64 文字。余裕で通る
- アカウント別の置き場所（フォルダ名に長い ID を使ったもの）は 113 文字。落ちる
- 同じ場所でフォルダ名だけ `p1` にすると 93 文字。通る

数え方の前提にしている固定のパス（`<CODEX_HOME>/app-server-control/app-server-control.sock`）は、上流の Issue に書かれています。
https://github.com/openai/codex/issues/27765

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/04_2026-09-26_codex-sun-len_fig1.png)

上の図のとおり、棒の長さの差はほとんどフォルダ名だけです。フォルダ名が 20 文字ほど違うだけで、上限の線をまたいでしまいます。

余談ですが、AI が最初にくれた説明では、既定のパスは 57 文字となっていました。数え直したら 64 文字でした。結論は変わりません。それでも、数字は自分の手でもう 1 回数えたほうが安全でした。

## 直し方は、フォルダ名を ID から切り離すこと

many-ai-cli では、契約の表示名から ID が作られ、その ID がそのままフォルダ名になっていました。表示名を長く付けると、フォルダ名も長くなる。一本の線です。

直し方は 2 通り考えました。

1 つ目は、ID を作るときに短く切ること。手は少なく済みますが、切り方しだいで別の契約と名前がぶつかります。切った結果、何のアカウントか分からない ID にもなります。

2 つ目は、フォルダ名だけを ID と別に持つこと。ID は今までどおり長くてよく、置き場所の長さだけを別に決められます。こちらにしました。

`config.yaml` にはこう書きます。

```yaml
subscriptions:
    codex:
        - id: chat-gpt-plus-personal
          name: Plus
          dir: p1
```

`dir` が空なら、これまでどおり ID をフォルダ名に使います。だから、すでにあるアカウントの置き場所は変わりません。

```go
// DirName は subscriptions/<provider>/ の下で使うフォルダ名を返す。Dir が空なら ID。
func (p SubscriptionProfile) DirName() string {
	if dir := NormalizeSubscriptionID(p.Dir); dir != "" {
		return dir
	}
	return NormalizeSubscriptionID(p.ID)
}
```

画面から新しくアカウントを足すと、`p1`、`p2` の順で空いている名前が振られます。

```go
func nextSubscriptionDirName(base string, used map[string]bool) string {
	for i := 1; ; i++ {
		candidate := fmt.Sprintf("p%d", i)
		if used[candidate] {
			continue
		}
		// 登録を外して認証だけ残ったフォルダを拾うと、前の契約のログインを引き継いでしまう。
		if _, err := os.Lstat(filepath.Join(base, candidate)); err == nil {
			continue
		}
		return candidate
	}
}
```

ディスクに残っているフォルダも避けているのは、登録だけ解除して認証を残したフォルダを次のアカウントが拾うと、前の人のログインで起動してしまうからです。これはテストにも入れました。

## 既存のフォルダを移したら、今度は別の場所で落ちた

新しく足すぶんはこれで済みます。すでにあるアカウントは、フォルダ名を変えて `dir` を書き足す必要があります。

```powershell
# Hub と、その中の Codex セッションを全部閉じてから
$root = "$env:USERPROFILE\.many-ai-cli\subscriptions\codex"
Rename-Item "$root\chat-gpt-plus-personal" p1
# config.yaml のそのアカウントに `dir: p1` を足す
```

これで起動し直したら、SUN_LEN は消えました。ところが今度は別のエラーです。

```text
Error: daemon executable not found at <CODEX_HOME>\packages\app-server-daemon\current\bin\codex.exe;
repair the existing installation, or run `codex app-server daemon start` to install a missing daemon.
```

`current` を見に行くと、これは junction（Windows のリンク）でした。しかもリンク先が、名前を変える前の絶対パスのままです。

```powershell
Get-Item "$root\p1\packages\app-server-daemon\current" | Select-Object LinkType, Target
```

移す前の場所を指しているのだから、実体が見つからないのは当たり前でした。フォルダごと名前を変えたのに、中に絶対パスを持つものがあった、というだけの話です。リンクを作り直しました。

```powershell
$pd     = "$root\p1\packages\app-server-daemon"
$target = Get-ChildItem "$pd\releases" -Directory | Select-Object -First 1   # バージョンが 1 つの場合
[IO.Directory]::Delete("$pd\current", $false)                                # リンクだけを消す
New-Item -ItemType Junction -Path "$pd\current" -Target $target.FullName | Out-Null
Test-Path "$pd\current\bin\codex.exe"                                        # True なら戻っている
```

このあと、Codex は起動しました。

ほかにも古いパスが残っていないか、フォルダの中のリンクと、テキストファイルの中身を探しました。リンクは今見た 1 本だけでした（もう 1 本は既定の設定側を指す意図したものです）。テキストのほうは、調べた範囲では、サンドボックスの権限を覚えておくファイルに、移す前のパスが 1 件だけ残っていました。今のところ困っていないので、そこは触っていません。

## 別のウィンドウが増えるのは、また別の話

起動できるようになったら、今度は真っ黒なコンソールのウィンドウが開いたままになっていることに気づきました。Codex を起動するたびに 1 つ増えます。

プロセスの親子関係を追うと、こういう形でした。

```text
Hub → wrap codex → codex（画面側）
  └ codex app-server --managed-daemon（バックグラウンドサーバー）
      └ node_repl.exe（ブラウザ操作などに使う MCP サーバー）
          └ 空のコンソール → Windows Terminal に新しいウィンドウ
```

普通のターミナルから `codex` を起動しても、同じウィンドウが出ました。なので many-ai-cli 側の問題ではなく、0.157.0 の挙動です。上流にも同じ報告が上がっています。どれも 0.157.0 で、9 月 25 日から 26 日に立っていて、9 月 26 日の時点でまだ open です。

- PowerShell から CLI を起動すると、Windows のバックグラウンドサーバーが見えるコンソールを 2 つ開く: https://github.com/openai/codex/issues/48090
- 更新した直後から、空のコンソールが開く（デスクトップアプリ・Windows 10）: https://github.com/openai/codex/issues/48114
- プロンプトを送るたびに、別々のコンソールが複数開く（CLI・Windows 11）: https://github.com/openai/codex/issues/48325
- VS Code でサインインすると、
ode_repl.exe と 
ode.exe の空ウィンドウが 2 つ出る: https://github.com/openai/codex/issues/48039

バックグラウンドサーバーは、セッションを閉じても動き続けていました（朝から動いているものが残っていました）。ウィンドウはそのサーバーの子なので、どのウィンドウがどのセッションのものかは分かりませんし、親のセッションを閉じても残ります。閉じるとブラウザ操作のツールが止まるかもしれないので、今は最小化して使っています。閉じても Codex 本体は止まらないはずですが、そこは試していません。修正を待つことにしました。

## 学んだこと

- 「前は動いていた」は、原因を絞るいちばん強い材料。長いパスは条件のひとつで、引き金は更新だった。更新の時刻と、新しくできたフォルダの作成時刻を並べたら早かった
- 環境変数で場所を切り替える作りにすると、その場所の長さも設計の一部になる。フォルダ名を ID と別にしておけば、あとから縛りが出ても逃げられる
- 名前を変えたフォルダは、中のリンクと絶対パスを疑う。`Get-ChildItem -Recurse -Force | Where-Object LinkType` で洗える
- AI が出した数字も、自分で数え直す。結論は合っていても、数字は違っていた

## many-ai-cli はこんなときに刺さります

- Claude Code・Codex・Copilot・Cursor・Grok を同時に走らせて、状態を 1 画面で見たい人
- 各 CLI の承認待ちを、1 つのタブでまとめて捌きたい人
- 離席中でも、スマホから承認したい人

いずれかに心当たりがあれば、`pnpm add -g many-ai-cli` と `many-ai-cli setup` の 2 コマンドで試せます。設定ファイルを書く必要はありません。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/many-ai-cli
- npm パッケージ: https://www.npmjs.com/package/many-ai-cli

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## あわせて読みたい

- [Windows の Codex CLI が WSL bash を勝手に使う問題を、Git Bash に固定して解決する](https://qiita.com/ishizakahiroshi/items/acec7f7c24db713cfcbe)（同じく Windows で Codex がハマった別件。今回の切り分けと並べて読める）

## おわりに

今回の修正は、手元では動いていますが、リリースにはまだ入っていません。

環境変数で場所を切り替えるツールを、この先また作ることがあったら、置き場所の長さを最初に数えます。数えるのに 1 分もかからなかったのに、今回はそこに来るまでに、ずいぶん回り道をしました。

ウィンドウのほうは、まだ増え続けています。直るのを待ちます。

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
