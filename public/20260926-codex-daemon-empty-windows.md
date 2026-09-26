---
title: >-
  Codex CLI 0.157 が Windows で空のターミナル窓を起動のたびに開く。config.toml の 1
  行で止めた（ブラウザ操作は残したまま）
tags:
  - CodexCLI
  - Windows
  - WindowsTerminal
  - AIエージェント
  - 個人開発
private: false
updated_at: '2026-09-26T17:11:24+09:00'
id: 2e87da3628a9ac7322c3
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![ヘッダー画像](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/01_2026-09-26_codex-empty-windows_hero.png)

Codex を起動するたびに、中身が空の黒いターミナル窓が 1 枚ずつ増えていきました。2 つの契約（アカウント）を切り替えて使っていたら、気づいたときにはタスクバーに 4 枚並んでいて、なんだこれ、と思いました。

先に結論です。**`~/.codex/config.toml` に `daemon_auto_start = false` を 1 行足すと、Windows で空の窓は出なくなりました。** ブラウザ操作のツールも動いたままです。Codex CLI 0.157.1 で確認しています（2026 年 9 月 26 日時点）。

```powershell
codex features disable daemon_auto_start   # config.toml にこの 1 行が入る
codex app-server daemon stop               # すでに動いている裏のサーバーを止める
```

戻すときは `codex features enable daemon_auto_start` です。

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

この話には前があります。前の記事で、Codex 0.157.0 が Windows で起動しない件（`CODEX_HOME` のパスが長すぎる）を直したところ、次は空のウィンドウが増える話が待っていました。最後に「直るのを待ちます」と書いたのですが、待たずに止めてしまいました。

前回の記事: [Codex CLI 0.157.0 が Windows で「path must be shorter than SUN_LEN」と言って起動しない。原因は CODEX_HOME のパスの長さだった](https://qiita.com/ishizakahiroshi/items/e1185939731120eb54dd)

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/02_2026-09-26_codex-empty-windows_infographic.png)

## 最初は、自分のダッシュボードのせいだと思った

窓のタイトルの多くは、起動した子プロセスの実行ファイルのパスになっていました。手元では自作のダッシュボードから Codex を起動していたので、まず疑ったのはそちらです。

AI に調べてもらうと、コードの中に「子プロセスを起動するとき、コンソール窓を隠す指定をしていない箇所」が本当に見つかりました。トレイに常駐する Hub はコンソールを持たないので、git や CLI を呼ぶたびに窓が開く、というのは筋が通った話です。

そのまま、AI が 19 ファイルを書き換えました。コンパイルもテストも通りました。

ここで止めました。「この欠陥が今の窓を出している」という観察を、まだ 1 つも取っていなかったからです。欠陥があることと、その欠陥が今の症状の原因であることは、別の話です。書き換えは全部戻しました。

あとで分かったのですが、同じ日の午後に残していた作業メモに、実測済みの原因がすでに書いてありました。読まずに動いていたことになります。

## 窓を出していたのは、裏のサーバーの子プロセスだった

そこから観察です。新しく起動したプロセスと、Windows Terminal の窓のタイトルを、0.4 秒間隔で記録しました。コマンドラインは記録しません。引数にトークンなどが入っていることがあるからです。プロセスの記録だけなら、次の短い PowerShell で取れます。

```powershell
# 新しく起動したプロセスを、親をたどって記録する（0.4 秒間隔）
$seen = @{}
Get-CimInstance Win32_Process | ForEach-Object { $seen["$($_.ProcessId)/$($_.CreationDate.Ticks)"] = $true }
while ($true) {
  $all = Get-CimInstance Win32_Process
  $byId = @{}; foreach ($p in $all) { $byId[[int]$p.ProcessId] = $p }
  foreach ($p in $all) {
    $key = "$($p.ProcessId)/$($p.CreationDate.Ticks)"
    if ($seen[$key]) { continue }
    $seen[$key] = $true
    $chain = @(); $c = $p
    for ($i = 0; $c -and $i -lt 5; $i++) { $chain += $c.Name; $c = $byId[[int]$c.ParentProcessId] }
    '{0:HH:mm:ss.fff} {1}' -f (Get-Date), ($chain -join ' <- ')
  }
  Start-Sleep -Milliseconds 400
}
```

