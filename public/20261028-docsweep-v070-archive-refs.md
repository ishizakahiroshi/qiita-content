---
title: 作業メモを archive へ移すと参照が古いパスのまま残る問題を、docsweep v0.7.0 で移す前に止めて移した後に確かめる形にした
tags:
  - docsweep
  - Python
  - CLI
  - ClaudeCode
  - AIエージェント
private: false
updated_at: '2026-10-28T23:19:46+09:00'
id: a51889e04e459ac3044e
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![docsweep v0.7.0 の記事のヘッダー画像](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20261028-docsweep-v070-archive-refs-q01_hero.png)

先に、結論を3行で書きます。

- Claude Code などの AIエージェントが残す作業メモ（Markdown）を archive へ移すと、外の docs やコードの注釈からの参照が、古いパスのまま残っていました。
- docsweep v0.7.0 の `scope: repo` は、移すと困る文書を移す前に止めます。リリース前の開発中の版で業務の repo の写しを回すと、31本を止めて49本を移しました。
- 移した後は、古いパスの残りと切れたリンクを機械で確かめます。同じ実行で0でした（履歴の置き場は数えない設定で）。

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20261028-docsweep-v070-archive-refs-q02_infographic.png)

2026年10月下旬、md が約1000本ある業務の repo で、同じ整理を使い捨てのスクリプトで2回書きました。完了した作業メモを移して、参照を直して、確かめる。

2回書いたので、さすがに道具の側に入れて、2026-10-28 に v0.7.0 として公開しました。

