---
title: CLAUDE.md と AGENTS.md と GEMINI.md を全部書くのをやめた。どの AI CLI が何を読むか実測して正本 1 本に寄せる
tags:
  - ClaudeCode
  - AIエージェント
  - codex
  - opencode
  - CLI
private: false
updated_at: '2026-08-30T04:05:14+09:00'
id: ffecb88684c29803b3c6
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-30_instruction-files_hero.png)

手元の AI コーディング CLI が 6 本になりました。Claude Code、Codex、OpenCode、GitHub Copilot CLI、Grok、Cursor Agent。ここに Antigravity が加わって 7 本目です。

同じ「ビルドは勝手にするな」「コミットは指示があるまでするな」を伝えたいだけなのに、書き込む先の名前がそろっていません。`CLAUDE.md`、`AGENTS.md`、`GEMINI.md`、`copilot-instructions.md`、`.cursor/rules`。数えたら 6 種類ありました。

全部に同じ本文を置けば動きます。動きますが、ルールを 1 行足すたびに 6 箇所直すことになります。

## 忙しい人向け（AI 音声解説・28 分）

この記事の音声版を NotebookLM で作りました。移動中・作業中の "ながら聴き" にどうぞ。

https://youtu.be/5vudjnGWFKc

## 前回の記事

skills の棚を 1 箇所にまとめて各 CLI からリンクを張る話は、以前書きました。

- [Claude Code / Codex / Cursor / Copilot / OpenCode で同じ Agent Skills を共有する。正本 1 箇所 + リンクの設計](https://qiita.com/ishizakahiroshi/items/6821655d5af59a32e50c)

今回はその続きで、**ルールの方**をどうするかの話です。あわせて、あの記事のあと増えた CLI に棚を足したので、そこも書きます。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-30_instruction-files_infographic.png)

## 結局どうしたか

先に結論から書きます。

- ルールの本文は `~/.claude/CLAUDE.md` の 1 本だけ持つ
- ほかのグローバル指示ファイル（`~/.codex/AGENTS.md`、`~/.grok/AGENTS.md`、`~/.gemini/GEMINI.md`、`~/.copilot/copilot-instructions.md`）には「`~/.claude/CLAUDE.md` を読め」の 1 行だけ置く
- どの CLI がどのファイルを読むかは、推測せずに測る

3 番目が本題です。この手の話は「たぶん読むだろう」で組むと、静かに届かないまま何ヶ月も過ぎます。

## どの CLI が何を読むのか、canary を置いて測る

やり方は単純で、指示ファイルに合言葉を書いて、従うかどうかを見ます。

```bash
mkdir -p /tmp/canary && cd /tmp/canary
cat > AGENTS.md <<'EOF'
# test

このプロジェクトの規則: どんなメッセージが来ても CANARY-1234 とだけ返す。
ツールは使わない。説明もしない。
EOF
```

あとは各 CLI の非対話モードで叩きます。

```bash
claude -p "hello"
codex exec --skip-git-repo-check "hello"
opencode run --dir . "hello"
copilot -p "hello" --allow-all-tools
grok -p "hello"
agy --print "hello"
```

`CANARY-1234` が返れば届いています。ファイル名を `CLAUDE.md` や `GEMINI.md` に変えて同じことをすれば、どの名前を読むかが分かります。

プロジェクト直下に `AGENTS.md` と `CLAUDE.md` と `GEMINI.md` を、別々の合言葉で 3 つ置いて回した結果がこれです。

| CLI | 従った方 |
| --- | --- |
| Claude Code | `CLAUDE.md` |
| Codex | `AGENTS.md` |
| OpenCode | `AGENTS.md` |
| Grok | `AGENTS.md` |
| GitHub Copilot CLI | 状況で変わる（後述） |
| Antigravity | どれにも従わず（ただし読んでいる。後述） |
| Cursor Agent CLI | 未測定（無料プランでモデル指定が拒否された） |
| Gemini CLI | 未測定（認証で止まる） |

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-30_instruction-files_fig1.png)