Codex を起動すると、こういう形になっていました。

```text
codex（画面側の CLI）
codex app-server --listen unix:// --managed-daemon    ← 裏のサーバー
  ├ node_repl.exe（MCP サーバー。ブラウザ操作などに使う）
  ├ codex-code-mode-host.exe
  └ git.exe（何本も）
```

窓が出るのは、この裏のサーバーの子プロセスでした。git が 3 本起動すると窓が 3 枚、4 本なら 4 枚。同じ秒に出る数が揃っていました。短命なプロセスは記録の間隔で取りこぼすので、数は目安です。

なぜ窓になるのかは、推測です。裏のサーバーはコンソールを持っておらず、そこからコンソールアプリを起動すると、Windows は子ごとに新しいコンソールを作ります。Windows Terminal が既定のターミナルなので、それが新しい窓として見える、という読みです。

ChatGPT のデスクトップアプリ側の Codex は、同じように `node_repl` や `git` や `cmd` を起動していましたが、窓は出ませんでした。観察できたのは数回だけです。

![裏の部屋の機械が小包を渡すたびに、表の部屋で空の窓が開く絵](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/03_2026-09-26_codex-empty-windows_illustration-backstage.png)

裏のサーバーは、見えないところで子プロセスを起動し続けています。そのたびに、見える窓が 1 枚ずつ増えていきます。

![裏のサーバーを使う場合と使わない場合の、プロセスの親子関係を並べた図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/04_2026-09-26_codex-empty-windows_fig1.png)

図の左は、裏のサーバーを使う既定の形です。子プロセスがサーバーの下にぶら下がり、窓が出ます。右は、サーバーを使わない形で、同じ子プロセスが Codex 本体の下にぶら下がり、窓は出ません。

## 「無効にする」案は、試す前にやめた

裏のサーバーの子を止めれば窓は消えるはず、と考えて、`node_repl` の MCP と `code_mode_host` の機能を無効にする案を立てました。試す前にやめました。理由は 2 つです。

1 つめは、`node_repl` の設定に `BROWSER_USE_...` という環境変数が並んでいたことです。無効にすると、ブラウザ操作を失う可能性が高いと見ました。設定の名前からの推測です。2 つめは、窓を出していた子の中に `git` がいて、この案では止まらないことです。

## 上流はもう直していた。ただ、安定版にはまだ入っていない

上流の Issue には、同じ症状の報告が並んでいました。9 月 26 日の時点で、どれも open です。

- 起動時に、Windows のバックグラウンドサーバーが見えるコンソールを 2 つ開く: https://github.com/openai/codex/issues/48090
- 起動した直後から、空のコンソールが開く: https://github.com/openai/codex/issues/48114
- メッセージを送るたびに、複数のコンソールが開く: https://github.com/openai/codex/issues/48325
- 起動時や裏の作業のたびに、外部のコンソール窓が出る: https://github.com/openai/codex/issues/48039
- ローカルの作業中に、補助のコンソール窓が一瞬出る（0.157.0）: https://github.com/openai/codex/issues/48152
- 遠隔操作用のサーバーが、ツールと MCP の子プロセスのたびに窓を出す: https://github.com/openai/codex/issues/46949
- サーバーが、フックやシェルコマンドのたびに窓を出す: https://github.com/openai/codex/issues/44768

9 月 25 日に、修正の PR が 2 本マージされています。

- code-mode host を起動するときに、コンソール窓を出さない: https://github.com/openai/codex/pull/48138
- ローカルの Windows 用 MCP サーバーで、コンソール窓を出さない: https://github.com/openai/codex/pull/48238

