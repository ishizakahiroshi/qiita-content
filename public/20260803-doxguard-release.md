---
title: "AI がコードに埋めた自分の本名をコミット前に止める doxguard を公開しました"
tags:
  - Rust
  - Git
  - セキュリティ
  - 個人情報
  - OSS
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![doxguard ヒーロー画像](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-03_doxguard_hero.png)

AI にコードを書かせていたら、生成されたテストコードのパス例に自分の Windows ユーザー名が入っていたことがあります。`C:\Users\<本名>\projects\...` の形で、しれっと。コミット直前に気づいて、静かに背筋が冷えました。

API キーなら gitleaks が止めてくれます。でも「自分の本名」は、どのスキャナも探してくれません。無いなら作るしかない、で作ったのが [doxguard](https://github.com/ishizakahiroshi/doxguard) です。本日 v0.1.0 を公開しました。

- リポジトリ: https://github.com/ishizakahiroshi/doxguard
- 図解ガイド: https://ishizakahiroshi.github.io/doxguard/

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-03_doxguard_infographic.png)

## まず結論。これだけで試せます

```bash
# 試しに 1 回スキャン（構造パターンのみ・設定不要）
npx doxguard scan --all-tracked

# 本格導入（config + pre-commit hook + CI workflow を一括生成）
npx doxguard init
```

`init` が生成するのはこの 3 つだけです。

```
your-repo/
├── doxguard.config.json             # 構造パターン設定と watchlist への「参照」だけ
├── .githooks/pre-commit             # ポータブルな fallback hook
└── .github/workflows/doxguard.yml   # CI は構造スキャンのみ
```

exit code は 0 = 通過、1 = 検知でブロック、2 = 使い方エラーの 3 値で、既存の hook や CI にそのまま組み込めます。

## gitleaks があるのに、なぜ別のツールなのか

gitleaks（https://github.com/gitleaks/gitleaks）や trufflehog（https://github.com/trufflesecurity/trufflehog）が探すのは API キーやトークン、つまり「サービスの秘密」です。エントロピーや既知パターンで機械的に見つかります。

一方で、漏れて困るものには「自分自身」もあります。本名、家族の名前、勤務先、私用メール、自宅マシンの絶対パス、自宅サーバーの private IP。これらはエントロピーがゼロで、汎用の正規表現にもなりません。「山田」という文字列が秘密かどうかは、その名字の本人にしか分からないからです。

AI コーディングの時代になって、この種の事故は増えたと感じています。AI は手元の環境をよく見ていて、実在のパスやユーザー名を「それっぽい例」としてコードに埋めてくることがあります。悪気はないぶん、たちが悪い。

## 監視語リストを公開リポに置いたら本末転倒

作るうえで一番悩んだのはここです。「本名や家族名のリストと突き合わせる」設計は、そのリスト自体が最大の個人情報になります。リポジトリに置いたら本末転倒です。

なので doxguard には譲れない原則を 1 つだけ置きました。**監視語リスト（watchlist）は自分のマシンから出ない**。

リポジトリ側の config に書くのは、環境変数で展開されるパス参照だけです。

```json
{
  "watchlists": [
    { "type": "lines", "path": "${DOXGUARD_WATCHLIST_DIR}/names.txt" }
  ]
}
```

手元では環境変数が解決されてフルスキャンが走ります。環境変数が無い CI や他人の環境では watchlist は静かにスキップされ、構造パターン（Windows 個人パス・POSIX home・private IP・許可していないメールアドレス）だけが走ります。リストの中身も、置き場所すらも、リポジトリには残りません。

![watchlist が手元から出ない構成図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-03_doxguard_fig1.png)

図にすると上のとおりで、同じ config を共有したまま「手元ではフル監視・公開側では構造だけ」に自然に分かれます。ここが gitleaks 系との一番の違いだと思っています。

## 速さは pre-commit の生命線

hook は遅いと必ず外されます。doxguard は Rust 製で、監視語の照合は Aho-Corasick（複数キーワードの一括照合）、ファイル走査は rayon で並列化しています。`install-hooks` 時に native バイナリを `.git` ディレクトリ内へキャッシュして直接起動するので、コミットのたびに Node や Cargo が立ち上がることもありません。手元の Windows x64 での実測では、staged 0 件時の pre-commit は 20 回計測の中央値で 70ms 前後でした。体感ゼロです。

もうひとつ地味に大事な点として、staged スキャンは worktree ではなく **index blob** を読みます。「ワークツリーでは直したけど stage し直し忘れた」ケースを素通りさせない、pre-commit として正しい方を見るためです。

## 公開当日に自分で自分を監査した話

正直に書くと、今日の公開作業は監査の残件消化から始まりました。2 週間前のセキュリティ監査で HIGH 2 件を含む 15 件の指摘が残っていて、「セキュリティツールが既知の穴を抱えたまま初公開はないだろう」と全部片付けてからのリリースです。

HIGH の 1 つは Windows 固有でした。`Command::new("git")` は CreateProcess の検索順序（https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw）でカレントディレクトリを先に探すため、悪意あるリポジトリに置かれた偽の `git.exe` を起動し得ます。PATH ディレクトリからの絶対パス解決に統一して塞ぎました。もう 1 つは、生成する hook スクリプトへのパス埋め込みが quote されておらず、`$(...)` を含むパスだと commit のたびにコマンドが走り得るというもの。POSIX の single-quote に統一しました。

仕上げのレビューでは「修正した private IP の正規表現が、`192.168.001.x` のようにゼロ埋めされた octet を拾わなくなっている」という回帰まで見つかって、ここで小一時間溶けました。スキャナの正規表現は、厳しくすると別の何かを取りこぼす。身に沁みました。

![公開当日、監査残件を片付けている深夜の机](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-03_doxguard_illustration-1.png)

深夜の机で、チェックリストの残りを 1 つずつ消していく。そんな公開前夜ならぬ「公開当日の朝」でした。地味な作業ほど、後から効いてきます。

## 使いどころ

- AI コーディングツールに公開リポのコードを書かせている人
- 勤務先や顧客の名前を、個人の OSS に書いてはいけない立場の人
- dotfiles や設定ファイルを公開していて、自宅環境の情報が混ざりがちな人

逆に、API キーの検知は守備範囲外です。gitleaks と並べて使う前提で作っています。

## おわりに

自分の名前は、自分で守るしかありません。まずは自分の全リポジトリに配線するところから。小さく、続けていきます。

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
