---
title: "Claude Code の /doctor prompt-audit で指示ファイルを監査したら、読み込まれていない CLAUDE.md が出てきた"
tags:
  - ClaudeCode
  - CLAUDE.md
  - AIエージェント
  - Windows
  - 個人開発
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![机に積まれた書類の束を、手元の明かりで一枚ずつ照らしている](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-29_claude-code-prompt-audit/01_2026-09-29_prompt-audit_hero.png)

203 件。Claude Code に入った `/doctor prompt-audit` を、自分の指示ファイル一式にかけたら返ってきた所見の数です。CLAUDE.md、スキル 54 本、スラッシュコマンド、import 先まで全部なめられました。

`/doctor prompt-audit` は v2.1.283 で追加された監査で、CLAUDE.md やスキルに残っている古い書き方や、実物とずれた記述を洗い出して修正案を出します。知ったきっかけは note の紹介記事でした。

- Claude Code の CHANGELOG: https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md
- 紹介記事「Claude Codeに『prompt-audit』登場。古いCLAUDE.mdやSkillsをAIが自動点検する時代へ」: https://note.com/dialogs_develop/n/ncb20c8764e7a

先に結論です。

- **`/doctor prompt-audit` は同梱の claude-api スキルで動きます。** `skillOverrides` で `off` にしていると使えません。一覧を小さく保ちたいなら `name-only` にします
- **Windows で `@C:\...` と書いた import は、バックスラッシュのところで切れて黙って読まれません。** `@C:/...` と `/` で書きます
- **当てたのは不具合だけにしました。** 書き方の好みの指摘は、複数の AI CLI で同じファイルを共有している運用には合わないものが多かったからです

## many-ai-cli について

