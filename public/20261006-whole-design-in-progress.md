---
title: "AIエージェントへの依頼をどう回すか 受け箱とMCPを考えている途中です"
tags:
  - AIエージェント
  - MCP
  - GitHub
  - ClaudeCode
  - 個人開発
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![AIへの依頼を回す仕組みを、いま作っている途中です](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/01_2026-10-06_whole-design-in-progress_hero.png)

dotsなどのAIエージェントへの依頼を、受付から受入まで追う仕組みを考えています。Claude Codeやmany-ai-cliで作業し、返事はMCPで読む構想ですが、受け箱はまだ作っていません。

### いま使っている道具、作っている道具も紹介します

many-ai-cliは「複数の AI コーディング CLI を並列で走らせ、承認をブラウザ 1 タブに集約。スマホからでも。」という道具です。Desklyは「誰の返事待ちか、次に何をするか。」を扱う連絡台帳として開発中で、一般向けの製品は未公開です。[many-ai-cliの紹介](https://ishizakahiroshi.com/work.html?id=many-ai-cli)、[Desklyの紹介](https://ishizakahiroshi.com/work.html?id=deskly)

pnpmと、使うAIコーディングCLIを別途インストールした環境なら、公開READMEの手順で導入できます。導入・初回設定・ターミナルからの起動は、次の順です。

```sh
pnpm add -g many-ai-cli
many-ai-cli setup
many-ai-cli serve --open
```

Hubが開いたら「+ 新しいセッション」で使うAI CLIを選びます。導入・起動の詳細は[many-ai-cliの公開README](https://github.com/ishizakahiroshi/many-ai-cli)、開発中のDesklyは[公開repo](https://github.com/ishizakahiroshi/deskly)へ。

10月4日はSlack経由の依頼、5日は開発の連鎖と独立レビューを書きました。今回はその運用の上に、連絡と資料の置き場をどう足すかを整理した、2026年10月6日時点の記録です。[4日の記事](https://qiita.com/ishizakahiroshi/items/6c7e68815711017c981b)、[5日の記事](https://qiita.com/ishizakahiroshi/items/0ad648d4bf28b11739fe)

![できているのはSlack経由の依頼と方針の整理、途中は受付の土台、これからは受け箱と共通検査](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/02_2026-10-06_whole-design-in-progress_infographic.png)

依頼と返信は今も使えています。受付の土台は読み取りまでで、受け箱＋MCP、取り決めの書き換え、共通検査はこれからです。

## 返事を待つだけで、AIを何度も起こしていませんか

朝、手元のAIで5分ごとの返信確認を回しました。最初はDM全体を読んでいたため、別件の返事も拾って報告が増え、相談のスレッドだけに絞りました。依頼と返事を同じスレッドでつなぐ運用は、前作から続けています。[依頼の入口](https://qiita.com/ishizakahiroshi/items/6c7e68815711017c981b)

ただ、セッションを6個ほど開き、それぞれが5分ごとに確認する形は不毛です。読む先を一つのファイルに変えても、AIが毎回起きて「何もない」を確かめる空振りは残ります。

![複数のセッションが何度も郵便受けを見に行く場面と、一つの受け箱が到着時だけ知らせる場面](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/03_2026-10-06_whole-design-in-progress_illustration.png)

左は各セッションが返事を探す形、右は受け箱が到着時だけ知らせる構想です。右側の仕組みは未実装です。

同じ朝、返事8件のうち3件には案件番号がありませんでした。毎回、案件のスレッド内で、先頭に番号を付けると合意しました。機械はスレッドで振り分け、番号は人が一目で分かる目印にします。

今回の記事制作依頼では、手元で数えた08:16〜08:59の返事11件すべてに、先頭の案件番号があり、スレッドの外への返事もありませんでした。

## 入口から受入まで、何をどこに置くのでしょうか

全体は次の6層で整理しました。これは構成の整理で、全層が完成したという意味ではありません。

![入口、受付・番号、正本、これから作る連絡の経路、作業するAI、品質の6層](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/04_2026-10-06_whole-design-in-progress_fig-layers.png)

入口と作業者の間に、受付、正本、連絡の経路を置きます。連絡の経路はこれから作る層で、品質の共通検査などにも未実装の部分があります。

- 入口：Desklyの画面、Claude Desktop、手元のCLI。会社のチャットは会社ゾーンとして分けます。
- 受付・番号：本人・権限・番号・処理記録を扱います。Desklyを使わない場合はagent-boardに同梱する採番の道具を使う方針です。
- 正本：仕事の状態はGitHubのProjects・Issue・PR。指示・資料・共通ルールはagent-board、コードは作業対象のrepoです。
- 連絡の経路：受け箱＋MCP。送り状は検討中で、起こし方は環境ごとに分けます。
- 作業するAI：手元のAIと、dots・GrokBotなどクラウドのbot。成果はDraft PRで戻します。
- 品質：共通ルールを読ませ、道具と検査をそろえ、最後に手元のAIが差分を読みます。

状態を2か所で書き換えず、AIへの許可はGitHubの名義と招くrepoで止めます。人が見るのは最初の承認と最後の受入です。

会社が了承した名義が先で、個人名義のbotを会社の仕事につなぎません。中継を挟んでも、この境界は変えない方針です。

## 資料をまとめたら、進捗も同じファイルに書きますか

資料用のagent-boardへ、指示・報告・検討用HTML・レビュー資料を集約する構成を考えました。管理対象ごとに分け、製品のコード・テスト・PR・CIは作業対象のrepoに残します。

所在の索引に進捗や担当、採番まで持たせると、状態を書く場所が増えます。索引は資料を探すために使い、仕事の状態はProjects・Issue・PRを正本にする方針です。既存案件を一括で移動・再採番・再送することはしません。[Projectsの役割](https://docs.github.com/en/issues/planning-and-tracking-with-projects/learning-about-projects/about-projects)

整理の途中では、現状を確かめない変更説明や、ここで何を終えて次の誰へ渡すのかが曖昧な資料に違和感がありました。現状と変更後を比べる資料を求め、責務・終点・次の担当を書き足してもらっています。

資料のrepo、実際に変更するrepo、受付側の案件は別です。将来は普段のAIから「登録して」と明示して共通受付を呼び、同じGitHubの案件へ登録したい。ただし、このMCP受付は未実装です。

フォルダを分けても閲覧権限は分かれません。会社の資料など、閲覧範囲の違うものを個人の資料repoへ移す考えではありません。

### 片方だけでも使えるように考えています

agent-boardとDesklyは兄弟アプリとして、3通りを想定しています。

- agent-boardだけ：状態はIssue・Projects、番号は同梱の道具。
- Desklyだけ：GitHubが無い人、Gitが無い組織、一時的につながらない場合。Gitの無い置き場は、できるかをこれから調べます。
- 両方：agent-board側に公開仕様を置き、受付側が使う方針です。資料repo自体は非公開です。

共通の取り決めは「管理サービスが状態の正本」のままですが、受付側の設計は「GitHubが正本」に進んでいます。取り決めを核と連携時の追加に分ける書き換えは未着手で、文書のずれが残っています。

## 返事が届いたら、誰が取りに行くのでしょうか

最小版は、普通のプログラム1本が返事を受け、案件ごとに溜める形です。AIには巡回させません。MCPには「依頼を送る」「返事を読む」「読んだ印を付ける」の3つを用意する方針です。

届いたらOSの通知を出し、人が「見て」と言ったときにAIが道具を1回呼びます。自動で起こす部分は後から足します。

送信も通せば、案件とスレッドの対応をその場で記録できます。手作業では直近15件のうち6件にスレッドの位置が残っていませんでした。

送り元のセッション、repo名、任意のラベル、最初の依頼の要旨を添える「送り状」も検討中です。番号は送る側で振る、振り分けはスレッドを使う、依頼は原文でなく要旨を添える、という手元のAIの提案を含め、採否は未決定です。

## 動いているAIへ、そのまま知らせられますか

2026年10月6日に手元で行った、公式資料だけの調査です。8環境ともMCPは使えるという整理ですが、動いているセッションへ外から入力できるかは違いました。実機では未確認です。

| 環境 | 動作中のセッションへの入力 | 代わりの形・根拠 |
|---|---|---|
| Claude Code（ターミナル） | channelsで可能。研究段階。自作には開発用起動フラグが必要。閉じている間は届きません | 既存セッションを1回再開。画面には出ません。[公式資料](https://code.claude.com/docs/en/channels) |
| opencode | 固定ポートで起動し、入力欄への追加と送信が可能。別経路には画面に出ない不具合報告があります | 既存セッションを1回再開。画面には出ません。[公式資料](https://opencode.ai/docs/server/) |
| Codex | 公式に完成した仕組みは無く、実験的な仕組みで同じ画面へ入るかは未確認 | `codex exec resume`。[開発用コマンド](https://learn.chatgpt.com/docs/developer-commands?surface=cli)、[非対話実行](https://learn.chatgpt.com/docs/non-interactive-mode) |
| GitHub Copilot CLI | 確認できず。リモートの途中指示はクラウド経由 | 再開と非対話の併用は明記を確認できず。[公式資料](https://docs.github.com/en/copilot/how-tos/copilot-cli/use-copilot-cli/steer-remotely) |
| Cursor CLI | できません | 再開は可能。[公式資料](https://cursor.com/docs/cli/acp) |
| Grok Build | できません | 再開は可能。[公式資料](https://docs.x.ai/build/cli/headless-scripting) |
| Command Code | できません | 再開は可能。対話の履歴には出ません。[公式資料](https://commandcode.ai/docs/headless) |
| Claude Desktop | できません。channelsはターミナル版のみ | 通知を見た人が頼みます。[公式資料](https://code.claude.com/docs/en/platforms) |

![受け箱とMCPを共通にし、many-ai-cli内、Claude Code・opencode、それ以外の通知と人の一声へ分ける構想。公式資料の調査・実機未確認](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/05_2026-10-06_whole-design-in-progress_fig-wake.png)

返事を読む口は共通にし、起こし方を3通りに分ける構想です。自動通知まで接続して試した図ではありません。

many-ai-cliには既存セッションの端末へ文字を送る口があることを、同日のソースで確認しています。その中で動かすAIなら各社の機能に頼らず起こせる、という位置づけです。ただし、受け箱との実接続はこれからです。公開repoへのリンクは冒頭に置いています。

## 作る場所が変わっても、品質をそろえられますか

共通化したいのは、手元とクラウドで作るコードや資料の基準です。次の3段で考えています。

- 読ませる：スキルとCLAUDE.mdを共有します。守るかどうかはbotしだいです。
- 同じ道具：スキル59本のうち26本が手元のPowerShellの道具に触れています。実行が必須かは未確認です。
- 同じ検査：記事・判断資料のlintや秘密の走査をPRのCIで動かす構想です。この共通検査はまだ作っていません。

dotsの回答では、AGENTS.md・CLAUDE.md・skillsは指示されたパスや手順で読み、repoに置くだけでは自動で使われないそうです。これはdotsの自己申告で、手元では未検証です。「参照先の一覧＋固定commit」を案件の入口に渡す案から試します。

検査をそろえても、最後に手元のAIが差分を読む工程は残します。前作で扱った独立レビューを、共通化の後にも置く考えです。[前作のレビュー](https://qiita.com/ishizakahiroshi/items/0ad648d4bf28b11739fe)

## 次は、誰の名義で返事が届くかを確かめます

できているのは依頼と返信、書き方の合意、構成と使い方の整理、起こし方の調査です。受付の土台はGitHubを読むところまでで、書き込みはまだです。

受け箱は非公開で作り、自分で使いながら直す方針です。まず、本人名義では返事をする相手が、新しいbot名義にも応答するかを確かめます。恒久的に使うと分かってから、OSSにするかを考えます。

## こんなときに刺さります

複数のAIとの作業を並べたい方へ、many-ai-cli。「複数の AI コーディング CLI を並列で走らせ、承認をブラウザ 1 タブに集約。スマホからでも。」[公開repo](https://github.com/ishizakahiroshi/many-ai-cli)

連絡の続きを追いたい方へ、Deskly。「誰の返事待ちか、次に何をするか。」一般向けの製品はまだ未公開です。公開repoへのリンクは冒頭に置いています。

気になるものがあれば、Starを付けてもらえるとうれしいです。

本文と画像はdotsが制作し、事実の整理・手元での独立検収・媒体への公開は手元が担当する分担です。制作の提出、手元の受入、公開は別の工程です。

ヘッダー画像とインフォグラフィックの絵、本文の挿絵はAI（OpenAIの画像生成）で作成し、文字と図解は別に組んでいます。

書いた人：ishizakahiroshi。群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）です。[個人サイト](https://ishizakahiroshi.com/)、[GitHub](https://github.com/ishizakahiroshi)

バックエンド・インフラ・AI連携の、フルリモートでの業務相談を受け付けています。[相談先のX](https://x.com/ishizakahiroshi)
