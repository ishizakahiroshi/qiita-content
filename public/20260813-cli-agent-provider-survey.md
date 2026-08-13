---
title: 対応 AI CLI を増やす前に、自分のツールの利用者数を測った方がいい
tags:
  - AI
  - 個人開発
  - CLI
  - OSS
  - 意思決定
private: false
updated_at: '2026-08-13T09:15:51+09:00'
id: f491815a42b157f15932
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-13_cli-survey_hero.png)

AI コーディング CLI を 20 製品調べました。Kimi、DeepSeek、Devin、Droid、CodeBuddy、Qwen Code、iFlow、Mistral Vibe、Sakana Fugu。中国系も含めて、月額サブスクの有無と無料枠まで洗いました。

結果、1 つも実装しないことにしました。

判断が変わった瞬間ははっきりしていて、候補の利用者数を並べ終えた後に、自分のツールの数字を測っていないことに気づいたときです。

## 自作ツールを持っている方へ

自作で [many-ai-cli](https://github.com/ishizakahiroshi/many-ai-cli) という Web ダッシュボードを作っています。複数の AI コーディング CLI を並列で走らせ、承認をブラウザ 1 タブに集約。スマホからでも。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/many-ai-cli

同じ悩みを持っている方は、下記で入ります。

```bash
npm i -g many-ai-cli
```

この話には前があります。前回は Agent Plugins を自作ツールに取り込むか検討して、配布モデルが逆向きだったので見送りました。今回も見送りの話です。

前回の記事: [Agent Plugins を自作ツールに取り込むか検討して、見送りました。配布モデルが逆向きだった](https://qiita.com/ishizakahiroshi/items/000383ea267490d25886)

## 「どれを足すか」から入ったのが間違いだった

現在このツールは Claude Code、Codex CLI、GitHub Copilot CLI、Cursor Agent、opencode、Grok の 6 つに対応しています。7 つ目に何を足すか、という話から始めました。

絞り込みの条件は 5 つ置きました。

- 専用のターミナル CLI があるか（API だけの製品は wrap する実体がない）
- 月額サブスクか無料で使えるか
- 対話 TUI で承認を出すか（出さないならダッシュボードの意味がない）
- AGENTS.md のような指示ファイルを持つか（承認マーカーを注入する先が要る）
- 規約とデータ送信が許容範囲か

この 5 つで 20 製品が 3 つまで減りました。Kimi Code CLI、Droid、Devin CLI。ここまでは順調でした。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-13_cli-survey_fig1.png)

図の左が判定条件、右がそこで落ちたものです。API しか出していない製品が最初の関門で大量に落ち、次に「定額プランを持たない」BYOK 系が落ちます。

## 調べて分かった各社の事情

絞り込みの過程で分かったことを先に置いておきます。同じことを調べる人の手間が減ると思うので。

**DeepSeek には公式のターミナル CLI がありません。** これは意外でした。Deep Code も Reasonix も第三者製で、DeepSeek 側は API ドキュメントで「コミュニティ統合」として案内しているだけです（https://api-docs.deepseek.com/quick_start/agent_integrations/deepcode/ ）。課金も API 従量のみで、月額プランがない。名指しで調べた製品でしたが、wrap する対象そのものが存在しませんでした。

**Qwen Code の無料枠は 2026-04-15 で終わりました。** CLI 自体は今も無料の OSS ですが、終わったのは Alibaba のサーバーで推論を回せた OAuth ログインの方です（https://qwenlm.github.io/qwen-code-docs/en/users/configuration/auth/ ）。今から使うには ModelStudio の Coding Plan か BYOK が要ります。

**Amazon Q Developer CLI は畳まれます。** 新規申し込みが 2026-05-15 に終了、サポート終了が 2027-04-30。後継の Kiro へ移行中です（https://aws.amazon.com/q/developer/pricing/ ）。実装しても寿命が 1 年半しかない。

**GLM と Sakana Fugu は、そもそも分類が違いました。** どちらも専用 CLI ではなく、既存の wrapper の裏に差すバックエンドです。GLM Coding Plan は Anthropic 互換のエンドポイントを持っていて Claude Code に無改造で差さる（https://docs.z.ai/devpack/overview ）。Sakana Fugu は OpenAI 互換 API で、Codex CLI から使う形になります（https://console.sakana.ai/pricing ）。

ついでに無料枠も調べましたが、**GLM Coding Plan（Lite $18/月）にも Sakana Fugu（Standard $20/月）にも無料枠はありませんでした。** 検証するだけでも課金が要ります。Fugu の方は「サブスクに API アクセスが含まれる」と公式に書いてあったのが収穫でした。Mistral Vibe のようにサブスクと API が別請求になっているケースがあるので、ここは製品ごとに確認しないと分かりません。

## 数字を並べたら、比較対象が抜けていた

3 つに絞ったところで、利用者数を調べました。公表しているベンダーはほぼ無いので、npm レジストリの週次ダウンロード数と GitHub のスター数を API から直接取りました。

```powershell
# npm 週次ダウンロード
Invoke-RestMethod -Uri "https://api.npmjs.org/downloads/point/last-week/@moonshot-ai/kimi-code"

# GitHub スター数
Invoke-RestMethod -Uri "https://api.github.com/repos/MoonshotAI/kimi-code" `
  -Headers @{ 'User-Agent'='ps'; 'Accept'='application/vnd.github+json' }
```

2026-08-13 時点の実測です。

| 製品 | npm 週次DL | GitHub スター |
|---|---:|---:|
| Codex CLI（対応済） | 16,311,927 | 105,554 |
| Claude Code（対応済） | 15,100,156 | 141,224 |
| opencode（対応済） | 2,138,728 | 196,571 |
| Copilot CLI（対応済） | 2,004,492 | 非公開リポ |
| CodeBuddy Code | 111,073 | 非公開リポ |
| Qwen Code | 74,279 | 26,948 |
| Kimi Code CLI | 34,997 | 6,456 |
| Droid | 1,463 | 非公開リポ |
| iFlow CLI | 812 | 5,110 |

候補はどれも既存の 100 分の 1 以下でした。「うーん、少ないな」と思いました。

思ってから、10 秒くらい止まりました。

**少ないと言うからには、比較する基準が要る。その基準を自分は持っていない。**

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-13_cli-survey_illustration-pause.png)

並べ終わったところで手が止まりました。表は埋まっているのに、何と比べているのかが自分でも言えない状態でした。

## 自分のツールを測った

測りました。

```powershell
Invoke-RestMethod -Uri "https://api.github.com/repos/ishizakahiroshi/many-ai-cli" `
  -Headers @{ 'User-Agent'='ps' }
Invoke-RestMethod -Uri "https://api.npmjs.org/downloads/point/last-week/many-ai-cli"
```

GitHub スター 6。npm 週次ダウンロード 11。最新リリース v0.6.0 の資産ダウンロードが 15。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-13_cli-survey_fig2.png)

