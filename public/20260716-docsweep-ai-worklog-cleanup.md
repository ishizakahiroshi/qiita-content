---
title: "AIが残すplan・bugfixを自動で片付けるdocsweepの始め方"
tags:
  - Python
  - CLI
  - OSS
  - 生成AI
  - 開発効率化
private: false
updated_at: ''
id: ''
organization_url_name: null
slide: false
ignorePublish: false
---

![AI作業ログを片付ける - docsweep はじめの一歩](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260716-docsweep-ai-worklog-cleanup/hero.png)

![docsweep導入ガイド](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260716-docsweep-ai-worklog-cleanup/infographic.png)

AIコーディングツールと作業していると、`plan_*.md`、`bugfix_*.md`、`pending_*.md` が増えていきます。書いた直後は役に立つ。でも数週間後に開くと、終わったのか、まだやるのか、そもそも今も正しいのかが分からない。

この「Markdownの作業記録が静かに腐っていく問題」を片付けるために、Python製CLIの `docsweep` を作っています。先日、自分でdocsweepのリリースを終えた直後に、docsweep自身がリリース記録だけ拾えないという穴を踏みました。なんでだ。

この記事では、今回の修正で何を変えたかに加えて、初めて触る方がインストールして、今日の1件を選び、完了ファイルを安全にarchiveするところまでをまとめます。

## docsweepは、AIの作業記録を片付けるCLIです

