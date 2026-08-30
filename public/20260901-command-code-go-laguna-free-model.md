---
title: "Command Code Go は月1ドル。無料モデル Laguna S 2.1 の Usage を実測して、自作ダッシュボードに 7 つ目の provider として足した"
tags:
  - CommandCode
  - AIエージェント
  - LLM
  - 生成AI
  - 個人開発
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![ヒーロー（記事トップ）](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_command-code-go_hero.png)

25 回動かして、$0.0000 でした。

入力トークンは 25 万を超えています。それでも Usage の Cost 欄は全部ゼロのまま。使ったのは Command Code の月 1 ドルプランと、そこで `(FREE)` と表示されていた Laguna S 2.1 というモデルです。

ただし、無課金のままでは 1 回も動きませんでした。そこがいちばん引っかかったので、順番に書きます。

この話には前があります。対応する AI CLI を増やす前に、自分のツールが何人に使われているか測った方がいい、という結論で終わった記事です。

前回の記事: [対応 AI CLI を増やす前に、自分のツールの利用者数を測った方がいい](https://qiita.com/ishizakahiroshi/items/f491815a42b157f15932)

測ったうえで、今回は 1 本足しました。

## 自作の many-ai-cli で複数の AI CLI をまとめています

自作で many-ai-cli というローカル Web ダッシュボードを作っています。複数の AI コーディング CLI を並列で走らせ、承認をブラウザ 1 タブに集約。スマホからでも。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/many-ai-cli

同じ悩みを持っている方は、下記で入ります。

```powershell
pnpm add -g many-ai-cli
```

`bun install -g many-ai-cli` でも `npm install -g many-ai-cli` でも同じ registry から入ります。

入れたら 1 回だけ `many-ai-cli setup` を実行します。PATH が通っていない場合は `pnpm exec many-ai-cli setup` で代替できます。

起動後の見え方は OS で違います。

Windows ではデスクトップに「MANY-AI-CLI」のショートカットが 1 個できます。ダブルクリックするとタスクトレイに常駐し、トレイアイコンから「Hub を開く」を選ぶとブラウザで Hub が開きます。止めるときはトレイの「Hub を停止」です。

macOS と Linux では「Many AI Hub Start」「Many AI Hub Stop」の 2 個ができます。Start を押すと黒いコンソールウィンドウとブラウザが一緒に開きますが、そのコンソールが Hub サーバの実体なので、閉じずに最小化してください。Linux（GNOME）では初回だけ `.desktop` を右クリックして「起動を許可」が要ります。

ターミナルから直接起動したい場合は、全 OS 共通で `many-ai-cli serve --open` です。停止は `many-ai-cli stop`。

起動したら、Hub UI 左下の「+ 新しいセッション」から使いたい AI CLI を選ぶところから始まります。

以上の手順は README（https://github.com/ishizakahiroshi/many-ai-cli/blob/main/README.ja.md）に書いてあるものです。

![インフォグラフィック（記事全体の要約）](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_command-code-go_infographic.png)

## 先に結論

2026 年 8 月 30 日時点で、手元で確認できたのは次の 4 点です。

| 確認したこと | 結果 |
|---|---|
| Go プランの月額 | $1（https://commandcode.ai/docs/plans/go） |
| 無課金 + Laguna S 2.1 (FREE) | `insufficient credits` で動かない |
| Go 契約後の Laguna S 2.1 の agent 実行 | Usage の Cost が全件 $0.0000 |
| Command Code 内部の title-gen（DeepSeek） | $0.000362 が計上される |

つまり `(FREE)` は「そのモデルのリクエスト料金が $0」であって、「無課金アカウントでも使える」ではありませんでした。ここは初見でかなり分かりにくいです。

Command Code のモデル構成と料金は頻繁に変わっています。契約前に必ず最新の公式 Pricing を見てください（https://commandcode.ai/docs/resources/pricing-limits）。

## Command Code は何をするもので、Go は月いくらか

Command Code はターミナルで動く AI コーディングエージェントです（https://commandcode.ai/）。プロジェクトディレクトリで起動して、コードを読ませる、バグを調べさせる、直させる、テストを書かせる、Git 差分を説明させる、といった作業をさせます。Claude Code や Codex を触ったことがあれば、だいたい同じ立ち位置だと思ってもらえれば近いです。

Skills、Memory、AGENTS.md、Plan、MCP、Permission 制御あたりは一通りあります。独特なのは taste-1 で、提案を承認したか却下したか、そのあとどう手直ししたかから、その人のコーディング上の好みを学習していく仕組みだそうです。まだ使い始めなので、これが効くかどうかは評価しません。

最安プランは Go で、記事執筆時点では月 $1 です。しかも Go には月 $10 分の LLM 利用枠が含まれています（https://commandcode.ai/docs/plans/go）。もちろん $10 の枠が何回分になるかはモデル・入力・出力・コンテキスト・キャッシュで全部変わるので、回数の目安としては読めません。

1 点だけ補足しておくと、実際に請求されたのは $1 ちょうどではありませんでした。手元の請求書は Command Code Individual Go が $1.00、processing fee が $0.36 で合計 $1.36、円換算で 226 円です（2026 年 8 月 30 日の請求）。表示価格そのままではない点は、頭に入れておくとよさそうです。

それでも、AI コーディングツールを試す入口として $1 は安い。合わなければ 1 か月で終わればいい、くらいの気持ちで契約しました。

## FREE と書いてあるのに、無課金では 1 回も動かなかった

Command Code を起動して `/model` を開くと、モデル一覧に `Laguna S 2.1 (FREE)` がありました。

FREE なら、まずこれだろう、と思って選択。そのまま無課金の状態で簡単な質問を投げたら、こう返ってきました。

```text
You have insufficient credits to make this request.
Please purchase more credits to continue using Command Code.
```

![many-ai-cli の中の Command Code。models は laguna-s-2.1 (free) なのに insufficient credits で止まる](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_command-code-go_shot1-insufficient-credits.png)

FREE って書いてあるのに使えないのか、と一瞬止まりました。

公式の Pricing & Limits を読むと、無料モデルであってもセッションを開始するためのクレジット条件があります（https://commandcode.ai/docs/resources/pricing-limits）。FREE はモデル単価の話で、アカウント側の条件はまた別、ということでした。

そこで予定どおり Go を契約して、同じモデルで同じリクエストを投げ直したら、普通に動きました。

```text
無課金 → Laguna S 2.1 (FREE) → insufficient credits
Go $1 契約 → Laguna S 2.1 → 正常に利用可能
```

「月 1 ドル払って Command Code を試す」という使い方なら、ここで問題なくスタートできます。

## Windows Native では cmdc で起動する

自分は Windows です。Command Code には Windows 向けの公式ドキュメントがあります（https://commandcode.ai/docs/windows）。

インストールは Node.js 環境から入れます。

```powershell
npm i -g command-code@latest
```

ここで 1 つだけ注意があります。Linux や macOS では短い起動コマンドとして `cmd` が使えますが、Windows には `cmd.exe` が居ます。なので Windows Native では `cmdc` を使います。

```powershell
cmdc --version
cmdc login
cmdc
```

フルネームの `command-code` でも起動します。AI CLI をいくつも並べている側としては、Windows でそのまま動くのはありがたいところです。

## Laguna S 2.1 は Poolside の open-weight モデル

今回メインで使った Laguna S 2.1 は、Command Code の公式では Poolside の open-weight モデルとして紹介されています。用途は agentic coding と long-horizon work、コンテキストは 256K。Pricing & Limits 上は Input / Output / Cache read がすべて Free です（https://commandcode.ai/docs/resources/pricing-limits）。

ただし条件が付いています。

```text
Free while capacity lasts
```

無料提供できるキャパシティが続いている間、という書き方です。今日無料だから永久に無料、ではありません。この記事も「今この瞬間の実測」として読んでください。

## 自分の名前を知らないモデルに、自分は誰かと聞いた

最初にこう聞いてみました。

```text
laguna-s-2.1 これはどこのAIメーカー？
```

Laguna 自身は「Laguna S 2.1 という名前を標準的な AI モデル名として認識できない」という趣旨の回答を返しました。そのうえで `whoami` や `uname -a` や `ver` を実行して、自分が動いている環境の方を調べ始めました。

![Laguna S 2.1 に「これはどこの AI メーカーか」と聞いたら、whoami と uname を実行して環境の方を調べ始めた](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_command-code-go_shot3-laguna-whoami.png)

これはちょっと面白かった。

とはいえ、LLM が自分の正式名称や提供形態を学習知識として持っていないのは珍しくありません。モデルが作られた時点では、そのモデル名がまだ世に出ていないこともあります。名前を知らないから性能が低い、という話ではない。見るべきは、実際のコードとドキュメントを扱えるかどうかです。

## 日本語は普通に通った

次に「日本語で出力して」と指示。これは普通に通りました。

そのまま Command Code 自身のドキュメントを読ませて、AGENTS.md まわりの挙動を説明させます。

```text
READ [Skill(command-code-knowledge/reference/custom-agents.md)]
READ [Skill(command-code-knowledge/reference/memory.md)]
```

のように関連ファイルを読み、AGENTS.md と Memory と system prompt とサブディレクトリと taste の関係を、日本語で整理して返してきました。

![Command Code のドキュメントを 2 本 READ し、AGENTS.md の読み込み順を日本語で整理して返してきたところ](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_command-code-go_shot4-agents-md.png)

日本語の命令を理解する、Markdown を読む、複数ファイルを読む、技術文書を要約する。この範囲なら問題ありませんでした。海外製の open-weight モデルだから日本語が厳しいのでは、という心配は、少なくとも今回の範囲では出てきていません。

長時間の実装や複雑な設計で指示追従がどうなるかは別の話です。そこは第二弾で。

## Usage を開いたら、本当に $0.0000 だった

ここがいちばん確認したかったところです。

モデル一覧に `(FREE)` と書いてあっても、内部で少しずつクレジットを削っている可能性は残ります。なので Usage 画面を見ました（https://commandcode.ai/usage）。以下は自分のアカウントの実測値です。

![Usage 画面。255.7K トークン / 25 runs で、5 時間・週・月の利用枠は 3 つとも 0%](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_command-code-go_shot5-usage.png)

確認した時点で TOTAL TOKENS が 255.7K、TOTAL RUNS が 25。実行履歴の Laguna 分はこうなっていました。

| Input | Output | Model | Mode | Cost |
|---:|---:|---|---|---:|
| 42,489 | 444 | poolside/laguna-s-2.1-free | agent | $0.0000 |
| 40,438 | 149 | poolside/laguna-s-2.1-free | agent | $0.0000 |
| 22,801 | 295 | poolside/laguna-s-2.1-free | agent | $0.0000 |

この 3 回だけで入力は 10 万トークンを超えています。それでも Cost は全部ゼロ。FREE 表示だけでなく、実際の agent 実行も $0 でした。

## ただし、1 セントも減らないわけではない

Usage を眺めていて、1 行だけ引っかかりました。DeepSeek を選んで使った覚えがないのに、履歴に入っています。

```text
deepseek/deepseek-v4-flash
Mode: title-gen
Cost: $0.000362
```

`Mode` が title-gen なので、これはコーディングモデルとして自分が選んだ実行ではなく、Command Code がセッションタイトルなどを生成する内部処理で呼んだものだと思われます。断定はできませんが、Mode 名からはそう読めます。

taste-1 の学習処理の方は `Mode: learning` で `Cost: $0.0000` でした。

![Usage の内訳（Laguna の agent は $0、taste-1 の learning も $0、内部の title-gen だけ $0.000362）](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_command-code-go_fig1.png)

Laguna 本体は無料でも、サービス全体が完全に $0 になるわけではありません。ただし今回計上された内部処理のコストは $0.000362 で、桁として無視できる範囲でした。

## Usage Limits は 0% のまま

同じ Usage 画面には 5-HOUR LIMIT / WEEKLY LIMIT / MONTHLY LIMIT も出ます。スクリーンショットを撮った時点では、3 つとも 0% でした。

約 25 万トークン、25 runs を使った状態でこの表示です。少なくとも Laguna の無料リクエストが通常の利用枠を大きく削っている様子はありません。公式 Pricing 側でも Laguna S 2.1 のリクエスト料金は無料と書かれています（https://commandcode.ai/docs/resources/pricing-limits）。

無料モデルを軽作業担当として置きたい人には、ここはかなり効きます。

## `/model` の (FREE) 表示は 3 つ。正体は末尾に `-free` が付いた別 ID

自分の Command Code CLI では、3 モデルに FREE 表示がありました。

```text
Laguna S 2.1 (FREE)
MiniMax M3 (FREE)
MiniMax M2.7 (FREE)
```

一方、公式の Pricing & Limits を見ると、Laguna S 2.1 は明確に FREE ですが、MiniMax M3 には割引後のトークン料金が載っていて、MiniMax M2.7 にも料金表示があります（https://commandcode.ai/docs/resources/pricing-limits）。

最初はこの食い違いの理由が分かりませんでした。答えは、自分のアカウントの請求コンソールに書いてありました。

![Go プランのコンソール。$1/mo が ACTIVE で、無料モデルの条件が DEAL として並んでいる](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_command-code-go_shot2-console-free-models.png)

DEAL の欄の記載はこうです。

| 記載 | 条件 |
|---|---|
| `laguna-s-2.1-free` is free. Requests on this model cost no credits. | while capacity lasts |
| `minimax-m3-free` and `minimax-m2.7-free` are free. | through September 5, 2026 |
| Your $10 in Go credits stretches to ~$20 of MiniMax M3 usage. | 記載なし |

つまり無料なのは **末尾に `-free` が付いたモデル ID の方**で、公式 Pricing に料金が載っている `MiniMax M3` とは別物でした。CLI の `(FREE)` 表示はこの `-free` 版を指しています。食い違って見えたのは、同じ「MiniMax M3」という名前で 2 つの ID が並んでいたからです。

そして期限が違います。Laguna は `while capacity lasts` で終了日の記載がありませんが、MiniMax の 2 つには **2026 年 9 月 5 日まで**という日付が入っています。この記事を書いているのが 8 月 31 日なので、残りは数日です。

同じ画面には `0% of monthly limit used · resets Sep 30` とも出ていて、Go に付いてくる $10 の枠は手つかずのままでした（Go プラン側の説明は https://commandcode.ai/docs/plans/go）。

## many-ai-cli に 7 つ目の provider として足した

ここからが自分にとっての本題です。

many-ai-cli はもともと Claude Code / Codex CLI / GitHub Copilot CLI / Cursor Agent CLI / Grok Build CLI / opencode の 6 つを並べて走らせるダッシュボードでした。Command Code をここへ 7 つ目として足しました。

![many-ai-cli の中で Command Code v1.37.0 が起動したところ。models は laguna-s-2.1 (free)](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_command-code-go_shot6-in-many-ai-cli.png)

実装で決めたことを、そのまま並べます。

**provider ID は `command-code`。`cmd` / `cmdc` は使わない。**
表示名は Command Code。OS 別エイリアスの `cmd` / `cmdc` は、ID にも shell 初期化の透過化にも使いません。Windows の `cmd.exe` と衝突するからです。実行ファイルの解決は `exec.LookPath` がそのまま効きます。Windows では pnpm が置く `command-code.CMD` が PATHEXT で拾えることを実測しました。

**通常起動では権限フラグを暗黙に足さない。**
`--trust` も `--auto-accept` も `--yolo` も `--skip-onboarding` も勝手に付けません。付けるのは、spawn パネルで承認モードを明示選択したときだけです。対応はこうしました。

| Hub 側の承認モード | Command Code に渡す引数 |
|---|---|
| plan | `--permission-mode plan` |
| acceptEdits / auto | `--auto-accept` |
| bypassPermissions | `--yolo` |

`bypassPermissions` は Hub 側で確認必須にしています。ここを黙って通す設計にはしません。

なお、この記事の対象である Command Code v1.37.0 の `--permission-mode` は standard / plan / auto-accept の 3 値でした。公式 docs には `dont-ask` の記載がありますが、この版には入っていません。

**doctor は `command-code status --json` を読む。**
PATH に `command-code` があるときだけ叩いて、未ログインなら Warn とログイン案内、ログイン済みなら OK に分けます。終了コードは見ません（未認証時に 1 を返すため）。Node 22 未満なら Warn を出します。

**slash コマンドは v1.37.0 の `--help` から 59 件。**
`/mode:xxx` の形はパーサの契約に合わないので除外し、`/orchestrate` は Hub での動作が未確認なので入れていません。Usage リンクは https://commandcode.ai/usage を設定画面から開けるようにしました。

**承認検出は、まだ空です。**
many-ai-cli の中心機能は「各 CLI の承認待ちを検出して 1 タブに集約する」ことなのですが、Command Code についてはこのパターン表が空のままです。実機の PTY から trigger 文言を採取していないからで、推測でパターンを書くと誤検出の方が害になります。ここは fixture を採ってから別で入れます。

![many-ai-cli 側の Command Code 配線（provider ID・承認モードの対応表・doctor・承認検出は未実装）](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_command-code-go_fig2.png)

つまり現時点の立て付けは、Command Code の TUI をそのまま many-ai-cli の中で扱えるようにした、というところまでです。承認のボタン化まで含めて完成、ではありません。そこは正直に書いておきます。

## Go では Provider API は使えない

Command Code には Provider API もあります（https://commandcode.ai/docs/provider）。OpenAI 互換の `/provider/v1/chat/completions` と、Anthropic 互換の `/provider/v1/messages` です。

ただし記事執筆時点では、月 1 ドルの Go で Provider API は使えません。

なので今回 many-ai-cli へ入れたのは、Command Code を API プロバイダーとして直接叩く実装ではありません。Command Code CLI そのものを many-ai-cli から扱えるようにした、という話です。API プロバイダーとして載せるかどうかは別の検討になります。

## v0.8 は 9 月 1 日に出す予定です

ここまでの Command Code 対応は develop に入っている状態で、この記事を書いている時点の公開済み最新タグはまだ v0.7.0 です。

この記事の翌日、2026 年 9 月 1 日に、Command Code 対応を含む **v0.8** をリリースする予定です。リリースはここに出ます。

https://github.com/ishizakahiroshi/many-ai-cli/releases

予定なので、ずれたらすみません。出たあとに `pnpm add -g many-ai-cli` を叩き直すと入ります。

## 無料モデルは主力の代替ではなく、振り分け先として面白い

まだ Laguna S 2.1 で大きな実装はしていません。なので Claude や Codex の代わりになる、とは言えません。

今いちばん面白いと思っているのは、代替ではなく振り分け先としての使い道です。

強いモデルに任せたいのは、新機能のアーキテクチャ設計、複雑な仕様の実装、難しいバグ、大規模リファクタリング、セキュリティ上重要な修正、大きな設計判断。

一方で、README の整理、リポジトリ構造の調査、Markdown 生成、コード検索、ログ解析、Git diff の説明、コメント追加、簡単なテスト追加、小さな修正、既存コードの説明。こういう仕事まで毎回いちばん高いモデルへ投げる必要があるのか、というと、たぶんない。

Laguna のようなモデルをここに置けば、強いモデルの利用枠を温存したまま雑用を回せます。月 1 ドルなら、AI コーディングのサブ回線として置いておく感覚に近い。

many-ai-cli を作っている理由も、結局そこです。AI CLI もモデルも、1 つに収束するより増え続けています。どれか 1 つを選んで全部任せるより、使えるものを並べておいて仕事で切り替える方が現実的かもしれない。今回 Command Code を足したことで、安いサブ回線も同じ画面に置けるようになりました。

## 第二弾でやること

FREE 表示の 3 モデルへ、同じ課題を投げます。FizzBuzz ではなく、実際の many-ai-cli リポジトリを使う予定です。

まずは読み取り専用のタスクから。

```text
このプロジェクトを確認して、
何をするソフトウェアなのか説明してください。

主要なディレクトリとファイルの役割も整理してください。

ファイルは変更しないでください。

回答は日本語でお願いします。
```

そのあと調査タスク、最後に「必要最小限の変更で修正し、関連テストを実行し、git diff を確認して日本語で説明する」まで。

比べたいのは、日本語の自然さ、リポジトリ理解、指示追従、修正精度、不要な変更の少なさ、処理速度、そして Usage と実 Cost です。とくに `minimax-m3-free` と `minimax-m2.7-free` が、コンソールの記載どおり実際の Usage でも $0 になるのか。無料期間が 9 月 5 日までなので、そこは早めに測ります。

## 日本語で Command Code を紹介している記事

調べる過程で参考にさせてもらったものを置いておきます。

まさおさん「今話題の激アツ AI サブスク『Command Code GOAT』をこっそり解説！ OpenCode Go と何が違う？」（https://note.com/masa_wunder/n/n7a3531976ae2）。今回の Go より上位の GOAT やモデルプロバイダーとしての使い方に踏み込んでいます。同じ方の YouTube 動画「本当は秘密にしたい今熱い AI モデルのサブスク 2 選！『CommandCode』と『B.AI』を解説します」もあります（https://www.youtube.com/watch?v=prlpTj18Qhs）。

sitne さん「OpenCode Go の対抗馬、Command Code」（https://note.com/sitne/n/n6cf5d027e6e0）。立ち位置を掴むのに読みました。

KeiTy さん「Command Code が $1 Go Plan 発表。OpenCode Go / OpenRouter と徹底比較」（https://note.com/keity717/n/n597987c6316d）。

同じく KeiTy さん「AIはもう、インフラである」（https://note.com/keity717/n/n98be0dbcd4ec）。Command Code と Mac 艦隊で API 課金から脱却する、という 2026 年 5 月の記事です。かなりヘビーユース寄りで、単なるサービス紹介とは違う面白さがあります。

AI 星図「Command Code」（https://www.myaiexp.com/jp/items/dev-tools/command-code）。概要をざっと確認したいとき用。

## セキュリティは別で確認したい

AI コーディングエージェントはローカルのコードを読み、場合によっては shell も叩きます。安いから使う、だけで済ませたくないところです。

Command Code 公式は、コードを学習に利用しないことなどを説明しています（https://commandcode.ai/docs/plans/go）。

そのうえで個人的には、どのサービスでも `.env` / API キー / パスワード / 秘密鍵 / 顧客情報 / 個人情報 / 本番データ / 社外秘を無造作に渡さない運用は必要だと思っています。初めて使うモデルで Permission をいきなり全部バイパスしないのも同じです。何のコマンドを実行しようとしているかを見ながら使った方が、挙動の理解が早い。

## many-ai-cli はこんなときに刺さります

- Claude Code / Codex / Copilot / Cursor / Grok / opencode を同時に走らせていて、状態を 1 画面で見たい人
- 各 CLI の承認待ちを 1 か所で捌きたい人（ターミナルの往復をやめたい人）
- 席を外している間もスマホから承認したい人
- 月 1 ドルの Command Code のような安いサブ回線を、主力の AI と同じ画面に並べたい人

いずれかに心当たりがあれば、`pnpm add -g many-ai-cli` と `many-ai-cli setup` の 2 コマンドで試せます。設定ファイルを書く必要はありません。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/many-ai-cli
- npm: https://www.npmjs.com/package/many-ai-cli

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## あわせて読みたい

- [many-ai-cli v0.7.0。承認パネルが握り潰される原因を138件のダンプから直した](https://qiita.com/ishizakahiroshi/items/fc31553bf9427109d4dc)（今回足した Command Code の承認検出が、なぜ「空」で止まっているかの背景になります）
- [Ox Alpha を無料で試す4つの経路と、正体を推測するための材料の集め方](https://qiita.com/ishizakahiroshi/items/707b2da49958c27f5e2e)（同じ「無料で使えるモデルを実測する」系の記事です）
- [OpenRouter を many-ai-cli に載せるなら、独立プロバイダにはしない](https://qiita.com/ishizakahiroshi/items/4feda29aa885644c982b)（今回のように provider を 1 本足すか、backend として載せるかの判断の話です）

## おわりに

$1 で 25 万トークン動かして、Cost 欄は $0.0000 のまま。数字としてはそれだけの話です。

ただ、無料モデルが 1 つ増えると、自分の中の仕事の振り分け方が少し変わります。全部を強いモデルに投げていたのを、調査と雑用だけ別の回線に流す。その回線が月 1 ドルなら、置いておくコストはほぼない。

MiniMax の `-free` 版が実際の Usage でも $0 で通るのかは、まだ測っていません。無料期間が 9 月 5 日までなので、第二弾はそこに間に合わせます。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-09-01_command-code-go-laguna-free-model/

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