並べてみると、対数スケールにしてもなお自分のツールの棒はほとんど伸びません。候補として悩んでいた製品の方が、こちらより 3 桁以上大きいところに居ます。

これを見て、判断の軸が根本から違っていたことが分かりました。Kimi の週 3.5 万人のうち many-ai-cli に辿り着く人が何人いるか、という掛け算をすると、期待値はゼロ付近です。これは Kimi に限った話ではありません。**どの製品を足しても同じです。** 対応済みの Codex や Claude Code が 1,600 万ダウンロードの母数を持っていても、こちら側は週 11 なのですから。

つまり「候補の利用者数を根拠に採否を決める」という枠組み自体が、この規模では機能しません。数字を並べた表を作った時点では、自分がまともな分析をしている気になっていました。実際には分母を見ていなかっただけでした。

## 残った軸は 2 つだけだった

外部ユーザーの獲得が理由にならないなら、何が理由になるのか。整理したら 2 つしか残りませんでした。

**1 つ目は、自分が実際に使うか。** 個人 OSS で最後まで保守されるのは、作者が日常的に使っている機能だけだと思っています。逆に言えば、ユーザー獲得が期待できない以上、他に理由が存在しない。「興味があるから入れたい」は、この文脈では弱い理由ではなく、むしろ唯一の正当な理由になります。

