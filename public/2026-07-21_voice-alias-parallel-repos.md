---
title: 並行リポ29個を音声で回す。呼称索引という小さな台帳を1個足しただけの話
tags:
  - AI
  - CLI
  - 音声入力
  - 個人開発
  - ワークフロー
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

![音声入力で複数リポを回す模式イラスト](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-21_voice-alias-parallel-repos/hero.png)

git のローカルクローン数を数えたら 29 個ありました。個人 OSS が 19、個人アプリが 6、業務案件が 4。全部を同時にコミットするわけではないけれど、直近 30 日で 1 回でも触ったやつだけで 29。これを音声入力で扱おうとして、まず名前で詰まりました。

AI CLI に「まなびまっぷの続きやって」と言うと、AI は当然 `manabi-map` を知らないので「どのリポですか」と聞き返してくる。「メニーエーアイシーエルアイ」なんて発音、そもそも音声入力が高確率で誤変換します。長い英字リポ名と音声入力の相性は、絶望的に悪い。

この記事はそれを一晩で回した仕組みの話です。新しく作ったのは CSV ファイル 2 個と、既存 skill への追記だけ。仕組みは軽いけれど、判断の勘所は書いておく価値がある気がしたので残します。

なお、後半に出てくる「今日やる 1 個」を選ばせる部分は、自作の [docsweep](https://github.com/ishizakahiroshi/docsweep) という OSS を前提にしています。AI コーディングツール（Claude Code / Codex 等）が量産する `plan_*.md` / `bugfix_*.md` / `pending_*.md` の蓄積と陳腐化を、H1 ステータスラベルと OKF frontmatter で機械的に処理する CLI + Web UI です。同じ悩みの方は `pip install docsweep` で入ります。リポジトリはこちら（Star をいただけると励みになります）: [github.com/ishizakahiroshi/docsweep](https://github.com/ishizakahiroshi/docsweep)

## 音声で呼びたいのに、綴りが長すぎる

音声入力で AI CLI を使い始めて数日で気付いたのは、**リポ名は「言うため」に設計されていない**ということでした。`many-ai-cli` `claude-code-context-diet` `tab-title-prefix` `vite-plugin-git-version`。文字で読む分には全く問題ない。むしろ検索性を上げるためにわざと固有性を強めた命名にしています。

でも音声で「クロード コード コンテキスト ダイエット」と喋ると、認識結果は毎回ブレる。「クラウド」になったり「クロート」になったり。カタカナと英字が混ざるとさらに歪む。

この時点で選択肢は 3 つありました。

- リポ名を短く付け直す（既存の GitHub URL / npm / import path が全部壊れる。論外）
- 音声入力を諦めてキーボードで書く（そもそも音声で速くしたかった話が消える）
- 呼称索引を挟む（発話 → 正式リポ名（canonical slug）への変換の中間層を作る）

3 つ目です。IME の辞書登録に近い発想ですが、AI CLI から使えるように独立ファイルに置きます。

## 呼称索引パターンは実は既に持っていた

前提として、自分は個人ナレッジを「人・会社・サーバー・システム」の 4 枚の薄い CSV（`kb`）で管理していて、AI には話題に応じて数行だけ引かせています。詳細は前作の Zenn 記事に書きました（[組織の暗黙知は 4 枚の CSV で足りている](https://zenn.dev/ishizakahiroshi/articles/20260708-thin-master-data-for-teams) / [5 枚目の呼称索引で音声入力の表記ゆれを吸収する](https://zenn.dev/ishizakahiroshi/articles/20260711-alias-index-for-voice-input)）。この記事はその **5 枚目の呼称索引を dev リポにも拡張した話** です。

面白かったのは、これと同じ問題を **10 日ほど前に別文脈で解いていた** ことです。会社名や社内システム名が音声入力で毎回崩れるので、`kb-alias-add` という小さな skill を作って、CSV ファイル 1 個で吸収する仕組みができていました。

構造は極小です。

```
alias,ref_type,ref_id,ref_name,note
わかば,application,x001,若葉商事(受注管理),音声入力の誤変換
HR,application,x002,ハーモニー(人事),社内略称
```

「別名 → 参照先」の対応表を追記していくだけ。skill は 1 行足す薄いラッパーです。台帳本体（applications / people / servers 等）を汚さず、索引側で吸収する。

これに `ref_type=project` を追加するだけで、そのまま dev リポの呼称ゆれ吸収に流用できました。**新しく設計するのではなく、既に動いている仕組みに 1 型を足す** のが最短でした。

## 3 層 + 実績 CSV の役割分担

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-21_voice-alias-parallel-repos/infographic.png)

構成はこうです。

- **Google Calendar** = 時間ブロック。「今日 14:00-17:00 は many-ai-cli」の粒度でざっくり。イベントタイトル頭に `[many-ai-cli]` のようにスラッグを前置
- **docsweep** = 個別タスク。`plan_*.md` / `bugfix_*.md` / `pending_*.md` を各リポ配下に置く。`docsweep brief` で今日の 1 個を断定
- **kb-local/aliases.csv + projects.csv** = 呼称索引。音声入力の誤変換を正式リポ名に変換
- **team-log.csv** = 実績（後述）

docsweep は自作の OSS で、`plan_*.md` に `due:` を書いて「今日やる 1 個」を選ばせるツールです。これ自体の紹介は本題ではないので、要は「タスク台帳の md ベース版」だと思ってください。

Calendar は「時間ブロック」でしかありません。ここが後で効いてきます。

## 音声から正式リポ名への 2 段確認

呼称解決の流れは 2 段構えにしました。

1. 音声「まなびまっぷの続きやって」を受け取る
2. `kb-local/aliases.csv` を grep。alias 列に一致があれば `ref_id`（= 正式リポ名）に解決してそのまま実行
3. 一致が無ければ `projects.csv` の `name` にファジーマッチ（部分一致・カタカナ↔英字変換）
4. 単一候補が見つかったら「**manabi-map ですよね?(Y/N)**」を 1 問
5. Yes なら実行 + 続けて「**kb-local に登録します?(Y/N)**」で追記まで一気通貫

一気に追記しないのがポイントです。人間側で「今回は違うプロジェクトだった」と却下できる余地を残しておく。一度登録されると次回から聞かれなくなるので、誤登録は面倒だからです。

![音声入力から正式リポ名までの分岐フローチャート](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-21_voice-alias-parallel-repos/fig-1.png)

これで運用開始時点で `aliases.csv` は空でもよくなりました。使いながら誤変換に遭遇したときに 1 行ずつ足していく。事前に全 29 リポの別名を 3 パターンずつ登録する、みたいな重い初期作業はいらない。

## 「秘匿度」で分けたら過剰警戒だった話

小さな失敗も書いておきます。台帳の置き場所を決めるとき、こう提案してしまいました。

> 「会社案件のリポ名は顧客名を含むかもしれないので、公開 kb と別の非公開 kb-local に分けましょう」

これは半分正しくて半分間違いでした。**"秘匿度" と "共有範囲" は別の軸**です。

自分の環境の kb は Nextcloud 経由で既に同僚と共有されていました。つまり非公開なのは間違いないけれど、共有範囲は「自分だけ」ではない。ここに team-log.csv を置いても、共有相手にとっては既存の共有物が 1 個増えるだけで、新規の情報漏洩リスクは無かったのです。

判断基準として書き直すとこうなります。

- 秘匿度: 「誰にも見せたくないか」
- 共有範囲: 「誰まで見せる想定か」

**分離の物差しは共有範囲の方が実用的**でした。秘匿度で切ると「じゃあ会社案件全部 local へ」となって、結果的に共有相手が読めなくなる。共有範囲で切ると「同僚まで見せる CSV は kb に、自分だけの CSV は kb-local に」と自然に落ちる。

同じ間違いをしそうな人に一言だけ残すなら、**"共有先の実態を先に 1 問確認する"** が全てだと思います。

## xlsx を選ばなかった理由

実績記録の CSV は当初「Excel の方が見やすいのでは」と言われました。妥当な直感です。ただ AI で読み書きする前提だと xlsx には地雷が 2 つあります。

1. **Excel で開いてる間はファイルロック** で AI が書き込めない。「今日 acme 6 時間」と言った瞬間に「Excel 閉じてから言ってください」になる
2. **バイナリなので git diff も Nextcloud のバージョン差分も追えない**

CSV なら Excel でダブルクリックすればカラム表示で開けるし、フィルタもソートもピボットも普通に使えます。違いは「保存時に xlsx に変換しないよう気を付ける」だけ。それだけなら CSV の方が総合的に軽い。

もうひとつ、CSV を選んだ理由に「自分がその道具を作っている最中だから」というのもあります。並行して [PlainSheet](https://github.com/ishizakahiroshi/PlainSheet) というローカル完結の CSV / TSV / Markdown Table エディタを開発中で、Excel を開かなくても CSV をカラム表示で編集できる環境を自前で整えつつあります。作った経緯は note に書きました（[PlainSheet を作りはじめた話](https://note.com/ishizakahiroshi/n/n360ca908bd4a)）。CSV 正本で運用するなら、Excel に依存しないビューア側も自分の手の届く範囲に置いておきたい、という判断です。

`team-log.csv` は long 形式にしました。

```
date, owner, project, hours, note
2026-07-20, ishiz, acme, 6, 
2026-07-20, ishiz, many-ai-cli, 2, v0.4 の作業
```

1 行 = 1 日 × 1 人 × 1 案件。空欄は「行を書かない」で表現。現行の運用シートは wide 形式（列 = 日付）で人間目視には優れるものの、AI で行追加するには列インデックスを都度計算する必要があり、それが地味に厳しい。

## 集計は AI に頼めばいい

wide 形式を捨てて困るのは「月次集計をどう出すか」ですが、これは AI に頼めば解決します。

```
先月の案件別集計出して
```

pandas か PowerShell の `Import-Csv | Group-Object` で 3 行のスクリプトが走って、Markdown 表で返ってくる。会社報告用に貼るならそのまま貼れる。可視化が要るなら「HTML 表で出して」と言えば表になる。

**"集計テーブルを人手でメンテする" というタスクを丸ごと消せた** のが大きくて、これはたぶん xlsx 直接運用よりよほど楽です。

![キーボードを打たずに口頭で集計依頼、AI が pivot して表を返す絵](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-21_voice-alias-parallel-repos/illustration-1.png)

もちろん限界もあります。リアルタイムのダッシュボードとして常時見たいならピボットや BI ツールが向く。他人に触ってもらう集計テンプレなら Excel が早い。**個人が月次で振り返る用途なら AI 集計で十分**、という切り分けです。

## 完璧に揃えず「使いながら育てる」

最後にひとつだけ、運用設計の姿勢の話です。

呼称索引 `aliases.csv` は今日時点で **空のまま** 運用開始しました。29 リポぶんの別名を予測して事前投入することもできたけれど、それはやらなかった。

理由は単純で、**実際に自分が誤変換する呼び方は使ってみないと分からない** からです。「マナビマップ」と喋るのか「まなびまっぷ」と喋るのか、「メニーエーアイ」と略すのか「メニエーアイシーエルアイ」と全部言うのか、これは机上で決められません。使い始めて 1 週間で自然に必要な 10 数個が貯まる方が、事前登録した 100 個より確実に精度が高いはずです。

台帳を "作り込む" 運用ではなく、"運用の中で育てる" 設計。これは docsweep でも kb-alias-add でもずっと同じ姿勢で作っていて、たぶんこのタイプの個人ツールでは共通の勘所だと思っています。

## 学んだこと

- 音声入力と長い英字リポ名の相性の悪さは、**中間層に呼称索引を 1 枚挟むだけ**で解ける
- 既に別文脈で解いてある仕組みは、**新規設計より "1 型を足す" 方が速い**（今回は `ref_type=project` の追加だけ）
- 台帳の分離基準は「秘匿度」ではなく「共有範囲」。**共有先の実態を先に 1 問確認する** が全て
- xlsx 直接運用は AI 書き込みでファイルロックとバイナリ diff 不可の 2 大地雷。**CSV 正本 + Excel で開く** で両立
- 集計は AI にその場で頼めばいい。**"集計テーブルを人手でメンテする" を丸ごと消す** 方が xlsx ピボット維持より楽
- 台帳は事前投入せず **使いながら育てる**。実際の誤変換パターンは机上で予測できない

## docsweep はこんなときに刺さります

- AI コーディングツールが吐いた `plan_*.md` / `bugfix_*.md` が各プロジェクトに散らかっていて、どれが生きてどれが死んでいるか分からない人
- 複数プロジェクトを横断して「今日 1 個だけ」を機械的に選ばせたい人
- md の frontmatter を OKF（Open Knowledge Format）互換で揃えて、他ツールとも読み合わせたい人
- 呼称索引と組み合わせて、音声だけでプロジェクト間を渡り歩きたい人

`pip install docsweep` で 1 分で入ります。設定ゼロで動きます。

- リポジトリ（Issue / PR 歓迎）: [github.com/ishizakahiroshi/docsweep](https://github.com/ishizakahiroshi/docsweep)
- PyPI: [pypi.org/project/docsweep](https://pypi.org/project/docsweep/)

他にも音声・AI 連携まわりで小さいツールをいくつか公開しています。個人サイトの [ishizakahiroshi.com](https://ishizakahiroshi.com/) の Works からまとめて眺められます。

## おわりに

29 リポの管理が「難しい」というより「発音が難しい」で詰まっていたのが、正直な話でした。設計としては CSV 2 枚と skill 追記だけ。図解の HTML を 1 枚生成して、あとは運用の中で育ちます。

同じ症状で困っている人がいたら、たぶん似た仕組みは 30 分で作れます。正式リポ名の台帳と別名索引の 2 ファイルと、「別名を 1 行足す」だけの小さな skill、それだけです。

小さく。運用の中で育つ台帳を、これからも増やしていきます。

---

書いた人: ishizakahiroshi
田舎在宅の受託エンジニア。バックエンド・インフラ・AI 連携を中心に、業務委託で個人開発の道具作りをしています。カジュアルな相談歓迎です。

- サイト: https://ishizakahiroshi.com/
- GitHub: https://github.com/ishizakahiroshi
- X: https://x.com/ishizakahiroshi

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。