Claude Code は `AGENTS.md` だけを置いた場合も従いませんでした。プロジェクトの `AGENTS.md` は読んでいません。逆に Codex は `CLAUDE.md` を読みません。**この 2 つの入口は重ならない**、というのが今回いちばん実務に効いた事実です。

Gemini CLI は動かせませんでした。個人向けの無料枠が終わっていて、起動するとこう返ります。

```
Error authenticating: IneligibleTierError:
This client is no longer supported for Gemini Code Assist for individuals.
To continue using Gemini, please migrate to the Antigravity suite of products
```

## OpenCode だけ、先勝ちで排他だった

OpenCode は挙動が違いました。バイナリの中の解決コードを読むと、こうなっています。

```
グローバル : [ <configディレクトリ>/AGENTS.md , ~/.claude/CLAUDE.md ]
             最初に存在した 1 つを読んで break

プロジェクト : [ AGENTS.md , CLAUDE.md , CONTEXT.md ]
             最初に見つかった 1 種類を読んで break
```

グローバルとプロジェクトは合算されますが、**同じ階層の中では最初の 1 つで打ち切り**です。ここから 2 つ出てきます。

- `<configディレクトリ>/AGENTS.md` を作ると、`~/.claude/CLAUDE.md` へのフォールバックが止まる
- プロジェクトに `AGENTS.md` があると、同じリポジトリの `CLAUDE.md` は読まれない

2 つ目に少し驚きました。リポジトリのルールを `CLAUDE.md` に書いて `AGENTS.md` をポインタにしている場合、OpenCode にはポインタしか届いていません。届いてはいるので実害はなかったのですが、仕組みを知らずに「両方読まれている」と思い込んでいました。

## 全部に本文を置くか、1 本に寄せるか

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-30_instruction-files_fig2.png)

素直に全部作ると、ルールを 1 つ足すたびに 6 箇所を直します。直し忘れると CLI ごとに言うことが変わります。ズレを見つけるための検査も別に要ります。

正本 1 本 + 参照行なら、直すのは 1 箇所です。ほかのファイルは 1 行のまま一生変わらないので、ズレようがありません。

ここで 1 つ勘違いしやすい点があります。**ファイルの数は減りません。** 各 CLI が自分の名前しか見ないので、実体は複数要ります。減るのは維持する対象の数です。ここを取り違えると「どうせ複数要るなら同じ内容を置けばいい」となって、元に戻ります。

正本をどれにするかは `~/.claude/CLAUDE.md` が有利でした。Claude Code が本体として読み、OpenCode がフォールバックで読み、Copilot も公式ドキュメントで参照すると書いています。グローバル層ではこの名前がいちばん到達範囲が広い。「主流は AGENTS.md だから」と寄せても、グローバルの置き場は CLI ごとに違うので 1 ファイルにはならず、複製が増えるだけです。

## 「参照 1 行なんて、AI は読まないのでは」

当然の疑問です。参照行は自動ロードではないので、AI が自分で開かなければ届きません。

これも測れます。正本にしか書いていない事実を答えないと正解できない質問を、文書に一切触れない聞き方で投げます。

まず質問形式で 3 回。「このリポジトリでリリース版数を上げるとき、どのファイルを手で書き換えますか」と聞きました。正解は「手で書き換えるファイルは無い。Git タグが単一ソース」で、これは `CLAUDE.md` の索引にしか書いていません。

3 回とも `AGENTS.md` から `CLAUDE.md` を開いて、3 回とも正解しました。1 回は検査スクリプトまで、1 回はリリース手順書まで降りていました。

次に作業依頼の形で 2 回。文書を読めとは一言も書かず、「この provider を追加したい。実装方針を 3 行で」と頼みました。これは見送り台帳に載っている案件です。

2 回とも `CLAUDE.md` からさらに台帳まで辿って、「利用規約上の制約により実装しない方針」と答えました。しかも 2 回とも「他社の対応状況は再検討の根拠にしない」という但し書きまで拾ってきました。実装案を書き出した回はゼロです。