**2 つ目は、メンテ負債がどれだけ増えるか。** これは軽くありませんでした。provider を 1 本足すと、承認パターンの定義ファイルとモデル一覧の追従対象が 1 本ずつ増えます。うちのプロジェクトはこの 2 つに鮮度チェックの仕組みを持っていなくて、陳腐化しても誰も気づきません。実際 2 日前に、モデル一覧が最新世代を丸ごと欠いたまま放置されていたのを見つけたばかりでした。

追加のコード量自体は大したことがなくて、直近で opencode を足したコミットは 18 ファイル・180 行でした。問題はそこじゃない。**書いた後に、誰も見ないまま腐る場所が 1 つ増えることの方が重い。**

## 使ってから実装を決める

第一候補だった Kimi Code CLI は、条件で見ると本当に良かったんです。MIT ライセンス、AGENTS.md がある、Windows の公式手順がある、第三者ツールからの利用を公式が許諾している（https://github.com/MoonshotAI/kimi-code ）。月額 $19 から。伸びも加速していて、月次ダウンロード 100,817 に対して週次 34,997 なので、直近の週が過去平均より速い。

それでも実装は見送りました。順序を逆にすべきだと思ったからです。

実装してから使うか決めるのではなく、**使ってから実装を決める。** 素で入れて何週間か普通に使ってみて、日常の並列運用に入るなら実装する。触って飽きたなら実装しない。この順序なら無駄になるのは課金分と数時間だけで、腐るコードは 1 行も生まれません。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-13_cli-survey_illustration-order.png)

先に道具を出して触ってみる。設計図を広げるのはその後でいい、という順序です。

逆の順序が一番損です。承認パターンだけが残って、半年後に「これ何のために追従してるんだっけ」となる。前回の Agent Plugins のときも同じ結論でしたが、あのときは「配布モデルが逆」という構造的な理由でした。今回は「そもそも判断軸を間違えていた」という、もっと手前の話です。

## many-ai-cli はこんなときに刺さります

- Claude Code や Codex を同時に何本も走らせていて、どれが承認待ちか分からなくなる人
- 承認のたびにターミナルを行き来していて、そのたびに集中が切れる人
- 席を外している間に AI が止まっていて、戻ったら何も進んでいなかった経験がある人

いずれかに心当たりがあれば、`npm i -g many-ai-cli` で試せます。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/many-ai-cli
- npm: https://www.npmjs.com/package/many-ai-cli

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## あわせて読みたい

- [Agent Plugins を自作ツールに取り込むか検討して、見送りました。配布モデルが逆向きだった](https://qiita.com/ishizakahiroshi/items/000383ea267490d25886)（同じツールで「調べて見送った」判断を書いた前回分。今回はその判断軸そのものを疑った話です）
- [4 つの AI と一緒に、many-ai-cli v0.5.0 を出した話](https://qiita.com/ishizakahiroshi/items/a3f40f08b2185b89c887)（対応 provider を実際に増やしていた頃の記録。増やす側の作業量が分かります）

## おわりに

調査そのものは無駄になっていません。GLM が Claude Code に無改造で差さること、Sakana Fugu のサブスクに API が含まれること、DeepSeek に公式 CLI がないこと。どれも今後の判断で効きます。調べた内容は全部 HTML にまとめて残しました。

ただ、20 製品を比較する表を作っている間、自分は分母を一度も見ていませんでした。実測値を並べているから客観的なつもりでいたけれど、比較の枠組み自体が間違っていた。

数字を出す前に、その数字を何と比べるのかを決める。次からはそこから始めます。

---

※ ヘッダー画像は AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
