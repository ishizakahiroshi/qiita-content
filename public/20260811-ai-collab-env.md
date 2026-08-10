---
title: レガシーシステム刷新を複数人・複数AIで進める共同編集環境の作り方
tags:
  - AIエージェント
  - Git
  - ドキュメント
  - 開発プロセス
  - レガシーシステム
private: false
updated_at: '2026-08-11T04:27:43+09:00'
id: efdfbf830f47181444b9
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![タイトル画像](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_ai-collab-env_hero.png)

あるレガシー業務システムのリプレイスで、複数の担当者と複数のAIエージェントが同じリポジトリを触る環境を作りました。

最初に困ったのは、AIの性能ではありませんでした。誰が何を決めたのか、どの計画が現在地なのか、どこまで確認したら「終わった」と言ってよいのか。その境界が、人間同士でもAI同士でも揃っていなかったことです。

Claude、Codex、GrokのようなAIを増やすだけなら簡単です。CLIを入れて、リポジトリを開いて、指示を渡せば動きます。ただ、それでは共同編集環境にはなりません。人数分の作業速度で、人数分の食い違いが増えるだけでした。

この記事では、既存のレガシー業務システムを段階的に置き換える架空プロジェクトを題材に、環境設定から共同編集の運用までをまとめます。実案件の名称、画面名、URL、内部パス、実日付、正確な件数は使っていません。構造と判断だけを残しています。

## 忙しい人向け（AI 音声解説・14 分）

この記事の音声版を NotebookLM で作りました。移動中・作業中の "ながら聴き" にどうぞ。

https://youtu.be/dQt7TjLf8eo

## TL;DR

先に全体像を置きます。

- グローバル規約、リポジトリ規約、作業MDの三層に分ける
- `AGENTS.md` は入口、`CLAUDE.md` はリポジトリ正典、skillは条件付きの手順書にする
- 作業は `plan / bugfix / pending` のMarkdownへ寄せ、安定したWork IDを付ける
- 大きな計画はC1、C2のような実行単位へ分ける
- MDを作ったAIと、実装・レビュー・検証をしたAIを別に記録する
- AIの製品名、runtime、model、人間の担当者を一つの文字列へ混ぜない
- repo固有台帳がある場合は、global台帳へ二重記録しない
- 「実装済み」「静的検証済み」「画面確認済み」「commit済み」「push済み」を別状態にする
- AI交代時は、前の実行を閉じてから新しいexecutionを始める
- dirty worktreeでは対象ファイルだけをstageし、他人の差分を巻き込まない
- 振り返りと引き継ぎを、チャットではなくリポジトリ相対の文書へ残す

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_ai-collab-env_infographic.png)

この仕組みの中心は、賢いプロンプトではありません。情報の置き場所と、状態の境界です。

## 今回使った土台について