自作で [docsweep](https://github.com/ishizakahiroshi/docsweep) というCLIを作っています。AI が量産する plan / bugfix の Markdown を、腐らせず自動で片付けるクロスプラットフォーム CLI。

同じ悩みを持っている方は、下記で入ります。

```bash
python -m pip install docsweep
```

リポジトリはこちらです（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/docsweep

やっていることは、意外と地味です。

- ファイル名から `plan`、`bugfix`、`pending` などの種類を判定する
- H1の `[計画]`、`[実行中]`、`[様子見]`、`[完了]` などを読む
- YAML frontmatterがあれば、状態、期日、担当、関連ファイルも読む
- 古くなった記録を「要判断」として見える場所へ出す
- `[完了]` と `[廃止]` だけを `archive/` へ移す

中身をAIに要約させて勝手に削除する道具ではありません。判断は人かAIエージェントが行い、docsweepは決まったルールで運ぶ役です。物理削除も持たず、最後はarchive止まり。このくらいの慎重さが、作業記録を預ける道具にはちょうどよいと思っています。

![docsweepが読むものと片付けるまでの流れ](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260716-docsweep-ai-worklog-cleanup/fig1.png)

上の図のとおり、docsweepは「ファイルの種類」と「現在の状態」を別々に読みます。何の記録かを判定してから、`[完了]` または `[廃止]` のときだけarchiveへ運ぶ流れです。

## まず3コマンドだけ試せば雰囲気が分かります

コアのCLIだけなら、依存はかなり軽めです。

```bash
python -m pip install docsweep
```

Web UI、MCP、対話レビューなどもまとめて使いたい場合はこちらです。

```bash
python -m pip install "docsweep[all]"
```

インストールできたら、`plan_*.md` や `bugfix_*.md` があるプロジェクトへ移動して、まず `brief` を実行します。

```bash
python -m docsweep brief
```

`brief` は候補を大量に並べず、「今日の1個」を1件に絞って返します。朝、前日の続きを思い出すために全ファイルを読み直す必要がなくなります。複数プロジェクトを横断したいときは `cross` です。

```bash
python -m docsweep cross
```

完了済みのファイルを片付けるときは、先にdry-runします。

```bash
python -m docsweep sweep --dry-run
python -m docsweep sweep
```

1行目は移送予定を表示するだけです。対象を見てから2行目を実行します。もし移送後に「戻したい」となっても、直近の処理なら戻せます。

```bash
python -m docsweep undo
```

最初からWeb画面で眺めたい場合は、all extrasを入れたうえで次を実行します。

```bash
python -m docsweep serve --root ~/dev
```

ブラウザ上で、今日、期限超過、実行中、様子見、archive候補をまとめて確認できます。まずCLIで3コマンド、その後でWeb UI。逆でもよいです。無理に全部覚える必要はありません。

## ルールを手書きしたくなければ、injectできます

docsweepはファイル名とH1ラベルを手がかりにします。たとえば計画書なら、最小形はこれだけです。

```markdown
# [計画] ログイン画面を見直す

## 概要

入力エラー時の案内を分かりやすくする。
```

完了したらH1を変えます。

```markdown
# [完了] ログイン画面を見直す
```

この運用ルールをClaude CodeやCodex向けの指示ファイルへ手で転記するのは、地味に面倒です。プロジェクトへ導線を入れる場合は `inject` が使えます。

```bash
python -m docsweep inject --project . --preset claude-jp
```

既存の手書き部分を丸ごと置き換えるのではなく、docsweep管理ブロックとして追加します。取り外すときは `eject` です。

```bash
python -m docsweep eject --project .
```

AIエージェントに「作業開始時はbriefを見て」と毎回説明するところまで含めて、少しずつ機械に寄せられます。

## 自分のリリース記録だけ、なぜか拾われませんでした

今回の修正のきっかけは、docsweep v0.3.0を公開した直後でした。

リリース作業は `manual_release-v0.3.0_2026-07-16.md` に記録していました。H1は `[完了]`。最後にいつもどおり `sweep` を実行すればarchiveへ移るはずでした。

ところが、移送対象は0件。

原因は状態判定ではなく、その1段前でした。docsweepは、管理対象ではないREADMEや普通のMarkdownを巻き込まないため、ファイル名が既知のパターンに一致したものだけを読みます。当時の内蔵設定には、次の3種類しかありませんでした。

```python
DEFAULT_TYPES = (
    TypeDef("plan", "plan_*.md", ...),
    TypeDef("bugfix", "bugfix_*.md", ...),
    TypeDef("pending", "pending_*.md", ...),
)
```

`manual_release-*.md` は入口で対象外になっていた。H1が `[完了]` かどうかを見るところまで届いていませんでした。

言われてみれば単純です。でも、作った本人は「完了ラベルがあるから拾うだろう」と思い込んでいました。道に荷物を置いたのに、集荷対象の伝票が付いていなかったようなものです。

## 修正は1行。でも1行だけでは終わりません

コード上の中心は、本当に1行です。

```python
TypeDef("manual_release", "manual_release-*.md", (), "", 180),
```

これでファイル名が `manual_release` 型として認識されます。ただし、ここへ `auto_move=True` のような設定は足していません。

docsweepでは「何のファイルか」と「今どの状態か」を分けています。archiveしてよいかはtypeではなく、`[完了]` や `[廃止]` に対応する状態モデルが決めます。`manual_release` だけ特別扱いを増やさず、既存のルールへ乗せました。

```text
manual_release-*.md
        ↓ ファイル名でtype判定
manual_release
        ↓ H1またはfrontmatterで状態判定
done
        ↓ 状態モデルがarchive可と判断
sweepでarchiveへ
```

![manual_releaseが対象外だった状態と修正後の流れ](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260716-docsweep-ai-worklog-cleanup/fig2.png)

図にすると、直したのは一番左の入口だけです。その後ろにある状態判定とarchive処理は触らず、既存の安全策をそのまま使っています。

そして、1行の変更でもテストは3つ追加しました。

- 既定typeに `manual_release` が入っている
- scanすると `manual_release` として認識される
- `[完了]` のリリース記録が実際にarchiveへ移る

全体テストは653件が通っています。設定の1行だけ見れば小さい変更ですが、入口、判定、実移送まで1本で確認したかった。ここは少ししつこいくらいでよいです。

README、日英の説明、OKF対応表、配布するAI向けテンプレも、typeの一覧を4種類へ揃えました。コードだけ直して説明が3種類のままだと、次に使う人が迷います。実装より、この横の整合に時間がかかりました。

:::note warn
この記事の執筆時点でPyPIの最新公開版はv0.3.0です。`plan`、`bugfix`、`pending` の整理は今すぐ利用できます。今回紹介した `manual_release` の内蔵対応は開発ブランチへ入った段階で、次の公開版に含める予定です。
:::

## 自分で使うと、小さな穴が見つかります

docsweepは、docsweep自身のリポジトリでも使っています。

今回の穴は、ユニットテストを眺めていて思いついたものではありません。リリースが終わり、計画書とリリース記録を片付けようとして、実際の `sweep` が何も運ばなかったから見つかりました。

以前のbugfix記録も同じ場所に残っています。Web UIをCtrl+Cで止めたら正常終了なのに大きなtracebackが出たこと、frontmatterの警告が同じ内容で何度も出たこと。そういう記録を `[様子見]` で寝かせ、再発しなければ `[完了]` にする。ツールの説明書というより、道具を使いながら直した跡がたまっています。

少し格好悪い。でも、この手の道具は自分で踏んだ小石を拾っていく方が信用できる気がしています。

## docsweepはこんなときに刺さります

- 完了した作業Markdownを、判断後に安全にarchiveへ移したい人
- 古い計画書が「まだやるのか分からない」状態で増えている人
- 複数プロジェクトのplan、bugfix、pendingを1画面で見渡したい人
- AIとのリリース作業記録まで、同じライフサイクルへ揃えたい人

いずれかに心当たりがあれば、`python -m pip install docsweep` で気軽に試せます。まずは設定なしで、作業中のプロジェクトから `python -m docsweep brief` を実行するだけでも雰囲気が分かります。

- リポジトリ（Issue / PR歓迎）: https://github.com/ishizakahiroshi/docsweep
- PyPI: https://pypi.org/project/docsweep/

Starをいただけると開発の励みになります。使ってみて「ここが不便」があれば、IssueでもXのDMでも大歓迎です。

## おわりに

作業記録は、書く瞬間より、数週間後に読み返す瞬間の方が扱いに困ります。だから、立派なナレッジ基盤を先に作るより、まず「今日の1個を出す」「終わったらarchiveへ運ぶ」の2つを小さく自動化しました。

今回は、その片付け役からリリース記録だけ漏れていました。直したのは1行です。でも、入口から実移送までテストし、説明も揃え、最後にこのplan自体をdocsweepでarchiveへ送りました。ちゃんと自分で片付いた。

まず1プロジェクト。`brief` を朝に1回。そこからで十分です。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
