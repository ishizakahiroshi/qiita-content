---
title: AI エージェント使用中の Defender 負荷を測る。会話ログの検査時間を切り分けた手順
tags:
  - Windows
  - PowerShell
  - CodexCLI
private: false
updated_at: '2026-10-18T05:16:20+09:00'
id: ea4770167cf82326fe93
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![複数の端末のそばで記録を検査する見張り番](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/01_2026-10-18_defender-ai-agents_hero.png)

Defender が何を検査しているかは、管理者の PowerShell で 2 分間記録すると切り分けられます。
自分の PC では git よりも AI の会話記録とログが目立ち、対象の 8 フォルダの検査時間は 6.93 秒から 0.03 秒になりました。
ただし、除外は保護を弱めます。プログラムを含む場所は外さず、Defender 全体の CPU が下がったという話とも分けて考えます。

![測る、書き足す記録を見つける、範囲を絞る、同じ場所を比較する](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/02_2026-10-18_defender-ai-agents_infographic.png)

時間に合わせて、読み方を選べます。

- 少し忙しい人向け：[5分で読める要約版（個人サイト）](https://ishizakahiroshi.com/articles/2026/2026-10-18_defender-ai-agents/)
- もっと深掘りしたい人は、このまま本文へどうぞ。

Codex CLI、Claude Code、grok を Windows で並べていると、Antimalware Service Executable が数%から十数%で動き続けていました。不毛だな、と思いました。

git のデータが頻繁に書き換わるせいかと思いましたが、測ると目立つ場所は別でした。

## many-ai-cli について

自作で [many-ai-cli](https://github.com/ishizakahiroshi/many-ai-cli) という AIツール / Webダッシュボードを作っています。複数の AI コーディング CLI を並列で走らせ、承認をブラウザ 1 タブに集約。スマホからでも。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/many-ai-cli

README の入れ方は次のとおりです。使う AI CLI は別にインストールしておきます。

```powershell
pnpm add -g many-ai-cli
many-ai-cli setup
```

Windows ではデスクトップの「MANY-AI-CLI」を開き、トレイメニューの「Hub を開く」で起動します。止めるときは「Hub を停止」です。setup はサインイン時にトレイを起動するショートカットも作ります。

macOS・Linux では「Many AI Hub Start」「Many AI Hub Stop」を使います。Start と一緒に開くコンソールは Hub 本体なので、閉じると停止します。全 OS 共通で、端末からの起動は `many-ai-cli serve --open`、停止は別の端末で `many-ai-cli stop` です。README ではネイティブの macOS・Linux は実機での検証が十分ではないとされています。

## まず、設定を変えずに 2 分間測る

使ったのは Microsoft Defender Antivirus の Performance Analyzer です。**PowerShell を管理者として開きます。** AI が普段どおり動いているときに、以下を順番に実行します。

```powershell
$stamp = Get-Date -Format 'yyyyMMdd-HHmmss'
$etl = Join-Path $env:TEMP "mpperf-$stamp.etl"
New-MpPerformanceRecording -RecordTo $etl -Seconds 120
```

`-Seconds 120` で自動停止します。省略時は Enter で止めます。記録終了後、次を実行します。最初は 2 本のコマンドを区切りなしで 1 行に貼り、「位置指定パラメーターが見つかりません」と失敗しました。順番に実行してください。

```powershell
Get-MpPerformanceReport -Path $etl -TopProcesses 10 -TopExtensions 10 -TopFiles 20
```

- `TopFiles`: どのファイルの検査に時間がかかったか
- `TopProcesses`: どのプログラムに関係するファイル検査が目立つか
- `TopExtensions`: どの拡張子の検査に時間がかかったか

検査時間の大きい順の一覧です。回数と時間を一緒に見ます。プロセス別の検査時間と、そのプログラム自身の CPU 使用率は別の値です。

コマンド名とパラメーターは [New-MpPerformanceRecording](https://learn.microsoft.com/en-us/powershell/module/defenderperformance/new-mpperformancerecording) と [Get-MpPerformanceReport](https://learn.microsoft.com/en-us/powershell/module/defenderperformance/get-mpperformancereport) で確認しました。Microsoft も、この機能は除外を提案するものではなく、除外は保護を弱めるので慎重に判断するよう注意しています。

![まず記録を集めてから原因を探す](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/03_2026-10-18_defender-ai-agents_illustration-measure.png)

集計用に JSON がほしい場合は、同じ記録から書き出せます。記録を取り直す必要はありません。

```powershell
$json = [IO.Path]::ChangeExtension($etl, 'json')
Get-MpPerformanceReport -Path $etl -TopProcesses 10 -TopExtensions 10 -TopFiles 20 -Raw |
    ConvertTo-Json -Depth 8 |
    Set-Content -LiteralPath $json -Encoding UTF8
```

`-Raw` の時間は 100 ナノ秒単位です。39004334 なら約 3.9 秒。通常表示とも照合しました。上位一覧の合計は全体の合計とは限りません。記録には個人情報を含むパスがあるので、そのまま公開しないでください。

## git を疑ったら、会話記録が上位にいた

測ったのは Windows 11 Pro 10.0.26200 の PC 1 台です。2026 年 10 月 18 日の深夜、日本時間 1:42:31〜1:44:32 の記録では、2,814 回のファイル検査に合計 10.9 秒かかっていました。

| 場所 | 検査時間 | 回数 |
|---|---:|---:|
| grok の docs と logs | 4.14 秒 | 171 |
| Codex の sessions | 2.79 秒 | 311 |
| many-ai-cli の作業用フォルダ | 1.12 秒 | 252 |
| ユーザーフォルダ | 1.10 秒 | 556 |
| AI のフォルダのうち除外していない場所 | 0.42 秒 | 129 |
| 作業ツリー | 0.33 秒 | 542 |
| その他 | 1.04 秒 | 未集計 |

![対策前の場所ごとの検査時間](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/04_2026-10-18_defender-ai-agents_fig-before-v2.png)

あとで除外した grok と Codex の 8 フォルダだけで、6.93 秒、482 回。合計 10.9 秒の約 6 割でした。Claude Code の会話記録は many-ai-cli の作業用フォルダに含まれますが、その分だけを取り出して測ってはいません。

git はプロセス別の上位 10 に入っていませんでした。10 位の claude.exe が 0.2 秒なので、それ未満です。最初の見立ては外れました。ただし、ここで見ているのはファイル検査です。プロセス起動に伴う監視まで含め、git の影響が一切無いと証明したわけではありません。

Codex は rollout の `.jsonl` に会話記録を 1 行ずつ書き足します。手元では、その変更に伴って同じ記録が繰り返し検査されていました。grok の docs と logs も頻繁に書き換わっていました。ソースコードの差分より、作業中ずっと増える記録のほうが目立っていたのです。

![記録が増えるたびに同じ束を検査する](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/05_2026-10-18_defender-ai-agents_illustration-append.png)

## 除外する前に、フォルダの中身を数える

**プログラムが入っているフォルダは外しません。** フォルダ名が logs や sessions だから安全、とも決めません。実際の保存先と、中にあるファイルの両方を確かめます。

自分が対象にしたのは、2 つの CODEX_HOME にある sessions と、3 つの grok のホームにある docs・logs の計 8 フォルダです。同じ 8 個を作る必要はありません。

OpenAI の[設定説明](https://learn.chatgpt.com/docs/config-file/config-advanced)では、CODEX_HOME の既定は `~/.codex` です。ただし以下では既定パスを決め打ちせず、**記録に実際に出た sessions フォルダ**を入力します。grok の既定パスも断定しません。Claude Code の会話記録は今回除外しておらず、効果も測っていません。

次は確認だけです。候補のフォルダのフルパスを入力します。ワイルドカードや環境変数の表記は使いません。

```powershell
$target = Read-Host '記録に出た対象フォルダのフルパス'
$item = Get-Item -LiteralPath $target -ErrorAction Stop
if (-not $item.PSIsContainer) { throw 'フォルダを指定してください' }
$target = $item.FullName
$programExtensions = '.exe', '.dll', '.cmd', '.bat', '.js', '.mjs', '.ps1'
$files = @(Get-ChildItem -LiteralPath $target -File -Recurse -Force -ErrorAction Stop)
$programs = @($files | Where-Object { $_.Extension -in $programExtensions })
$programs | Select-Object FullName
"対象: $target"
"確認したファイル数: $($files.Count)"
"指定した拡張子に一致: $($programs.Count)"
```

手元の 8 フォルダでは、これらの拡張子は 0 本でした。ただし、この一覧は実行形式のすべてではなく、拡張子の偽装や読み込ませる内容の安全性も判定しません。**0 本は調査の入口で、安全証明ではない**と考えます。アクセスエラーが出たときも、調べられなかったものを 0 本と扱わず止めます。

道具のホーム全体には exe が 64 本、dll が 109 本、js が 630 本ありました。文章の記録だけと確認できた場所に限定しました。

## 登録したら、必ず一覧を読み返す

除外を選ぶ場合は、そのフォルダのウイルス検査が行われなくなることを理解したうえで進めます。会社管理の PC なら管理者の方針に従ってください。以下は**管理者の PowerShell**で、直前に確認した `$target` を使う例です。

```powershell
if (-not $target -or $null -eq $programs) { throw '直前の確認を実施してください' }
if ($programs.Count -gt 0) { throw '実行形式が見つかったため除外を中止します' }
Add-MpPreference -ExclusionPath $target
(Get-MpPreference).ExclusionPath
```

[Add-MpPreference](https://learn.microsoft.com/en-us/powershell/module/defender/add-mppreference) は既存の一覧に追加する操作です。一覧の表示までをひと続きの確認にします。自分の環境では管理者でないと一覧が見えず、管理者が必要だという表示になりました。

長い 1 行を貼る失敗もしました。折り返し位置に空白が入り、2 件が壊れたパスで登録されました。Add-MpPreference はエラーを出さず、一覧で気づきました。

壊れた 2 件だけを消し、短いスクリプトで登録し直しました。パスを変数に分け、一覧を読み返す。他の既存の除外は残します。

![長いパスを分けて登録内容を確認する](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/06_2026-10-18_defender-ai-agents_illustration-path-corrected.png)

取り消す場合も、同じパスを指定します。フォルダや記録そのものを削除する操作ではありません。

```powershell
Remove-MpPreference -ExclusionPath $target
(Get-MpPreference).ExclusionPath
```

## 減ったのは、対象の場所の検査時間

同じ夜の 2:00:01〜2:02:01 に、もう一度 2 分間記録しました。

| 項目（2 分あたり） | 対策前 | 対策後 |
|---|---:|---:|
| 外した 8 フォルダ | 6.93 秒、482 回 | 0.03 秒、302 回 |
| grok が読み書きしたファイル | 3.95 秒 | 0.80 秒 |
| Codex が読み書きしたファイル | 2.93 秒 | 1.89 秒 |
| 記録全体 | 10.9 秒 | 50.2 秒 |

**記録全体が増えたのは、対策後の記録中に Go のテストが走っていたためです。** モジュールキャッシュで 24.51 秒、2,169 回。ビルドキャッシュで 4.62 秒、520 回ありました。誰がテストを動かしたかは分かっていません。

![対象8フォルダの前後比較。記録全体には別のGoテスト負荷が混在](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/07_2026-10-18_defender-ai-agents_fig-comparison.png)

Go のキャッシュは除外していません。ネットから取得した部品やツール本体が入り、「文章の記録だけ」という前提が成り立たないためです。

## それでも Defender の CPU は残った

対策後もタスクマネージャーの数字はゼロになりませんでした。その夜は 4.5〜19% の間で上下し、2:58 の 30 秒平均は PC 全体に対して 5.7% でした。対象とは別の何かを検査している分が残り、その正体までは突き止めていません。

調査で大量のファイルを読む負荷も混ざりました。調査中は 20%、止めると 8.6% でした。

![対象の記録以外にも検査するものが残る](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/08_2026-10-18_defender-ai-agents_illustration-other.png)

今回の比較は **PC 1 台、対策前後に 2 分を 1 回ずつ、計 2 回だけ**です。同じ処理を同じ順序で再生したベンチマークではありません。対象の場所の検査時間が減ったことは観察できましたが、PC 全体の高速化率や、ほかの環境での効果は言えません。

## 範囲を絞っても、危険は残る

外したのは、記録用の 8 フォルダの中です。それ以外の場所の検査は残ります。除外していない場所へファイルをコピーすれば、そのコピー先は検査対象になります。ただし、すべての悪意あるファイルが必ず検出されるという意味ではありません。

除外先に悪いプログラムが置かれれば、その場所のファイル検査による保護は期待できません。文章だけで、調べた実行形式は 0 本だったため、範囲を絞れば危険は小さいと判断しました。それでも更新で用途が変わる可能性はあります。他の入口の検査も、必ず検出できる保証ではありません。

![外した場所と外さなかった場所の境界](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/09_2026-10-18_defender-ai-agents_fig-boundary.png)

特に、codex.exe や grok.exe の**プロセス除外は使っていません**。プロセスが開くファイルにまで対象が広がり、リポジトリへの出力も検査されなくなるおそれがあるためです。ホーム全体やキャッシュ全体を除外する方法も、この記事では勧めません。

プロンプトインジェクションは別に考えます。会話記録に AI への悪意ある指示が入る問題を、このフォルダ除外で解決することはできません。Defender のファイル検査を、AI が読んだ文章に従うかどうかの承認機構としては使えません。外部の文章を読ませる作業では、削除や送信などの危険な操作の前に人が確認することが重要です。操作を自動許可すれば、その確認の機会は減ります。

![文章の置き場とプログラムの置き場を分けて考える](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/10_2026-10-18_defender-ai-agents_illustration-boundary.png)

## 公式の説明と、ほかの人の報告

以下は 2026 年 10 月 18 日（日本時間）の確認です。Issue は利用者の報告で、公式見解とは分けます。

Anthropic の [Troubleshooting](https://code.claude.com/docs/en/troubleshooting) と [Advanced setup](https://code.claude.com/docs/en/setup) には、確認した範囲で Defender・ウイルス対策ソフトの項目はありませんでした。一方、公開 Issue には次の報告があります。

- [claude-code #94478](https://github.com/anthropics/claude-code/issues/94478): デスクトップアプリが git を高頻度で起動する報告。open、コメント 11 件
- [#101009](https://github.com/anthropics/claude-code/issues/101009): PowerShell の解析と実行で 2 回起動する報告。open、コメント 1 件
- [#100899](https://github.com/anthropics/claude-code/issues/100899): 画像貼り付け時の PowerShell 起動と、ウイルス対策ソフトによる遅延の報告。open、コメント 1 件。これは Defender そのものの実測ではありません

OpenAI の Windows 向け説明にも、確認した範囲で今回の会話記録への対処はありませんでした。ただし、[codex doctor のソース](https://github.com/openai/codex/blob/ce0323f2f2d3b25e82806cd555237da3db4e1897/codex-rs/cli/src/doctor/security.rs)には、Defender を検出すると本体と補助プログラムを証明書または実行ファイルのパスで除外するよう案内する処理があります。文面からプロセス除外と決めつけることはできません。

[Codex #45103](https://github.com/openai/codex/issues/45103) は、その診断が除外の有無を実際には検査せず警告する点を指摘しています。open、コメント 3 件です。投稿者は除外を変更して試したのではなく、ソースの制御フローから指摘しています。

高 CPU の報告は [#30527](https://github.com/openai/codex/issues/30527) がコメント 19 件、[#33875](https://github.com/openai/codex/issues/33875) が 18 件、空の .git を繰り返し調べる [#29911](https://github.com/openai/codex/issues/29911) が 11 件で、いずれも open でした。デスクトップアプリの報告と、自分の CLI の会話記録の話を同じ原因としてまとめないようにしています。

開発時の検査との付き合い方という意味で近い公式機能は、Microsoft の [Dev Drive のパフォーマンスモード](https://learn.microsoft.com/en-us/defender-endpoint/microsoft-defender-endpoint-antivirus-performance-mode)です。検査を無くすのでなく、ファイルを開いたあとへ遅らせます。ただし公式ページは、MsMpEng の高 CPU 問題に適用する機能ではないとも注意しています。自分の PC には Dev Drive が無く、試していません。

今回の「実測した場所のうち文章の記録だけに絞る」方法は、各社が AI の会話ログ向けに示した公式手順ではありません。

## many-ai-cli はこんなときに刺さります

- Claude Code・Codex・Copilot・Cursor・Grok を並べ、状態を 1 画面で見たい人
- 各 CLI の承認待ちを 1 つのタブでまとめて扱いたい人
- 離席中もスマホから承認したい人

試すときは `pnpm add -g many-ai-cli`、続けて `many-ai-cli setup` です。スマホからの接続には README の接続設定が必要です。

- 紹介ページ: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/many-ai-cli

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## あわせて読みたい

- [Cドライブの空きが謎に減るので Claude Code に調べさせたら、犯人は Claude Code 自身だった（%TEMP% に 37.8GB）](https://qiita.com/ishizakahiroshi/items/2bc449a68951386ac48e): AI の道具が Windows 上でファイルを作り続ける、という点でつながる話です。
- [Codex CLI 0.157.0 が Windows で「path must be shorter than SUN_LEN」と言って起動しない。原因は CODEX_HOME のパスの長さだった](https://qiita.com/ishizakahiroshi/items/e1185939731120eb54dd): CODEX_HOME を分けて使うとき、保存先が動作に影響した話です。
- [AI CLI を 7 本並べたら、次に要るのは「契約」と「引き継ぎ」と「委譲」だった。many-ai-cli v0.8.0](https://qiita.com/ishizakahiroshi/items/49f6a29a43bf64168c1a): 今回 AI を並べていた道具の背景です。

## おわりに

測ったら会話記録が目立ち、そこだけを見直しました。Defender は今も別の仕事をしています。

まず 2 分間記録し、何を検査しているか知る。外す場所を考えるのはそのあとです。

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
