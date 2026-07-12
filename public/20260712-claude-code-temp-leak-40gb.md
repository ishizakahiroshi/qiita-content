---
title: Cドライブの空きが謎に減るので Claude Code に調べさせたら、犯人は Claude Code 自身だった（%TEMP% に 37.8GB）
tags:
  - Windows
  - PowerShell
  - トラブルシューティング
  - Bun
  - ClaudeCode
private: false
updated_at: '2026-07-12T14:04:49+09:00'
id: 2bc449a68951386ac48e
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-12_claude-temp-leak_hero.png)

255GB の C ドライブ、空きが 50.5GB。定期的に一時ファイルは消しているのに、気づくとまた減っている。そういう状態が続いていたので、重い腰を上げて調べました。

調べさせた相手は Claude Code です。そして 40 分後、Claude Code が突き止めた犯人は Claude Code 自身でした。自分で自分を検挙する AI を眺めるのは、なかなか味わい深い時間でした。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-12_claude-temp-leak_infographic.png)

## 結論から。Windows で C ドライブが減り続けたら、まずここを見る

`%TEMP%`（`C:\Users\<名前>\AppData\Local\Temp`）の直下に、こういう名前のファイルが大量にないか確認します。

```
.{16桁のhex}-00000000.dll   （1 個 1.77MB）
.{16桁のhex}-00000001.node  （1 個 0.49MB）
```

PowerShell ならこの 1 行で総量が分かります。

```powershell
Get-ChildItem $env:TEMP -File -Force | Where-Object Name -match '^\.[0-9a-f]{16}-' |
  Measure-Object Length -Sum | ForEach-Object { "{0} 個 / {1:N1} GB" -f $_.Count, ($_.Sum/1GB) }
```

手元の実測はこうでした。

```
33,966 個 / 37.8 GB
```

これらは Claude Code（Windows ネイティブ版）が起動のたびに展開するネイティブモジュールの置き土産です。消しても増えるのは当然で、使うたびに数 MB ずつ確実に増える仕組みになっていました。以下、突き止めるまでの調査ログです。

## 誰が 40GB 使っているのか分からない

まず全体の内訳から。フォルダごとのサイズを PowerShell で集計していくと、ユーザーフォルダ配下で AppData が 91.9GB。その中で飛び抜けていたのが Temp の 40.2GB でした。

普段この場所は数 GB もあれば多いほうです。中を見ると、直下にファイルが 34,181 個。1 個ずつは 2MB 弱で小さいのに、数で押し切られていました。

面白いのはここからで、名前のパターンで集計すると、実質 2 種類しかなかった。

```
.{16桁hex}-00000000.dll   17,202 個  29.65 GB
.{16桁hex}-00000001.node  16,764 個   8.16 GB
```

さらにサイズでグループ化すると、dll は 17,199 個が全部 1.77MB、node は 16,693 個が全部 0.49MB。つまり「同じ 2 つのファイルが、名前だけ変えて 1 万 7 千回コピーされている」状態です。しかも dll と node の個数がほぼ同じ。1 回の何かにつき 1 ペアずつ増えている、と読めます。

## 5 日で 40GB。しかも 1 日に集中していた

タイムスタンプを日別に集計したら、こうなりました。

```
07/07      18 個   0.03 GB
07/08      14 個   0.02 GB
07/09  31,678 個  35.26 GB   ← ここ
07/10   2,172 個   2.40 GB
07/11      50 個   0.06 GB
07/12      34 個   0.04 GB
```

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-12_claude-temp-leak_fig1.png)

上のグラフのとおり、7/9 の 1 日だけで 31,678 個・35.26GB。全体の 9 割超がこの日に生まれています。7/7 以前のファイルが 1 個もないのは、直前に自分で Temp を掃除していたから。つまり「毎回消してるのに増える」の実態は、恒常的に毎日数十個ずつ漏れる分に加えて、7/9 に何かが暴走した分が乗っていた、という二層構造でした。

7/9 を時間帯別に割ると、13 時から 23 時まで毎時 2,600〜3,000 個。約 2.4 秒に 1 ファイルのペースが 11 時間続いています。人間の操作ではありえない。何かのプロセスがループで走っていた形跡です。

## ファイルの中身を読んだら、正体が並んでいた

拡張子が .node（Node.js のネイティブアドオン）なので、この時点で Node 系ツールの仕業までは絞れます。ただ「どれが」が分からない。そこでバイナリの中の文字列を漁りました。

```powershell
rg -a -o -N "[A-Za-z0-9_-]{4,}\.pdb" <対象ファイル>
```

出てきたのは 3 つ。