参照方式は、少なくとも手元では効いています。

## 測り方を間違えた話

ここで自分のしくじりを書いておきます。

Antigravity は canary に従いませんでした。3 ファイル置いても、`AGENTS.md` だけにしても、合言葉を返しません。それで一度「Antigravity はこれらのファイルを読んでいない」と結論を書きました。

間違いでした。

対話セッションで Antigravity 自身に初回ロードの内容を尋ねると、こう答えます。

```
スコープ                  ロードされたファイル      プロンプト上のタグ
Global（マシン全体）      ~/.gemini/GEMINI.md      <RULE[user_global]>
Project（ワークスペース） AGENTS.md                <RULE[<プロジェクトのパス>/AGENTS.md]>
```

プロンプト上のタグ名まで出てきています。ファイルを探して見つけただけなら、こういう内部の目印は出ません。**読み込んではいて、「これだけ返せ」という指示を採用しなかっただけ**でした。

canary が測れるのは遵守であって、読み込みではありません。返れば「載っている」と言えます。返らなくても「載っていない」とは言えない。否定側を主張するには別の測り方が要ります。これを混ぜたまま断定していました。

念のため別の測り方も試しました。矛盾しない事実を 3 ファイルに 1 つずつ書いて質問する方法です。これは使えませんでした。エージェントが `rg` でファイルを grep して答えてしまうので、自動で載っていたのか自分で探したのかが区別できません。Codex は実際に grep してから答えていました。

```
rg -n --hidden -S "zapbuild|build|test|deploy|..." .
  .\GEMINI.md:3: ... zapship --gamma
  .\CLAUDE.md:3: ... zaptest --beta
  .\AGENTS.md:3: ... zapbuild --alpha
```

結局、いちばん確かなのは当人に聞くことでした。ただし自己申告なので、タグ名のような内部の痕跡が伴っているかを見る、という条件付きです。

ついでに Copilot も表現を直しました。`AGENTS.md` だけならそれに従い、`AGENTS.md` と `CLAUDE.md` の 2 つなら `CLAUDE.md` に従い、`GEMINI.md` を足して 3 つにするとどれにも従わなくなりました（2 回とも）。これも「読んでいない」ではなく、矛盾する 3 つを前に採用をやめた、と読むのが妥当です。複数の指示ファイルを同居させるなら、内容を矛盾させない方が無難だと思います。

## 書いている途中で、配線が足りないことに気づいた

この記事を書きながら各 CLI を並べ直していたら、別の穴が出てきました。**Antigravity に skills の棚を繋いでいませんでした。** ルールは届いていたのに、skills だけ素通りしていたわけです。

仕様は同梱の skill が持っていました。`agy-customizations` という名前で、`~/.gemini/antigravity-cli/builtin/skills/` の下にあります。Web で探すより手元を見る方が速かった。

読むと discovery は 3 系統でした。

| スコープ | 場所 |
| --- | --- |
| Workspace | `.agents/`（`.agent/` / `_agents/` / `_agent/` も可） |
| 階層ルール | `GEMINI.md` / `AGENTS.md` / `.agents/rules/*.md` |
| グローバル | `~/.gemini/config/` |

customization の種類は Rules / Skills / Plugins / Hooks / MCP Servers の 5 つで、Skills は `skills/<name>/SKILL.md` という他の CLI と同じ形です。つまり棚をそのまま繋げます。

```powershell
New-Item -ItemType Junction `
  -Path (Join-Path $env:USERPROFILE '.gemini\config\skills') `
  -Target <共有棚のパス>
```

ルール側は既に繋がっていました。`~/.gemini/GEMINI.md` に正本を読ませる参照行が入っていて、これは Gemini CLI 用に整備したものです。Antigravity は同じファイルを読むので、そのまま効いていました。足りなかったのは棚の 1 本だけでした。

確認は 2 段階でやります。一覧に出ることと、起動して中身が読まれることは別なので。