自作で [many-ai-cli](https://github.com/ishizakahiroshi/many-ai-cli) という AIツール / Webダッシュボードを作っています。複数の AI コーディング CLI を並列で走らせ、承認をブラウザ 1 タブに集約。スマホからでも。今回の監査では、このツール自身の不具合も 1 つ見つかりました。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/many-ai-cli

同じ悩みを持っている方は、下記で入ります。

```sh
npm install -g many-ai-cli
many-ai-cli setup
```

入れたら 1 回だけ `many-ai-cli setup` を実行します。対応する AI CLI（Claude Code や Codex など）は、使うものを別に入れておいてください。

起動後の見え方は OS で違います。

Windows では、デスクトップに「MANY-AI-CLI」のショートカットができます。ダブルクリックするとトレイにアイコンが出るので、「Hub を開く」を選ぶとブラウザで Hub が開きます。止めるときはトレイメニューの「Hub を停止」です。

macOS と Linux では「Many AI Hub Start」と「Many AI Hub Stop」ができます。Start と一緒に開くコンソールが Hub 本体なので、閉じずに最小化してください。

ターミナルから直接起動するなら `many-ai-cli serve --open`、止めるなら `many-ai-cli stop` です（全 OS 共通）。Hub が開いたら、新しいセッションから Claude Code などを起動します。

README には、実機で確認できているのは Windows で、macOS と Linux のネイティブ環境は十分には確認できていない、と書いてあります。

前回は、同じ `/doctor` でスキルの棚卸しをしました。今回はその続きで、スキルの中身まで監査にかけた話です。

前回の記事: [30 個のスキルを積んだ Claude Code に /doctor をかけて、棚卸ししてみた](https://qiita.com/ishizakahiroshi/items/c114346e08dc1f382b0e)

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-29_claude-code-prompt-audit/02_2026-09-29_prompt-audit_infographic.png)

## 動かす前に、自分の設定で止まった

最初に `/doctor prompt-audit` を打ったら、監査は始まらず「このセッションでは使えない」と返ってきました。

原因は自分でした。使わない同梱スキルを `skillOverrides` でまとめて `off` にしていて、その中に claude-api が入っていました。prompt-audit はこのスキルに乗っています。

`off` はモデルからも `/` からも隠します。`name-only` なら一覧には名前だけが載り、呼べば使えます。

```json:settings.json
{
  "skillOverrides": {
    "claude-api": "name-only"
  }
}
```

これで動きました。再起動は要りませんでした。

## 203 件のうち、いちばん多かったのは「中身が古い」

所見の内訳です。

| 分類 | 件数 |
|---|---|
| 古いプロンプトの書き方（強調しすぎ、昔のモデル向けの言い回し） | 47 |
| スキルと設定ファイルの中身（実物とのずれ、ファイル間の矛盾、経緯の書き残し） | 147 |
| スキルの説明文 | 7 |
| 構造やコードの問題 | 2 |

「古いモデル向けの書き方を直すツール」だと思っていたので、ここは意外でした。多かったのは書き方ではなく中身です。

たとえば、プロジェクトの CLAUDE.md に「v0.8.0 まで出荷済み」と書いてありました。2 日前に v0.9.0 のタグを打っています。しかも同じファイルの 2 行上に「実装状況をここに書き写さない」という規則がありました。書いた理由どおりに古くなっていたわけです。

ほかにも、もう無い設計書を「正本」と呼んでいる行や、グローバルの CLAUDE.md に存在しない節名を参照しているスキルが出てきました。どれも書いた日には正しかったものです。

監査は 7 体のサブエージェントに分かれて走り、1 体だけで 20 万トークンを超えていました。重いです。

![山積みの紙片を、小さな箱と大きな箱に仕分けている手元](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-29_claude-code-prompt-audit/03_2026-09-29_prompt-audit_illustration-sorting.png)

203 件をそのまま当てるのではなく、1 件ずつ実物と突き合わせて、直すものと残すものに分けました。直す側の箱は、最後まで小さいままでした。

## 当てたのは不具合だけにした

うちのスキルは 1 か所に正本を置いて、Claude Code、Codex、Copilot などの各 CLI からリンクで同じものを見せています。監査の判断は Claude 基準なので、「今のモデルに強調は要らない」「経緯の段落は消してよい」といった案は、ほかの CLI には当てはまらないことがあります。経緯の段落は、規則の理由としてわざと残しているものも多いです。

そこで「不具合」を次の 5 つに絞って、それだけを当てました。

- 実物と食い違っている（版数、ファイル名、コマンド）
- 同じファイルの中や、グローバルの規則と矛盾している
- 参照先が無い
- 手順どおりにやると壊れる
- 秘密の扱いの規則に反している

1 回目に 37 件、2 回目に 52 件。判断が要るものは選択肢にして、5 問だけ自分で決めました。当てる前に根拠の行を開き直したら、不具合ではなかったものも 5 件ありました。監査の所見を、そのまま事実として扱わないほうがいいです。

## いちばん効いたのは、読まれていなかった import 行

グローバルの CLAUDE.md には import が 5 行あります。監査は「そのうち 1 本だけ、このセッションに読み込まれていない」と指摘していました。チームで共有している指示ファイルを読み込む行です。ファイルは存在します。原因は「確かめていない」とありました。

そこから先は自分で見ました。Claude Code 本体（実行ファイルに同梱されている JS）で import を拾う正規表現を探すと、こうなっていました。

```js
/(?:^|\s)@((?:[^\s\\]|\\ )+)/g
```

`@` の後ろは「空白でもバックスラッシュでもない文字」か「バックスラッシュ＋空白（空白のエスケープ）」だけを拾います。つまり `@C:\path\to\shared\CLAUDE.md` は `@C:` までしか読まれません。エラーは出ません。

![import 行はバックスラッシュのところで切れ、/ 区切りなら最後まで読まれる](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-29_claude-code-prompt-audit/04_2026-09-29_prompt-audit_fig1.png)

図のとおり、Windows のパス区切りのまま書くと、`@` の直後のドライブ名だけが拾われて、その先は捨てられます。区切りを `/` にすれば最後まで読まれます。

```text
@C:\path\to\shared\CLAUDE.md   ← C: までで切れる。黙って読まれない
@C:/path/to/shared/CLAUDE.md   ← 最後まで読まれる
```

この行を書いたのは、チームで共有しているセットアップ手順でした。PowerShell の `Join-Path` で import 行を組み立てていたので、区切りが `\` になっていました。手順のほうを `/` で組み立てる形に直し、古い行が見つかったら置き換え候補を見せて承認を取るようにしました。すでに入れているメンバーの分は、各自で直してもらう必要があります。

正直、これは監査が無ければ気づきませんでした。読み込まれていないことは、何も起きないことでしか分からないからです。直したあとの読み込みは、次のセッションで `/memory` を開いて確かめるところです。

## ドル記号と数字は、スキルの中では別の意味になる

直している途中で、監査の外でもう 1 つ見つけました。スキルを引数付きで呼ぶと、本文の `$0`、`$1` のようなドル記号と数字は、その番号の引数に置き換わります。ドル記号を文字として残したいときはバックスラッシュを前に付ける決まりで、そのバックスラッシュは消えます。引数なしで呼んだときは何も変わりません（ここは Claude Code 本体のコードで確かめました）。

- 公式ドキュメント（スキルへの引数の渡し方）: https://code.claude.com/docs/en/skills

困るのは、スキルの中に awk を書いているときです。

```sh
# 引数付きで読み込むと \ が消え、シェルが $2 を空に展開する
awk "/VmRSS/{print \$2}" status

# 括弧付きの列指定なら、置き換えの対象にならない
awk "/VmRSS/{print \$(2)}" status
```

awk は `$(2)` を 2 列目として読みます。awk のほうは、書き換えの前後で同じ出力になることを bash で確かめました。ドル記号と数字を書いていたスキルは、ほかも含めて 4 本を直しました。

## Windows では claude.md が CLAUDE.md として読まれる

many-ai-cli には、provider 名をファイル名にしたデータファイルがあります。`resources/slash-commands/claude.md` は、118 行のスラッシュコマンドの表です。

Windows はファイル名の大文字と小文字を区別しません。そのフォルダの別のファイルを読んだとき、この表が「入れ子の CLAUDE.md」として指示の枠に注入されていました。ファイル名は配信の約束ごとなので、変えられません。

プロジェクトの設定で読み込みから外しました。

```json:.claude/settings.json
{
  "claudeMdExcludes": ["**/resources/*/{CLAUDE,claude}.md"]
}
```

このパターンが該当のパスに当たることは確かめました。新しいセッションで注入が止まったかは、まだ見ていません。

## 監査は、自作ツールの空行バグも拾っていた

グローバルの CLAUDE.md の末尾に、空行が 33 行続いていました。その直後にあるのは、many-ai-cli が承認ルールを読み込ませるために差し込む import 行です。

many-ai-cli は、セッションを始めるときに CLAUDE.md の末尾へ import 行を足し、終わるときに取り除きます。足すときは前に改行を 1 つ付けているのに、取り除くときは import 行だけを消していました。往復するたびに空行が 1 行残ります。

控えを見返すと、8 月の時点で 76 行、片付ける直前で 144 行。片付けて 1 行にしてから 8 日で、また 33 行になっていました。なんで今まで気づかなかったのか。

「元の内容に差し込んで取り除く、を 3 往復したら、元とバイト単位で一致する」というテストを書くと、見当どおり落ちました。

```text
want: "# rules\n\nbody\n"
got:  "# rules\n\nbody\n\n\n\n"
```

取り除くときに、差し込んだ形がそのまま残っていればその分だけを落とし、崩れていれば import 行と直前の空行 1 行だけを落とすように直して、テストは通りました。修正はまだリリース前で、実機での往復も確かめていません。

このリポジトリには「利用者のファイルへ書く機能は、次回起動時の回収まで設計する」という原則を書いてあります。回収したつもりで、空行を置き去りにしていました。

もう 1 つ、利用者全員に配っている承認ルールの文面に、手元の非公開メモのパスと事件の番号が入っていました。規則とその理由は同じ行に書き切ってあったので、その括弧だけを消して配布の版を上げました。

## ついでに揃えたもの

監査の指摘の中に、「ビルドやタグの push はユーザーの指示があるまでやらない」というグローバルの規則と、公開手順のスキルがぶつかっているものがありました。

npm、crates、PyPI への公開、タグの push、X の予約投稿。どれも出したら取り消せないので、実行の直前に「この名前、この版、この中身で出していいか」を 1 行だけ確認する形に揃えました。聞くのは取り消せない操作の直前だけです。

サーバー再起動のスキルでは、再起動後に動かすつもりの一時タイマーが、再起動で消える作りになっていました。これも起動時に動く形へ直しました。

## 学んだこと

- 監査の所見は「どこが食い違っているか」までです。import が読まれない理由は、所見の外にありました
- 指示ファイルでいちばん腐るのは、書き方より中身でした。版数、ファイル名、節名
- 複数の CLI で共有している棚には、Claude 基準の書き方の直しをそのまま当てない。不具合だけ当てる
- ファイルに書いて消す機能は、往復でバイトが一致するかをテストで見る

## many-ai-cli はこんなときに刺さります

- Claude Code・Codex・Copilot・Cursor・Grok を同時に走らせて、状態を 1 画面で見たい人
- 各 CLI の承認待ちを、1 つのタブでまとめて捌きたい人
- 離席中でも、スマホから承認したい人
- Claude Code の CLAUDE.md へ承認ルールを差し込む仕組みも、往復したら元のバイト列に戻るよう直しました（リリース前）

いずれかに心当たりがあれば、`npm install -g many-ai-cli` と `many-ai-cli setup` の 2 コマンドで試せます。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/many-ai-cli
- npm パッケージ: https://www.npmjs.com/package/many-ai-cli

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## あわせて読みたい

- [200 行ルールを疑って、自分の CLAUDE.md を『発火頻度』で仕分け直した話](https://qiita.com/ishizakahiroshi/items/8ffdb968963c4e992662)（同じ CLAUDE.md を、前は自分の手で仕分けていました）
- [Claude Code / Codex / Cursor / Copilot / OpenCode で同じ Agent Skills を共有する。正本 1 箇所 + リンクの設計](https://qiita.com/ishizakahiroshi/items/6821655d5af59a32e50c)（今回、Claude 基準の直しを当てなかった理由になっている棚の作りです）
- [CLAUDE.md と AGENTS.md と GEMINI.md を全部書くのをやめた。どの AI CLI が何を読むか実測して正本 1 本に寄せる](https://qiita.com/ishizakahiroshi/items/ffecb88684c29803b3c6)（「本当に読み込まれているか」を実測した前の話です）

## おわりに

直したもののうち、import 行と claude.md の除外は、次にセッションを開くまで効いたかどうか分かりません。読み込まれていないことは、何も起きないことでしか分からない。そこが今回いちばん怖かったところです。

指示ファイルは書いた日から古くなっていきます。たまにこうして監査にかけて、実物と突き合わせる。それを続けていきます。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-09-29_claude-code-prompt-audit/

※ ヘッダー画像とインフォグラフィックの絵は AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