- `napi_keyring.pdb`（OS の資格情報ストアにアクセスするモジュール）
- `ghostty`（ターミナルエミュレーションのライブラリ）
- `image_processor.pdb`（画像処理。スクリーンショットを読ませたときに使われる）

そして決定打はこれです。今まさに実行中のプロセスが、Temp のこのファイルをロードしていないかを見る。

```powershell
Get-Process | ForEach-Object { $p = $_; try { $p.Modules |
  Where-Object { $_.FileName -like "$env:TEMP\.*" } |
  ForEach-Object { "{0} (PID {1}) → {2}" -f $p.ProcessName, $p.Id, (Split-Path $_.FileName -Leaf) } } catch {} }
```

```
claude (PID 7400)  → .{hex}-0.node
claude (PID 22672) → .{hex}-0.node
claude (PID 23440) → .{hex}-0.node
```

実行中の claude プロセスが 3 つ、Temp の .node を現にロードしていました。しかもうち 1 つは、この調査をやらせているセッション自身のもの。生成時刻もセッションの起動記録と秒単位で一致しました（11:54:00 にファイル生成、11:54:02 にセッション開始、という並びがいくつも取れた）。ここで容疑者は確定です。お前だったのか。

## 仕組み。Bun 製バイナリと Windows の相性問題

Claude Code の Windows ネイティブ版は、Bun でコンパイルされた 238MB の単一 exe です（バイナリ内に `Bun v1.4.0` の文字列が埋まっています）。単一ファイル配布の実行形式は、内蔵しているネイティブモジュール（今回の keyring、ghostty、image_processor）をそのままではロードできないので、起動時に一時フォルダへ書き出してから読み込みます。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-12_claude-temp-leak_fig2.png)

図にすると単純で、「起動のたびに %TEMP% へ 2 ファイル展開 → ロード → 終了時に後始末されない」。これだけです。Windows はロード中の DLL を削除できない制約があるうえ、AI エージェントのセッションは kill で終わることも多い。行儀よく後始末が走る余地がそもそも少ない環境で、展開だけが積み上がっていきます。1 起動あたり約 2.3MB。塵も積もれば 37.8GB でした。

7/9 の暴走のほうは、バージョン履歴と重ねると輪郭が出ました。Claude Code は自動更新で、手元には直近 3 バージョンが残っています。

```
2.1.205  07/09 09:14 導入  → その日 31,678 個（毎時 ~2,900）
2.1.206  07/10 09:02 導入  → 2,172 個に急減
2.1.207  07/11 10:03 導入  → 日 30〜50 個の通常ペース
```

特定バージョンの日だけ内部の子プロセス起動が暴れ、翌日の更新で収まった、と読んでいます。ここは中の実装を確認したわけではないので推測です。ただ、対話セッションの記録は当日 27 件しかなく、1.5 万起動の説明が付くのはセッション外の内部プロセスしかない。なお、この Temp への置き土産自体は手元だけの現象ではなく、GitHub にも同種の報告が複数上がっている既知の問題です。

## 対処。根治はできないので、安全に消す

ツール側の修正待ちなので、根治はできません。当面は溜まった分を消すだけです。該当パターンだけを対象にすれば安全で、実行中のプロセスが使っているファイルは削除に失敗してスキップされるだけなので、雑に流して問題ありません。

```powershell
Get-ChildItem $env:TEMP -File -Force | Where-Object Name -match '^\.[0-9a-f]{16}-' |
  Remove-Item -Force -ErrorAction SilentlyContinue
```

あとは Windows のストレージセンサー（設定 → システム → 記憶域）を有効にしておけば、Temp は定期的に自動で掃除されます。「起動ごとに数 MB」は今後も続くので、手動で消す派の人も、この場所だけは定期便に入れておくのがよさそうです。

## 学んだこと

- ディスク逼迫の調査は「大きいファイル探し」より先に「同一サイズ × 大量個数」を疑う。名前パターンとサイズでグループ化すると一発で浮く
- タイムスタンプは口ほどに物を言う。日別・時間帯別に割るだけで「恒常リーク」と「単発の暴走」を分離できた
- 拡張子で当たりを付けたら、バイナリ内の文字列（.pdb 名）とロード中モジュールの実測で確定できる。推測で止めない
- 一時フォルダは「消したら終わり」ではなく「増える速度」を見る場所だった

Claude Code には毎日世話になっていて、これからも使います。ただ、自分の尻拭いを自分に調査させて、淡々と「犯人は私です」と報告してくる様子には、なんとも言えない気持ちになりました。悪気はないんです。後始末ができないだけで。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