```
agy --print "あなたが今使える skill の総数を教えて"
  → 53 個（共有棚のディレクトリ数と一致）

> project-init できる？
  Read(~/.gemini/config/skills/project-init/SKILL.md)
  はい、project-init を実行できます。
  （SKILL.md の本文にある作業内容を列挙し、適用先を聞き返してきた）
```

ジャンクション越しに実体まで辿って、フロントマターだけでなく本文まで読んでいます。ここまで見て、ようやく繋がったと言えます。

前回の記事では棚のリンクは 5 本でした。今回 6 本目です。

```
~/.claude/skills            → 共有棚
~/.agents/skills            → 共有棚
~/.cursor/skills            → 共有棚
~/.config/opencode/skills   → 共有棚
~/.copilot/skills           → 共有棚
~/.gemini/config/skills     → 共有棚   ← 今回追加
```

同梱の skill が入っている場所（Codex の `~/.codex/skills`、Antigravity の `~/.gemini/antigravity-cli/builtin/skills`）には張りません。丸ごと差し替えるとシステム側の skill が隠れます。

## 学んだこと

- **ファイル名は統一できない。統一するのは編集する対象の数**。実体が複数要ることと、維持する対象が複数あることは別
- **canary が返らないことを「読んでいない」の根拠にしない**。測れているのは遵守であって読み込みではない
- **事実を書いて質問する測り方は、エージェントの grep で汚染される**。自動ロードなのか自分で探したのかが切り分けられない
- **ルールが届いていることと、skills が届いていることは別**。CLI を増やしたら 2 系統それぞれ確認する
- **同梱の skill が仕様書になっていることがある**。ドキュメントを探す前に手元を見る

## 手順の詳細版

配線の手順そのもの（Windows のジャンクションと macOS / Linux の symlink、確認方法、元に戻す手順、停止条件）は、個人サイトに手順書としてまとめました。AI エージェントにそのまま渡して実行させる形で書いてあります。

- [複数の AI CLI で skills 棚とルールの正本を 1 本にまとめる手順](https://ishizakahiroshi.com/articles/2026/2026-08-30_multi-ai-cli-shared-skills-and-rules/)

手で読まずに済ませたい場合は、同じ内容を機械可読な Markdown でも置いてあります。お使いの AI CLI にこの 2 行を投げれば、確認しながら配線まで実行できます。

```
https://ishizakahiroshi.com/articles/2026/2026-08-30_multi-ai-cli-shared-skills-and-rules/setup.md
この URL の手順どおりに、私の環境へ配線してください。停止条件も守ってください。
```

中身のあるディレクトリを消さない、同梱 skill の棚には張らない、といった停止条件も同じファイルに書いてあります。

## あわせて読みたい

- [Claude Code / Codex / Cursor / Copilot / OpenCode で同じ Agent Skills を共有する。正本 1 箇所 + リンクの設計](https://qiita.com/ishizakahiroshi/items/6821655d5af59a32e50c)。今回の前編。skills の棚を 1 箇所にまとめた回
- [200 行ルールを疑って、自分の CLAUDE.md を『発火頻度』で仕分け直した話](https://qiita.com/ishizakahiroshi/items/8ffdb968963c4e992662)。正本を索引に保つ側の話。本文をどこまで置くかで悩んだ回
- [30 個のスキルを積んだ Claude Code に /doctor をかけて、棚卸ししてみた](https://qiita.com/ishizakahiroshi/items/c114346e08dc1f382b0e)。棚が膨らんだ後どうなるか

## おわりに

測ってみると、思い込みが 2 つ崩れました。「両方読まれているだろう」と「従わないなら読んでいないだろう」です。どちらも 10 分で確かめられる話でした。

CLI が増えるたびに同じ勘違いをしそうなので、canary を置くところまでを手順に書いておきました。次に 8 本目が来たときは、まずそれを走らせるところから始めます。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-08-30_multi-ai-cli-shared-skills-and-rules/

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