自作で [docsweep](https://github.com/ishizakahiroshi/docsweep) という CLI を作っています。AI が量産する作業ログ Markdown を、腐らせず自動で片付ける。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=docsweep
- 使い方と設計の詳しい説明: https://ishizakahiroshi.com/articles/docsweep/usage.html
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/docsweep

```bash
pip install docsweep
python -m docsweep init                                # 初回だけ。既定の設定を作る
python -m docsweep sweep --dry-run --ref-scope repo    # 何が移り、何が止まるかを先に見る
```

PATH に無くても `python -m docsweep` で動きます（macOS と Linux は `python3`、Linux の新しめの distro は venv か `pip install --user`）。確認は Windows と Linux（CI）で、macOS は確かめていません。

## 移したはずの文書を、外のファイルがまだ指していた

v0.6.0 までの archive への移動は、`docs/local` を git で追跡しない repo が前提でした。参照を直すのも、queue の中の md の frontmatter と相対リンクだけ。

業務の repo では、ほかの docs やコードの注釈からも作業メモを指しています。移すと外の参照は古いパスのまま残り、コードが参照している文書も止まらずに移りました。

![移した後も、外のファイルから古いパスを指したまま残った参照の場面](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20261028-docsweep-v070-archive-refs-q03_illust.png)

挿絵は、移した後も外の参照が元の場所を指したまま残った場面です。

機能を足す前の docsweep で業務の repo の写しを回すと、80本を移して、本文のパス 229か所のうち 186か所が旧パスのまま残りました。コードから参照されている文書 16本も移り、参照の更新の失敗も 28件出ています。

※ 186か所は、除外の設定なしで全ファイルを数えた値です。後で出る検証の0とは数え方が違います。

## `scope: repo` を1つ書くと、止める・移す・直す・確かめるが揃う

| | v0.6.0 まで | v0.7.0 の `scope: repo` |
|---|---|---|
| 参照を直す範囲 | queue の中の md | repo 全体。本文のパスと html の `href` / `src` も |
| コードが参照している文書 | 止まらずに移る | 移す前に止める |
| 移した後 | 確かめない | 3つを数え、1件でもあれば終了コード 1 |

`.docsweep.yaml` に書きます。1回だけ試すなら `sweep` / `promote` / `apply` に `--ref-scope repo` です。

```yaml
archive_refs:
  scope: repo                # queue（既定）| repo
  exclude:                   # 書き換えない・検証しない場所（repo 相対の glob。* は / もまたぐ）
    - docs/obsidian/evidence/**
  code_paths: [src/**]       # ここから参照されている文書は移さない
  rewrite_extensions: [.md, .html, .htm, .txt, .json]   # 書き換える拡張子（既定）
```

止める文書を決め、残りを移し、移せた文書だけ参照を直して、確かめます。`exclude` と、未コミットの変更があるファイルには触りません。

![scope: repo の流れ。移す前に止める、移す、参照を直す、確かめるの4段と、各段で見るもの](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20261028-docsweep-v070-archive-refs-q04_fig.png)

図は4段と、各段で見るものです。止めた文書は JSON の `held`、検証は `verify` に出ます。

止める理由は5つ。

- コード・設定から参照されている
- 文書自身に未コミット・未追跡の変更がある
- 未コミットのファイルから参照されている
- 移すと git の ignore から外れる
- 同じ系統（`plan_x.md` と `plan_x_c1_ui.md` など）が queue に残る。`sweep` / `promote` だけ

止めても終了コードは 0。`apply` だけは、止める理由があれば何も変えずに 2 で終わります。

## なぜオプションを分けず、1つの切り替えにしたのか

機能ごとのオプションにすると、範囲だけ広げて、止める仕組みを付け忘れる組み合わせが作れてしまう。なので1つにまとめました。

先に書き換えると、移動に失敗した文書の参照が壊れます。だから書き換えは、移動が成功した文書だけ、移動の後です。

切れたリンクは「移す前は届いていた」ものだけ数えます。今のディスクだけで判定すると、元から切れていたリンクも拾うからです。手作りのときは、これで31件の誤検出が出ました。

## 業務の repo の写しと、公開版の合成 repo で回してみた

同じ写しで、`exclude` に履歴・旧記録の置き場を指定して回しました。

※ v0.7.0 そのものではなく、リリース前の開発中の版で、ある業務の repo の写しを回した結果です。

| 実行 | 移した | 参照の書き換え | 書き換えの失敗 | 止めた | 検証 |
|---|---|---|---|---|---|
| `--ref-scope queue --dry-run` | 80 | 138 | 0（以前は 28） | なし | 走らない |
| `--ref-scope repo --dry-run` | 49 | 143 | 0 | 31（コード参照 18・系統 13） | 走らない |
| `--ref-scope repo`（19秒） | 49 | 143 | 0 | 31 | 3つとも 0 |

前は外の参照が古いまま残り、コード参照の文書も移っていました。今回は31本を止め、移した49本で古いパスの残りは0です（履歴の置き場は数えない設定）。

リリース直前の見落としも1つ。コードの注釈がファイル名だけ（`plan_x_c5_report.md` など）で書いていて、止めるべき文書を「移せる」と出していました。直して dry-run だけ回し直すと、移す 40・止める 40（コード参照 21・系統 19）。表はこの直しの前の値です。

公開版は、`pip install docsweep==0.7.0` を新しい環境に入れ、合成の repo で確かめました。

```text
docs/guide.md            # リンクと本文のパスで plan_done.md を指す
docs/local/plan_done.md     [完了]
docs/local/plan_coderef.md  [完了]
docs/local/plan_x.md        [計画]
src/app.py               # plan_coderef.md を参照
```

dry-run（終了コード 0）は移す1本と、止める1本（`plan_coderef.md`、`referenced_by_code`）を出し、本実行も同じに動いて検証は3つとも 0。`docs/guide.md` は `archive/plan_done.md` を指すように書き換わりました。

```text
held    止めた文書。理由と参照元つき
verify  stale_refs（旧パスの残り）/ missing_targets（実体のない参照）/ broken_links（切れたリンク）
```

## Web UI と `release close` は、まだ queue の範囲のまま

Web UI の「完了を archive」と `release close` は、`scope: repo` でも queue の範囲のままです。区切りの混在・`file:///`・`./` 付きのパスも対象外。

v0.7.0 で入ったこの仕組みは、v0.8.0 でバックスラッシュ区切りと repo の中の絶対パス、dry-run での検証の見込み、`--root docs/local` の拒否を補いました。`pip install docsweep` で入るのは 0.8.0 以降です。v0.7.0 には表示の日英切り替えと `docsweep mv` も入っています。

## docsweep はこんなときに刺さります

- 完了した plan / bugfix の md がたまっている人
- 古くなった作業メモを「要判断」として拾いたい人
- 複数のプロジェクトの作業メモを1画面で見たい人
- 作業メモを git で追跡し、外の docs やコードからも指している人

`pip install docsweep` と `python -m docsweep init` の2つで試せます。今回の機能は `.docsweep.yaml` に書くか、`--ref-scope repo` を付けたときだけ動きます。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=docsweep
- 使い方と設計の詳しい説明: https://ishizakahiroshi.com/articles/docsweep/usage.html
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/docsweep
- PyPI: https://pypi.org/project/docsweep/

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## あわせて読みたい

- [AIが残すplan・bugfixを自動で片付けるdocsweepの始め方](https://qiita.com/ishizakahiroshi/items/17bf7a02efcb8c4718ec)
- [手元の pytest が緑でもリリースしてはいけない。タグを打つまでに踏んだ4つの穴（CI / CSP / 依存上限 / wheel）](https://qiita.com/ishizakahiroshi/items/9653a0a9273abd99a699)

## おわりに

スクリプトで2回やった「移す・直す・確かめる」は、設定1つになりました。

Web UI と `release close` のどちらから repo の範囲に揃えるかは、まだ決めていません。片方ずつ進めます。

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