作業MDの生成、残作業の確認、frontmatter、AI execution provenanceには、自作の[docsweep](https://github.com/ishizakahiroshi/docsweep)を使っています。AIが量産する作業ログMarkdownを、腐らせず自動で片付けるためのクロスプラットフォームCLIです。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=docsweep
- リポジトリ: https://github.com/ishizakahiroshi/docsweep

この記事の考え方はdocsweepがなくても使えます。Markdown、CSV、Gitだけでも組めます。ただし、毎回同じfrontmatterを作り、状態を検査し、複数プロジェクトを横断して探すところはCLIへ寄せたほうが楽でした。

## 「AIを追加する」と「共同編集環境を作る」は別だった

最初は単純に考えていました。

人間が三人、AIが三種類なら、得意な相手へ仕事を振り分ければ速くなる。画面調査はブラウザ操作に強いAI、コード実装はコードベースを読めるAI、レビューは別のAI。いかにも効率が良さそうです。

実際、単発の速度は上がりました。問題は、その次です。

- AI Aが作ったplanをAI Bが読んだが、どこまで実装済みか分からない
- 人間の担当者名と、AIの実行主体が同じowner欄へ入っている
- 「検証済み」と書いてあるが、静的テストなのかブラウザ確認なのか分からない
- 同じ作業をglobal台帳とrepo台帳へ二重に記録している
- 別のAIがMDを更新し、最初に作ったAIの情報まで上書きした
- commitは済んでいるがpushされていないのに「完了」と報告された
- 画面のスクリーンショットはあるが、どの仕様・状態・操作の証拠なのか結びつかない

どれも、コードのバグではありません。共同編集の座標系がないことによる事故でした。

正直、ここで一度かなり迷いました。規約を増やせばAIが賢くなるように見えます。でも、巨大な一枚の指示書へ全部を書いたら、今度は誰も読み切れません。AIも人間も同じです。

そこで、規約を増やす前に、置き場所を分けました。

![複数のAIが同じ机へ書類を積み上げ、どれが最新版か迷っている場面](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_ai-collab-env_illustration-1.png)

AIの数が問題なのではなく、同じ書類に違う意味で書き込んでいたことが問題でした。

## 規約は三層に分けると読める量になる

共同編集環境の規約は、次の三層に分けました。

```text
global rules
  ├─ OSやshellの使い方
  ├─ commit、push、公開の承認境界
  ├─ 共通のMarkdown作成規約
  └─ 共通skillへの入口

repository rules
  ├─ ドメイン固有の不変条件
  ├─ 認証、テナント、データ境界
  ├─ repo固有の台帳とvalidator
  └─ global規約との委譲境界

work document
  ├─ 今回の目的と対象外
  ├─ C1、C2などの実行単位
  ├─ 変更予定ファイル
  ├─ 検証条件
  └─ 引き継ぎと証跡
```

![global、repository、work documentの三層責務図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_ai-collab-env_fig1.png)

上の層ほど共通で、下の層ほど具体的です。全リポジトリへ効くshellルールを各repoへコピーせず、ある業務システムだけに存在する画面台帳をglobalへ持ち出しません。

### globalへ置いたもの

global規約へ置いたのは、どのプロジェクトでも変わらない作業の癖です。

- WindowsではPowerShell 7かGit Bashを使う
- WSLを作業中に混ぜない
- `.sh` はGit Bashのフルパスで実行する
- Node系のバックグラウンド起動は`.cmd`経由にする
- 新しい作業MDはgeneratorから作る
- commit、push、deploy、公開は別の状態として扱う
- managedなガイダンスは生成物を直接直さず、生成元を直す
- secretを持つファイルを公開成果物の素材として読まない

ここには、特定の業務システム名や画面名を書きません。fresh cloneでも意味が通る内容だけにします。

### repositoryへ置いたもの

repo規約には、そのシステムを壊さないための不変条件を置きました。

たとえば、次のような内容です。

- 組織階層を飛び越えるデータ参照を作らない
- 認証を独自実装しない
- AI処理を自社サーバーへ持ち込まない
- 画面仕様、状態、証跡、作業の正典台帳を明示する
- 人間の担当者を安定keyで解決する
- repo独自のAI実行台帳がある場合は、そのvalidatorへ委譲する

これらをglobalへ置くと、別のリポジトリへ意味のない制約が漏れます。逆に、repo内へshellやcommitの共通ルールを全部コピーすると、数か月後には表現がずれます。

### work documentへ置いたもの

作業MDは、その仕事だけの契約です。

「レガシーシステムを刷新する」のような大きな目的だけでは、AIは動けません。対象画面、調査方法、変更範囲、検証方法、停止条件まで落とします。

この層は短命です。だからこそ、完了したらarchiveへ移せる形式にします。恒久的な設計判断は、作業MDの中へ閉じ込めず、reference文書や画面仕様へ戻します。

## AGENTS.mdは入口、CLAUDE.mdは正典にした

AIツールごとに、最初に読むファイル名が違います。あるツールは`CLAUDE.md`を読み、別のツールは`AGENTS.md`を読みます。

両方へ同じ長文をコピーすると、ほぼ確実にずれます。

そこで、`AGENTS.md`は薄い入口にしました。

```markdown
# Agent Entry Point

このリポジトリの運用ガイダンスは CLAUDE.md を正本とする。

- プロジェクト概要・ルール: ./CLAUDE.md
- 恒久文書: ./docs/
- ローカル作業記録: ./docs/local/

矛盾した場合は CLAUDE.md を優先する。
```

`CLAUDE.md`には、プロジェクト固有の不変条件だけを書きます。個人の話し方、OS全体のshellルール、特定AI製品の一時的な制約はglobalへ残します。

この形の利点は、AIの種類が増えても入口だけ追加すればよいことです。正典の本文を複製しません。

## skillは「常時読む規約」ではなく「条件付きの手順書」にした

CLAUDE.mdへ何でも書くと、毎ターン読む量が増えます。そこで、長い手順はskillへ分けました。

skillに向くのは、次の三条件を満たす作業です。

- 繰り返し発生する
- 判断や複数ステップを含む
- 手順化で時間短縮か事故防止になる

たとえば「作業MDを作ったAIと実行AIを記録する」はskillに向きます。managerの判定、metadataの解決、executionの開始と終了、AI交代、完了前checkまで複数ステップがあるからです。

一方、junctionのパス解決バグのような内部実装の詳細は、利用者向けskillへ増やしません。コードと回帰テストで塞ぎます。Gitの一部stageで一度だけ踏んだ細かい罠も、再発するまでは新skillにしません。

常時読む規約と、必要なときだけ開く手順書を分ける。これは思った以上に効きました。

## Windows環境はshellを混ぜないところから始めた

共同編集では、同じコマンドが同じ意味で動くことが大事です。Windowsではここが意外に難しい。

PowerShell、Git Bash、WSLは、それぞれ引用符、環境変数、パス、改行の扱いが違います。人間一人なら頭の中で切り替えられても、複数AIが交代すると事故になります。

今回の基本ルールは単純です。

```text
PowerShell用コマンド  -> PowerShell 7
.shスクリプト         -> Git Bashの実体を明示
WSL                    -> 自動では起動しない
Nodeの前面実行         -> pnpm など通常コマンド
Nodeの背面実行         -> pnpm.cmd または cmd.exe /c
```

なぜWSLを避けたかというと、同じ`bash`という名前でWindowsのWSL launcherが起動することがあるからです。Git Bashを使うつもりでWSLが起動すると、ドライブの見え方、改行、スクリプトの挙動が変わります。

記事用の一般形では、次のようにフルパスを固定します。

```powershell
& "C:\Program Files\Git\bin\bash.exe" ./scripts/check.sh
```

バックグラウンドでNode系コマンドを起動するときも、PowerShell shimをそのまま渡しません。

```powershell
Start-Process `
  -FilePath "cmd.exe" `
  -ArgumentList "/c", "pnpm.cmd", "dev", "--host", "127.0.0.1" `
  -WorkingDirectory "<project-dir>" `
  -WindowStyle Hidden
```

環境構築というと、パッケージのインストール一覧を書きたくなります。でも、共同編集では「どのshellで動かすか」のほうが先でした。

![PowerShell、Git Bash、WSLの分岐標識を前に立ち止まる開発者](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_ai-collab-env_illustration-2.png)

コマンドそのものより、実行面を固定する。ここを曖昧にすると、同じ失敗が別AIで何度も戻ってきます。

## 作業MDは手書きせずgeneratorから作る

共同編集の作業単位は、`plan_*.md`、`bugfix_*.md`、`pending_*.md`へ寄せました。

ただし、空のMarkdownを手で作りません。作成時点で必要なfrontmatterが揃うgeneratorを使います。

```bash
python -m docsweep new plan legacy-screen-inventory
python -m docsweep new bugfix duplicate-save-request
python -m docsweep new pending external-api-decision
```

generatorを通す理由は、単なる時短ではありません。

- 種別と状態が揃う
- dueを自動計算できる
- 類似の現役文書を検出できる
- Work IDを採番できる
- 作成AIのsnapshotを固定できる
- 作業queueの置き場所をrepo設定から解決できる

手書きすると、`due`だけある、ownerが空、状態が二重管理、関連文書がない、といった小さな差が増えます。最初は小さく見えても、数十本になると検索と集計が壊れます。

## frontmatterは「機械が読む最小契約」にした

作業MDのfrontmatterには、文章ではなく機械的に照合したい情報を置きます。

```yaml
---
type: plan
status: draft
docsweep_state: planned
tags: [replacement, inventory]
owner: <owner>
review_status: draft
related: []
last_reviewed: <today>
due: <due-date>
work_id: WK-EXAMPLE-001
ai_author_agent: codex
ai_author_runtime: codex-cli
ai_author_provider: openai
ai_author_model_id: unknown
ai_author_model_source: unavailable
ai_execution_refs: [AIX-EXAMPLE-001]
---
```

`status`と`docsweep_state`を分けているのは、文書自体のライフサイクルと、作業の進行状態を混ぜないためです。

ownerは人間の主担当です。AI製品名を入れません。AIの情報は`ai_author_*`とexecution台帳へ分けます。

ここで大事なのは、値が分からないときの扱いでした。分からないmodel IDを、表示名や過去の傾向から組み立てません。`unknown`と`unavailable`を正しく使います。

空欄を埋めることより、嘘を入れないことを優先しました。

## planをC単位へ分けると、引き継ぎが具体になる

大きなplanは、C1、C2、C3のようなcontextへ分けました。

```markdown
## context配分

| C | 内容 | 種別 | AI実行 |
|---|---|---|---|
| C1 | 現行画面と操作の棚卸し | investigation | |
| C2 | 新システムの状態と機能へ対応付け | design | |
| C3 | 実装と自動テスト | implementation | |
| C4 | ブラウザ証跡と受入確認 | verification | |
```

「レガシーシステムを調査する」では広すぎます。「現行画面を棚卸しし、操作と状態を対応付ける」まで落とすと、開始条件と完了条件が書けます。

C単位には、次の項目を持たせます。

- 目的
- 入力
- 変更対象
- 対象外
- 完了条件
- 検証方法
- 手動確認が必要な箇所
- 停止条件
- 次のCへの受け渡し

これでAI交代時の言葉が変わりました。

「続きやって」ではなく、「C2は完了、C3は未着手。C2の証跡はこのID。次はこのファイルを読む」と渡せます。

![大きな設計図をC1からC4のカードへ分割し、チームで受け渡す場面](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_ai-collab-env_illustration-3.png)

大きな計画を細切れにするのが目的ではありません。完了の意味を、引き継げる大きさへ落とすのが目的です。

## 作成AIと実行AIを分けるまで、履歴は毎回少し嘘だった

最初は、作業MDに`model`欄を一つ置けば十分だと思っていました。

足りませんでした。

planを作ったAIと、実装したAIと、レビューしたAIが違うからです。さらに、同じAI製品でもruntimeが違い、表示名しか分からないことがあります。人間の操作担当も別軸です。

そこで、次を分離しました。

```text
owner_key        人間の主担当者
actor_key        そのAI実行を操作した人間
agent            codex / claude / grok / gemini / other / unknown
runtime          codex-cli / claude-code / many-ai-cli / other / unknown
provider         openai / anthropic / xai / google / other / unknown
model_id         runtimeが報告したexact ID。不明ならunknown
model_display    UIなどに出た表示名。不明ならunknown
reasoning        reasoning設定。不明ならunknown
model_source     orchestrator / runtime / cli / ui / user-reported / unavailable
```

人間とAIを同じowner文字列へ詰め込まない。製品名とmodel名を一つに連結しない。取得できない値を推測しない。

この分離は地味ですが、後から「誰の判断だったか」を調べるときに効きます。

## MD作成時のsnapshotは後から上書きしない

新しいwork MDを作ったら、作成時点のAI情報を`ai_author_*`へ固定します。

その後、別のAIが実装しても`ai_author_*`は変えません。実装AIは新しいexecutionとして追記します。

```text
work: WK-EXAMPLE-001

AIX-EXAMPLE-001
  role: authoring
  agent: codex
  result: completed

AIX-EXAMPLE-002
  role: implementation
  context: C1
  agent: claude
  result: completed

AIX-EXAMPLE-003
  role: verification
  context: C1
  agent: codex
  result: completed
```

この形なら、「最初の計画を書いたAI」と「検証したAI」を両方残せます。

![authoring、implementation、review、verificationを追記する実行履歴図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_ai-collab-env_fig2.png)

作成情報はsnapshot、作業実体はappend-onlyのexecution。この二つを分けると、AI交代を履歴破壊なしで扱えます。

## executionは変更前に開始し、終わったら必ず閉じる

docsweep管理のプロジェクトでは、実装前にexecutionを開始します。

```bash
python -m docsweep provenance start \
  --path docs/local/plan_legacy-screen-inventory.md \
  --context C1 \
  --role implementation \
  --agent codex \
  --runtime codex-cli \
  --provider openai \
  --model-id unknown \
  --model-source unavailable \
  --json
```

返されたAIX IDを、作業が終わったときに閉じます。

```bash
python -m docsweep provenance finish \
  --execution <AIX-ID> \
  --result completed \
  --evidence-ref <test-result> \
  --evidence-ref <review-result> \
  --json
```

最後にMDと台帳の整合を確認します。

```bash
python -m docsweep provenance check \
  --path docs/local/plan_legacy-screen-inventory.md \
  --json
```

実装、レビュー、検証は別executionです。同じ行のroleを書き換えません。

作業を途中でやめた場合も、開きっぱなしにせず`partial`、`failed`、`cancelled`のどれかで閉じます。後任AIは新しいexecutionを開始します。

ここを曖昧にすると、「startedのまま三日前から残っている行」が、実行中なのか放置なのか分からなくなります。

## repo固有台帳があるならglobalへ重ねない

すべてのプロジェクトが同じ証跡構造とは限りません。

小さな個人リポジトリなら、globalのCSV台帳で十分です。一方、画面、機能、状態、証跡、担当者を既に台帳化している大きなリプレイスでは、repo固有の実行台帳とvalidatorがあります。

そこで、managerを設定で切り替えます。

```yaml
provenance:
  enabled: true
  manager: repo
  delegate_skill: project-doc
```

`manager: repo`なら、global skillやdocsweepはrepo固有手順へ委譲します。汎用台帳へ同じexecutionを追加しません。

```yaml
provenance:
  enabled: true
  manager: docsweep
  ledger: provenance/ai-executions.csv
  actor_key: your-key
```

`manager: docsweep`なら、共通CLIと個人台帳を使います。

この境界がないと、一つの実行に二つのAIX IDが生まれます。片方だけ更新され、どちらが正典か分からなくなる。記録を増やしたのに、信頼性が下がります。

正典は一つ。委譲先も一つ。これは譲れない線にしました。

## 人間の担当者もGitの表示名から推測しない

AI provenanceを整えたあと、人間のidentityにも同じ問題があると気づきました。

`git config user.name`は表示名です。端末やリポジトリによって変わります。日本語名、GitHubのアカウント名、会社PCの短縮名が混在することもあります。

そこで、開発者registryに安定keyを持たせました。

```csv
actor_key,display_name,github_numeric_id,github_login,status
alice,Alice,<numeric-id>,alice-example,active
bob,Bob,<numeric-id>,bob-example,active
```

実際の値を記事へ持ち込まないため、ここでは説明用の一般名にしています。

解決時はGitHubの数値IDのような安定した値を優先します。表示名だけで推測しません。未登録、矛盾、曖昧があれば、作業MDを作る前に止めます。

ownerとactorも分けます。

- `owner_key`: そのworkの主担当者
- `actor_key`: そのAI executionを操作した人間

別の担当者のplanを代理で準備することはあります。その場合、ownerとactorが違うのは正常です。

## 画面刷新ではスクリーンショットだけを証跡にしない

レガシーシステムのリプレイスでは、画面調査が大きな比重を占めます。

ここでも、画像ファイルをフォルダへ置くだけでは足りませんでした。

スクリーンショット一枚から分からないことがあります。

- 誰の権限で開いた画面か
- どの操作の前後か
- 画面全体か、モーダルか、エラー状態か
- DOM上にあった文字や属性は何か
- どの画面仕様、機能、状態へ対応するか
- いつの時点の現行システムか

そこで、画面の証跡を三点セットにしました。

```text
PNG             見た目
DOM snapshot    構造と文字
visual metadata 権限、状態、取得条件、対応ID
```

さらに、証跡台帳で画面ID、状態ID、機能ID、Work IDへ接続します。

```csv
evidence_id,screen_id,state_id,work_id,role,artifact_path,status
EV-EXAMPLE-001,SCR-EXAMPLE-001,STATE-EXAMPLE-001,WK-EXAMPLE-001,user,<path>,accepted
```

この例も説明用です。実案件のIDやパスではありません。

画像だけを見る運用から、証跡を参照できる運用へ変えると、レビューの質問が変わります。

「このスクショは何ですか」ではなく、「この状態の証跡は揃っていますか」と聞けます。

![画面画像、DOM、メタデータの三枚を一つの証跡カードへ束ねる場面](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_ai-collab-env_illustration-4.png)

見た目を保存するのではなく、判断を再現できる材料を保存します。

## 画面、機能、状態、証跡、workをリンク台帳でつなぐ

リプレイスでは、一つの画面に複数機能があり、一つの機能が複数画面へ現れます。状態も通常、空、入力中、確認、完了、エラーと分かれます。

Markdown同士のリンクだけで全部を表すと、逆引きが難しくなりました。

そこで、関係をCSVへ分けます。

```text
screen-catalog.csv      画面の一覧
screen-functions.csv    画面と機能の多対多
screen-evidence.csv     画面状態と証跡
work-links.csv          Work IDと各正典ID
ai-executions.csv       Work IDとAI実行
```

本文の設計説明はMarkdown、繰り返し検索する関係はCSV。この分担です。

たとえば、`work-links.csv`から一つのworkに関係する画面、機能、証跡、executionを引けます。画面側から、その仕様変更がどのworkで行われたかも逆引きできます。

ここで注意したのは、CSVを万能データベースにしないことでした。

判断理由、例外、読み方までセルへ詰め込むと、人間が読めません。CSVはリンクと状態の正規化に使い、長い説明は正典Markdownの節へ戻します。

## 「完了」を一語で扱うのをやめた

AIとの共同作業で一番危なかった言葉は、「完了」でした。

完了には、少なくとも次の段階があります。

```text
実装完了
  ↓
静的検証済み
  ↓
ブラウザまたは実機確認済み
  ↓
手動受入済み
  ↓
commit済み
  ↓
push済み
  ↓
deploy済み
  ↓
本番受入済み
  ↓
archive済み
```

これらを一つの`done`へ潰すと、「テストは通ったが画面は見ていない」「commitはあるがremoteにない」「deployしたが利用者確認はまだ」が見えなくなります。

作業MDには、今回どこまでを完了条件にするか明記します。

```markdown
## 完了条件

- 実装: 対象ファイルの変更が完了
- 静的検証: unit testとvalidatorが成功
- 画面確認: 対象状態をブラウザで確認
- Git: commitは別承認、pushも別承認
- 本番: 今回のscope外
```

scope外を明記することも大事です。全部やらないことを、未完了と混同しなくなります。

## 親子planのcloseoutはread-only checkから始める

大きな刷新planは、複数の子planへ分かれます。子が実装済みになったからといって、親をすぐ完了へ変えません。

最初にread-onlyのcloseout checkを実行します。

```bash
python -m docsweep closeout-check \
  --path <parent-plan> \
  --json
```

checkでは、少なくとも次を分けます。

- 機械的なblocker
- 人間の手動確認
- dirty worktreeと変更予定ファイルの重複
- 子planと親planの状態不一致
- 検証証跡の不足

checkが緑でも、勝手にH1やfrontmatterを完了へ変更しません。人間の承認後に、子から親の順で閉じます。archiveも別のdry-runと別承認にします。

![実装、検証、承認、push、archiveを別ゲートで表す図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_ai-collab-env_fig3.png)

状態を細かくしたのは管理を重くするためではありません。戻せない操作の前だけ、確認点をはっきりさせるためです。

## dirty worktreeを「掃除」しない

複数人、複数AI、複数セッションで作業すると、リポジトリがcleanとは限りません。

このとき一番やってはいけないのは、目の前の作業をきれいにするために、他人の差分を戻すことです。

避ける操作を明確にしました。

- `git reset --hard`を使わない
- `git checkout -- <file>`で他人の変更を戻さない
- mixed worktreeで`git add -A`を使わない
- 差分の所有者が分からないまま整形を全体へかけない
- commit対象外のファイルを「ついでに」直さない

対象ファイルを明示してstageします。

```bash
git add -- path/to/file-a path/to/file-b
git diff --cached --check
git diff --cached --stat
git diff --cached --name-only
```

同じファイルに他人の差分と自分の差分が混ざっている場合は、ファイル単位のstageでも足りません。indexへ適用するpatchを作り、staged diffを読み直します。

これは地味で、面倒です。でも、共同編集では速さより所有境界が先です。

![散らかった作業机で、自分の二枚だけを選んで封筒へ入れる場面](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_ai-collab-env_illustration-5.png)

dirty worktreeは異常ではありません。他人の作業が存在するという状態です。

## commitとpushの権限は作業開始時に決める

AIへ「実装して」と頼んだとき、commitやpushまで含むかは人によって解釈が違います。

そのため、作業開始時に外部書き込みの境界を決めます。

```text
read-only inspection  自由に実行してよい
local file edits      依頼scope内で実行してよい
tests                 安全な範囲で実行してよい
commit                明示指示が必要
push                  明示指示が必要
PR                    明示指示が必要
deploy                環境ごとの明示指示が必要
production mutation   個別の明示承認が必要
```

今回の環境では、ユーザーが「commit pushして」と言った時点で初めてremoteへ反映します。

push前には、次を確認します。

- branchとupstream
- localとremoteのahead/behind
- staged file一覧
- staged diff
- relevant tests
- remote URL
- push後のlocal HEADとremote branch HEADの一致

「pushコマンドが成功した」だけで終わらず、remoteのcommit hashまで確認します。

## AIの完了報告は、証拠の種類まで書く

完了報告は、成果と検証と残作業を分けます。

悪い例は短い。

```text
完了しました。
```

良い例は少し長くなります。

```text
作業MDの生成とAI execution台帳の同期を実装しました。
focused testと全体testは成功し、provenance checkはvalidです。
local commitは作成済み、remote pushは未実施です。
ブラウザ受入は今回のscope外です。
```

これなら、次の担当者が「何を信じてよいか」を判断できます。

以前、AIの「完了しました」を受け取り、台帳の行数を数えたら不足していたことがありました。それ以来、完了報告は主張ではなく、検査可能な索引として扱っています。

## managed guidanceは生成物を直接直さない

globalなAIガイダンスを複数ツールへ配ると、また正典の問題が出ます。

Claude側は中央guidanceをimportし、Codex側は同じ本文をmanaged blockへ展開する。こうした違いを手作業で維持すると、片方だけ古くなります。

そこで、本文を生成する関数を正典にしました。

```text
generator source
  ├─ central guidance for tool A
  ├─ inline managed block for tool B
  └─ preview / dry-run
```

文面を変えるときはgeneratorを直し、guidance versionを上げ、日英の契約テストを追加します。生成済みのglobalファイルは直接編集しません。

再injectの前後では、managed block外の内容が同じか確認します。

ここでWindowsの改行コードにやられました。

managed blockだけを置換したつもりでも、ファイル全体を通常のtext APIで読み書きすると、LFがCRLFへ変換されることがあります。文字としては同じでも、バイト列は変わります。共同編集環境の「管理外領域を保持する」という契約には違反です。

修正は、既存の改行を読み取り時に維持し、新しいmanaged blockだけ同じ改行へ合わせる形にしました。hash比較ではmanaged block内の改行だけ正規化します。

LFとCRLFの両方を回帰テストへ追加しました。

これは想定外でした。ガイダンスの文章を一節足すだけの作業で、改行保持のバグまで出てきた。こういうところが実装です。

## PowerShellのセミコロンはデータではなく命令だった

証跡IDを複数渡すとき、最初はセミコロン区切りの一文字列にしました。

```powershell
--evidence-refs test;review;browser
```

PowerShellでは、セミコロンはコマンドの区切りです。意図した一つの値になりません。

CLI側を、繰り返しオプションへ変えました。

```powershell
--evidence-ref test `
--evidence-ref review `
--evidence-ref browser
```

アプリ側の区切り文字と、shell側の制御文字を同じにしない。CSV内部でセミコロンを使えることと、CLI引数にそのまま書けることは別でした。

## junctionでは物理パスが正しいとは限らない

作業queueを別の場所へ置き、リポジトリからjunctionで参照する構成も使いました。

ここで`Path.resolve()`を使うと、リポジトリ内の正しい作業MDが、物理的にはリポジトリ外へあるように見えます。「対象ファイルがproject外です」と拒否されました。

セキュリティ上、resolveして範囲確認するのは正しそうに見えます。でも、junctionを正式な入口として採用している場合、利用者が指定したlexicalなrepo相対経路も正典です。

対策は、用途を分けることでした。

- ファイル実体へ安全に到達するための物理パス
- work IDや台帳へ保存するためのlexicalなrepo相対パス

何でもresolveすれば安全、ではありません。どのidentityを保存したいのかを先に決めます。

## 台帳とMDは片方だけ成功してはいけない

execution開始では、CSV台帳へ行を追加し、MDのfrontmatterとcontext配分へAIX IDを追記します。

二つのファイルを別々に更新すると、途中失敗で片方だけ残ります。

```text
ledger append success
MD update failed
```

この状態は、あとでcheckしても直し方を判断しにくい。台帳の行を消してよいのか、MDへ復元すべきなのか、一次証跡が必要になります。

そこで、同じlock内で更新し、後段に失敗したら前段を戻すようにしました。lockがプロセス異常で残った場合に備え、stale lockの回復も入れます。

CSVだから軽い、と考えていました。複数プロセスが触る瞬間、立派な共有状態です。

## 振り返りはチャットの最後だけに置かない

長いセッションの最後に、次の五項目で振り返ります。

- 今回の成果
- 学んだこと
- 改善できたこと
- 次にやること
- 引き継ぎ

チャットにも出しますが、リポジトリ相対の`docs/obsidian/`へMarkdownを残します。

中央の個人Vaultへ直接書かず、リポジトリ側の入口を使います。こうすると、記事、設計、作業記録のGit境界が崩れません。別の環境にcloneしても、リポジトリ内の文脈として辿れます。

引き継ぎには、最低限これを書きます。

```markdown
## 引き継ぎ

- 現在地
- 完了したもの
- 未完了のもの
- 次に開くwork MDの絶対パスまたはrepo相対パス
- 次のC
- 再実行してはいけない完了済み作業
- dirty worktreeで触れてはいけない差分
- commit / push / deployの権限境界
```

「次は頑張る」では引き継げません。次に開くファイルと、最初の一手まで書きます。

![夜の作業机で、AIから次のAIへラベル付きのバトンを渡す場面](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_ai-collab-env_illustration-6.png)

セッションは終わっても、workは続きます。チャットの文脈を保存するのではなく、次の行動を保存します。

## この環境での一日の流れ

仕組みを並べるだけでは使い方が見えにくいので、架空の一日へ落とします。

### 朝、残作業を一つ決める

```bash
python -m docsweep brief
```

複数プロジェクトを横断したい場合だけ`cross`を使います。朝から全planを眺めて選びません。現在のprojectで優先度が高い一件を入口にします。

### 作業MDとmanagerを確認する

対象MDのfrontmatter、C配分、完了条件を読みます。`.docsweep.yaml`とglobal configのprovenance設定を確認します。

- `manager: repo`ならdelegate skillへ進む
- `manager: docsweep`なら共通CLIを使う
- 設定が無効なら、勝手に有効化せず報告する

### 変更前にexecutionを開始する

実装なら`implementation`、レビューなら`review`、検証なら`verification`です。

一つのAIが全部を行っても、roleは分けます。別AIなら当然、新しいexecutionです。

### 対象ファイルだけを変更する

作業開始時の`git status`を保存し、自分のscopeと重なるか確認します。無関係なdirty差分は保持します。

### 検証を証跡へする

test名、validator名、ブラウザ証跡ID、レビュー結果を記録します。「確認した」ではなく、再確認できる参照を残します。

### executionを閉じ、checkする

結果を`completed`、`partial`、`failed`、`cancelled`から選びます。その後、MDと台帳をcheckします。

### commitとpushは別に進める

許可がある場合だけ、対象ファイルをstageし、commitし、pushします。remote HEADまで確認します。

### セッションを閉じる

振り返りと引き継ぎを残し、次のwork MDとCを明記します。

この流れにしてから、AIの交代が怖くなくなりました。速くなったというより、途中で止められるようになった感覚に近いです。

## 導入は四段階に分けたほうがよい

ここまでの仕組みを一度に入れると、たぶん失敗します。

おすすめは四段階です。

### 第1段階: 入口と置き場所だけ決める

- `AGENTS.md`から`CLAUDE.md`へ案内する
- `docs/`と`docs/local/`の役割を分ける
- `plan / bugfix / pending`の命名を揃える
- commit、push、deployの権限境界を書く

この段階では複雑な台帳を作りません。

### 第2段階: work IDとC単位を入れる

- generatorからMDを作る
- Work IDを付ける
- context配分を書く
- 完了条件と対象外を書く
- closeout checkを導入する

ここまでで、引き継ぎの品質がかなり上がります。

### 第3段階: AI execution provenanceを入れる

- 作成AI snapshotを固定する
- execution台帳を追記型にする
- implementation、review、verificationを分ける
- model不明時の`unknown / unavailable`を決める
- AI交代のcloseとrestartを決める

実行履歴が必要になってから入れても遅くありません。

### 第4段階: ドメイン正典と証跡をつなぐ

- 画面台帳
- 機能台帳
- 状態台帳
- 証跡台帳
- work-links
- repo固有validator

これはレガシー刷新の規模に応じて導入します。小さなCLIリポジトリへ画面台帳は不要です。

## 逆に、自動化しなかったもの

自動化しない境界も決めました。

- 本番環境への変更
- destructive migration
- 実ユーザーの権限変更
- archive前の最終判断
- 顧客固有情報を含む公開記事への変換
- exact model IDの推測
- dirty差分の自動掃除

自動化できないからではありません。間違えたときの影響が大きく、人間の判断を残したいからです。

AI共同編集環境というと、どこまで自動化できるかへ目が向きます。実際には、どこで止まるかを決めたほうが運用しやすい。

## 導入前のチェックリスト

最後に、別のプロジェクトへ持ち込むときの確認項目をまとめます。

### 環境

- [ ] OSごとのshellを固定した
- [ ] WindowsでWSLを自動起動しない
- [ ] `.sh`の実行面を固定した
- [ ] バックグラウンドプロセスの起動方法を固定した
- [ ] secret保有ファイルを公開素材として読まない

### ガイダンス

- [ ] global、repository、work documentの責務を分けた
- [ ] `AGENTS.md`を薄い入口にした
- [ ] `CLAUDE.md`をrepo固有正典にした
- [ ] managed guidanceの生成元を一つにした
- [ ] 条件付き手順をskillへ分けた

### 作業MD

- [ ] `plan / bugfix / pending`の置き場所が決まっている
- [ ] generatorからfrontmatterを作る
- [ ] Work IDが安定している
- [ ] C単位と完了条件がある
- [ ] 対象外と停止条件がある
- [ ] 親子planのcloseout手順がある

### identityとprovenance

- [ ] ownerとactorを分けた
- [ ] 人間のidentityを表示名だけで推測しない
- [ ] 作成AI snapshotを固定する
- [ ] executionを一実行一行で追記する
- [ ] agent、runtime、provider、modelを分けた
- [ ] exact model不明時は推測しない
- [ ] AI交代時に前executionを閉じる
- [ ] repo managerとglobal managerを二重運用しない

### 証跡

- [ ] 静的testとブラウザ確認を分けた
- [ ] スクリーンショットに状態と権限のmetadataがある
- [ ] DOMとvisual metadataを必要に応じて保存する
- [ ] 証跡IDから画面、状態、workを引ける
- [ ] 完了報告に残作業と未確認事項がある

### Gitと公開

- [ ] mixed worktreeで`git add -A`しない
- [ ] staged file一覧を確認する
- [ ] commit、push、PR、deployを別権限にした
- [ ] push後にremote HEADを確認する
- [ ] 振り返りと引き継ぎをrepo相対文書へ残す

## docsweepはこんなときに刺さります

- AIが作るplanやbugfixが増え、どれが現役か分からなくなっている人
- 完了した作業MDを手でarchiveへ移している人
- 複数プロジェクトの残作業を一画面で見たい人
- AIが作った文書と、実装・検証の実行履歴を分けて残したい人

`pip install docsweep`で試せます。

- 紹介ページ: https://ishizakahiroshi.com/work.html?id=docsweep
- リポジトリ: https://github.com/ishizakahiroshi/docsweep

使ってみて「この状態を区別したい」「この台帳へ委譲したい」があれば、IssueでもXのDMでも歓迎です。

## あわせて読みたい

- [AI が「完了しました」と言ってきたけど、CSV の行数を数えたら嘘だった話](https://qiita.com/ishizakahiroshi/items/42a2d9da7d175b9efe84)（完了報告を一次証跡で確認する理由につながります）
- [AIコード監査の「修正済み」が分からない。findingをC管理にして結果報告を作業ハブにした](https://qiita.com/ishizakahiroshi/items/364c1125fa15fc8b9c77)（C単位と証跡を結びつける考え方が近いです）
- [AIが残すplan・bugfixを自動で片付けるdocsweepの始め方](https://qiita.com/ishizakahiroshi/items/17bf7a02efcb8c4718ec)（作業MDの入口を先に知りたい場合はこちらです）

## おわりに

複数のAIを同じリポジトリへ入れる前は、性能の違いばかり見ていました。

実際に困ったのは、誰が作ったか、誰が実行したか、何を確認したか、どこまで外へ出したかが混ざることでした。

規約を三層へ分け、作業をC単位へ落とし、作成AIと実行AIを分ける。証跡とGitの境界を残す。やったことは派手ではありません。

それでも、途中でAIを交代できるようになりました。セッションを閉じても、次の一手が消えにくくなりました。

まだ重いところはあります。台帳が増えれば更新コストも増えます。すべての小さなリポジトリに同じ仕組みが必要だとも思っていません。

だから、入口、Work ID、C単位の三つから小さく始めるのがよさそうです。必要になったらprovenanceと証跡を足す。そのくらいで続けていきます。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-08-11_multi-ai-collab-environment/

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
