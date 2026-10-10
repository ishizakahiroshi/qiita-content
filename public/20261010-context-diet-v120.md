---
title: Claude Code の手順書を出そうとしたら、3週間前に出した版がもう古くなっていた
tags:
  - ClaudeCode
  - Claude
  - AI
  - LLM
  - プロンプト
private: false
updated_at: '2026-10-10T13:36:27+09:00'
id: ce0b03506c6455599946
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![古い手順書と新しい道筋を照らす机。context-diet 1.2.0](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20261010-context-diet-v120_01_hero.png)

3週間前に Claude Code 2.1.278 に合わせた手順書を、2.1.295 で見直したら、確認用コマンドまで古くなっていました。常駐 context と CLAUDE.md を棚卸しする自作スキルを直し、`/doctor prompt-audit` を足して、1.2.0 として出した記録です。

自作で claude-code-context-diet という Claude Code の常駐 context を棚卸しする手順スキルを公開しています。Claude Code の常駐 context を実測し、削る価値を判定してから減らす手順集と簡易スキル。効くキーを実測で見極めた実践知。

何ができるかの紹介ページ:
https://ishizakahiroshi.com/work.html?id=claude-code-context-diet

リポジトリ（Star をいただけると励みになります）:
https://github.com/ishizakahiroshi/claude-code-context-diet

入れるときは、Claude Code で次の2行を実行します。

```text
/plugin marketplace add ishizakahiroshi/claude-code-context-diet
/plugin install context-diet@ishizakahiroshi
```

再起動して「コンテキスト棚卸して」と頼むと使えます。直接呼ぶなら `/context-diet:context-diet` です。既に入れている人は次の2行で更新します。

```text
/plugin marketplace update ishizakahiroshi
/plugin update context-diet@ishizakahiroshi
```

導入・更新の根拠は1.2.0の README です。
https://github.com/ishizakahiroshi/claude-code-context-diet/blob/e39049fcfb0d05a8aa8ba56452607b360d5aab04/README.md

前回の記事: 2026-06-22「Claude Code の常駐 context を 14% 削った話。disabledTools は存在しなかった」。
https://zenn.dev/ishizakahiroshi/articles/20260622-claude-code-context-diet

![2.1.278から2.1.295への変化をCHANGELOGと照合し、記述を直して1.2.0として配布。実測値は2.1.278のまま](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20261010-context-diet-v120_02_infographic.png)

2.1.278から2.1.295までの変化を CHANGELOG と照らし、説明と確認手順を直して1.2.0として配りました。削減量の実測値は2.1.278で測ったままで、今回は測り直していません。

## 出そうとした版は、もう出ていました

2026-10-10、Claude Code に「最近新しくしたのでリリースしようと思う。全体的にチェックして」と頼みました。ところが1.1.0は9月21日に version 込みで push 済みで、利用者にはもう届いていました。

機械的な検査は通っていましたが、配布物は2.1.278、手元は2.1.295です。公開 CHANGELOG、公式 docs、実行ファイルの設定説明を照合しました。
https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md

下調べは別の AI エージェントに任せ、主要項目は原文と実行ファイルの文字列で再確認しました。

## 「0件」の理由が、キーの不在ではありませんでした

1.1.0の `rg` コマンドは、実行ファイルから設定の説明文を取り出します。2.1.295では短縮変数名が変わり、1本目で `skillOverrides` が0件になりました。2本目では、`disableWorkflows` の説明が635文字に伸びていて、検索の400文字上限から漏れました。

![古い地図と新しい地図を照らし、検索結果が空になった理由を確かめる手元](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20261010-context-diet-v120_03_illustration.png)

どちらも実在するキーです。空の検索結果をそのまま「存在しない」と読める手順を、自分で配っていました。

1.2.0の修正版です。PowerShell と ripgrep を使い、`$rg` に実際のパスを入れます。短縮変数名に依存せず、引用符は両方拾います。
https://github.com/ishizakahiroshi/claude-code-context-diet/blob/e39049fcfb0d05a8aa8ba56452607b360d5aab04/skills/context-diet/references/settings-keys-reference.md

```powershell
$rg = '<rg.exe のフルパス>'
$exe = "$env:USERPROFILE\.local\bin\claude.exe"
$pat = '(skillOverrides|enableWorkflows|disableWorkflows|enableArtifact|disableArtifact|disableBundledSkills|disableClaudeAiConnectors):.{0,160}?describe\((?:"[^"]{0,1200}"|''[^'']{0,1200}'')\)'
& $rg -a -o --no-filename $pat $exe | Sort-Object -Unique
& $rg -a -o --no-filename 'disabledTools|enabledTools|maxSkillDescriptionChars|skillListingMaxDescChars' $exe | Group-Object | ForEach-Object { "{0} : {1}" -f $_.Name, $_.Count }
```