ただ、この時点の安定版 0.157.1 には、どちらも入っていません。GitHub の比較機能で、マージされたコミットが各リリースのタグに含まれるかを調べました。0.159.0-alpha.4 には 2 本とも入っています（#48138 は alpha.1 以降）。リリース一覧はこちらです: https://github.com/openai/codex/releases

## alpha を入れても、窓は減っただけだった

修正入りの alpha.4 を、試しに入れました。窓は変わりませんでした。バージョンを見ると、裏のサーバーだけ 0.157.0 のままです。サーバーの実体は `CODEX_HOME` の下の `packages/app-server-daemon` に入っていて、CLI を新しくしても入れ替わりません。

```powershell
codex app-server daemon update --from-cli -y   # CLI と同じ版のサーバーに入れ替える
```

入れ替えたあと、`node_repl` と `codex-code-mode-host` の窓は出なくなりました。ところが、`git` の窓が、開いて 1 秒ほどで閉じる形で残りました。ゴーストのように一瞬だけ出る窓です。今回の 2 本の修正の対象ではなさそうです（PR のタイトルからの判断です）。alpha は、そのあと安定版に戻しました。

## 裏のサーバーを使わない、という手があった

窓は裏のサーバーの子が出しているので、サーバーを使わなければ出ないはずです。Codex の `--help` を見ると、`--no-daemon` がありました。共有のバックグラウンドサーバーなしで実行する、というオプションで、すでに動いていても使わない、と説明されています。前の記事に載せたエラーメッセージにも、同じ案内が出ていました。

設定でも同じことができます。`codex features list` に `daemon_auto_start` という項目があり、stable で、既定は true です。

```powershell
codex features disable daemon_auto_start
```

これで `config.toml` に `daemon_auto_start = false` が 1 行入ります。設定ファイルの差分は、この 1 行だけでした。

確かめたのは 3 通りです。Hub から 2 つの契約でそれぞれ Codex を起動したのと、PowerShell から直接起動したの。どれも、窓は 0 件でした。裏のサーバーも起動せず、`node_repl` と `codex-code-mode-host` は Codex 本体の子として動きました。契約ごとの設定フォルダには、many-ai-cli が起動時に設定を同期する仕組みがあり、1 行足しただけで両方に届きました。

ブラウザ操作も試しました。example.com を開いて見出しを読ませたら、「Example Domain」と返ってきました。途中で 1 回、`Capability is not available: visibility` と出て失敗しましたが、次の操作で成功しています。これが設定のせいかどうかは、比べていないので分かりません。

## 注意したいこと

分かっていないことを、先に書いておきます。

- 共有のサーバーを使う機能が、使えなくなる可能性があります。`codex --help` では、`codex agents` が「共有のサーバー上のセッション一覧」と説明されています。試していません
- `~/.codex/config.toml` は、デスクトップアプリと共有しています。アプリへの影響は確認していません。アプリは自前の Codex を持っているので、影響は小さいと見ていますが、推測です
- 観察は、それぞれ起動から数十秒の 1 回です。長く使ったあとの様子は見ていません
- すでに動いているサーバーは、`codex app-server daemon stop` で止めます。止めたあとも、自動更新のループ（`app-server daemon pid-update-loop`）のプロセスが残ることがありました。自分の環境では手で終了しました

上流の修正が安定版に入ったら、設定は戻すつもりです。

## 学んだこと

- 欠陥が見つかることと、それが今の症状の原因であることは、別の話。観察を 1 つ取ってから直す
- 原因を調べる前に、当日の作業メモを読む。答えが自分の記録にあった
- 上流の修正を待つときも、「何が直って、どの版に入ったか」は、Issue と PR とタグで機械的に確かめられる
- コードに手を入れる前に、`--help` と `features list` を見る。設定 1 行で済む回避が、いちばん安く戻せる

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

自分のコードを疑って、19 ファイルを書き換えて、全部戻す。今回いちばん遠回りしたのは、そこでした。

窓は、今は出ません。ただ、これは上流の修正が入るまでのつなぎだと思っています。入ったら、設定を戻します。

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