説明文が返らないだけで、不在とは判定しません。最後の行の検索語を調べたいキー名に替えて数え、0件だったときだけ「存在しない」と書き、確認した版を添えます。

## 入口の判定も変わっていました

![Claude Codeの版ごとに変わった表示、計上、Workflows、1M窓と、再現コマンド・docs URLの修正](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20261010-context-diet-v120_04_fig-changes.png)

1.1.0の記述は、2.1.281・2.1.283・2.1.285・2.1.287の4つの版でそれぞれ古くなっていました。版に結び付かない再現コマンドと docs URL の直しも別にあります。まず `/context` の分母で削る価値を判断するため、1M コンテキストを既定で使う条件の変化が重要でした。

- 2.1.281: claude.ai 同期 skill は、名前が衝突しなければ短い名前で表示されます。`anthropic-skills:` の接頭辞で探す説明を直しました。
https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md#21281
- 2.1.283: MCP server instructions が `/context` の独立行と合計に入ります。claude.ai のコネクタを切る `disableClaudeAiConnectors` の効果は、対話の `/context` ならこの行の差から推定できます。
https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md#21283
- 2.1.285: 独自 `ANTHROPIC_BASE_URL` 経由でも、対応モデルは1M窓を使います。`disableWorkflows` が有効でも Code Review と `/ultrareview` は動きますが、実行マシンの管理者設定は例外です。
https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md#21285
- 2.1.287: Opus 4.7以降と Fable は、Bedrock・Vertex・Foundry・Claude apps gateway でも `[1m]` なしで1M窓が既定になりました。
https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md#21287

200kが残るのは、Opus 4.6 / Sonnet 4.6を `[1m]` なしで使う場合、1M非対応モデル、`CLAUDE_CODE_DISABLE_1M_CONTEXT=1` の場合です。ゲートウェイが200Kで止めるなら、表示が1mでも実際の上限は200Kです。
https://code.claude.com/docs/en/model-config

`disableWorkflows` は非推奨ではなく、全員に効かせるためのキーです。自分だけなら `enableWorkflows` を使います。
https://code.claude.com/docs/en/settings-reference

環境変数の公式ページへのリンクも、404になった旧URLから直しました。
https://code.claude.com/docs/en/env-vars

## 減らす前に、公式の監査も挟みます

2.1.283で加わった `/doctor prompt-audit` を、内容の監査の最初に入れました。セッション内で実行します。
https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md#21283

```text
/doctor prompt-audit
```

対象は CLAUDE.md・CLAUDE.local.md・AGENTS.md と、`.claude/`・`~/.claude/` の rules・skills・commands・subagents・output styles です。範囲を絞る例は `/doctor prompt-audit .claude/skills/deploy` です。古い指示や矛盾への修正案を出し、適用を頼むまでファイルは変わりません。
https://code.claude.com/docs/en/memory

prompt-audit は同梱の `claude-api` skill を通して動くため、`claude-api` を `skillOverrides` で `off` にするか、`disableBundledSkills` で無効にすると使えません。このスキルは使っていない skill を off にする手順を持つので、使用実績が0回でも `claude-api` を off にすると prompt-audit が使えなくなる、という注意を足しました。
https://code.claude.com/docs/en/memory

公式説明に重複と曖昧は挙がっていないため、矛盾・重複・陳腐化・曖昧の4観点の表は残しました。

訂正と追記を承認し、10月10日に1.2.0を push しました。削減量は2.1.295で測り直しておらず、この更新で増減したとは言えません。
https://github.com/ishizakahiroshi/claude-code-context-diet/commit/e39049fcfb0d05a8aa8ba56452607b360d5aab04

## あわせて読みたい

- 「Claude Code の /doctor prompt-audit で指示ファイルを監査したら、読み込まれていない CLAUDE.md が出てきた」
https://qiita.com/ishizakahiroshi/items/7cc143942aab2b98e741

## claude-code-context-diet はこんなときに刺さります

- `/context` を開いたら何もしていないのに常駐が多かった人
- CLAUDE.md や skills が増えて、どれが効いているか分からなくなった人
- settings.json のどのキーが本当に効くのか、実測で知りたい人
- 今回足した `/doctor prompt-audit` と4観点の監査を、一緒に回したい人

冒頭の plugin コマンドで試せます。導入のために設定ファイルを書く必要はなく、入れて再起動して頼むだけです。

何ができるかの紹介ページ:
https://ishizakahiroshi.com/work.html?id=claude-code-context-diet

リポジトリ（Issue / PR 歓迎）:
https://github.com/ishizakahiroshi/claude-code-context-diet

役に立ったら Star をいただけると励みになります。気づいた点は Issue または X の DM へどうぞ。

## おわりに

否定形の事実には版を添える、と書いていたのに、その判定に使うコマンドが古くなっていました。次に本体の版が上がったら、説明と一緒に確認手順も照らすところから始めます。

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
