---
title: >-
  Codex は API を叩き、Claude は Web でタダだった。多 AI CLI 時代の skill
  経路分岐と、次に狙う「browser-delegate」の話
tags:
  - AIエージェント
  - Claude
  - MCP
  - OSS
  - codex
private: false
updated_at: '2026-07-23T01:57:26+09:00'
id: a72e4637bdd7499421be
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![Codex は API、Claude は Web の対比ヒーロー](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-23_codex-vs-claude-mcp-browser-delegate/hero.png)

前作の note で「`gh run watch` の tail を信じたら Validate が赤いままタグを push していた話」を書きました。https://note.com/ishizakahiroshi （公開後に URL 差し替え予定）。それを書いている最中に、ちょっと別方向から強い風が吹きました。

自作のスキルを Codex CLI で動かしたときの挙動と、Claude Code で動かしたときの挙動を、目の前で並べて比べてしまったのです。

同じスキル。同じ記事のヒーロー画像を作る目的。それなのに Codex は Gemini の画像生成 API を課金しながら叩き、Claude は Chrome を自動操作して Gemini Web の無料枠を使い切っていました。

見た目には同じ結果。裏では月に数百円から数千円の差が積み上がる構造。

## 忙しい人向け（AI 音声解説・18 分）

この記事の音声版を NotebookLM で作りました。移動中・作業中の "ながら聴き" にどうぞ。

https://youtu.be/zdyBH7-WOYY

## TL;DR

- 自作の記事作成スキルを Codex CLI と Claude Code で並べて動かしたら、**Codex は Gemini 画像 API を有料で叩き、Claude は Chrome を自動操作して Web の無料枠でタダで生成**していた
- この差は個々の CLI の性能差ではなく、**Anthropic が MCP サーバーとして純正の Chrome 拡張（Claude for Chrome）を提供している**エコシステムの厚みが効いている
- MCP は中立プロトコルなので技術的には他 CLI でも browser 自動化は可能（`playwright-mcp` 等）。ただ「配線ゼロで即使える」は現状 Claude だけ
- ここに **「Codex 親セッションの状態を保ちつつ、browser が必要な瞬間だけ Claude 子セッションに肩代わりさせる」**という orchestration の空きスペースがある
- 自作の多 AI CLI ラッパー `many-ai-cli` の次のリリース候補として起票済み。設計メモを `docs/local/pending_browser-delegate-cross-provider.md` に置いた
- 記事末尾に、この機能が 3 ヶ月・6 ヶ月・1 年後にどうなっていそうかの予測も置きます

## many-ai-cli について

自作で [many-ai-cli](https://github.com/ishizakahiroshi/many-ai-cli) という、複数の AI コーディング CLI（Claude Code・Codex・Copilot・Cursor・Grok）を並列で走らせて承認を 1 画面に集約するローカル Web ダッシュボードを作っています。複数のAIコーディングCLIセッションの承認管理・監視を行うローカルWebダッシュボードです。

同じ悩みを持っている方は、下記で入ります。

```bash
npm i -g many-ai-cli
```

リポジトリはこちらです（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/many-ai-cli

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-23_codex-vs-claude-mcp-browser-delegate/infographic.png)

## きっかけの会話

きっかけは v0.5.1 リリースの振り返り記事を書いている最中でした。

記事執筆用のスキル（`write-article`）を使い、ヒーロー画像を作らせるフェーズで、Claude Code が Chrome を開いて `https://gemini.google.com/images` に飛び、そこにプロンプトを流し込み、生成待ちの間クルクル回るスピナーを見ながら「これ、無料枠だな」と思ったところで話が飛びました。

前日、同じスキルを Codex CLI で試したとき、コンソールに走っていたのは `POST https://generativelanguage.googleapis.com/...` の HTTP コールと `Successfully generated image (cost: $0.039)` の 1 行でした。

同じ「ヒーロー画像を作る」というスキルの結果として、**片方は Chrome の自動化、もう片方は課金 API の呼び出し**。

これは面白い、と思ってその場でメモしました。以下、事実の順番に整理していきます。

### 最初のメモ

自分の手元に残っていた事実を、時系列で並べます。

1. `write-article` スキルは、記事のヒーロー画像を生成する手順を `references/hero-pipeline.md` に持っている
2. `hero-pipeline.md` の Step 2 は「Gemini Web を Chrome で操作して無料で作る」経路
3. 同じ md の Step 2B は「Web 経路が失敗したら Gemini API を有料で叩く」経路
4. Claude Code は `claude-in-chrome` MCP サーバーを持っているので Step 2 を実行できる
5. Codex CLI は `claude-in-chrome` を持たないので、実質 Step 2B しか通れない
6. 結果、**同じスキルなのに Claude ではタダ・Codex では有料**という現象が発生する

これを見て、いくつか疑問が浮かびました。

- なぜ Claude だけがブラウザ自動化を持てているのか？
- 他 CLI（Grok Build CLI、GitHub Copilot CLI、Cursor Agent CLI）はどうなっているのか？
- 実際に月ベースでどのぐらいコストが変わるのか？
- Codex 側のセッションを進めつつ、必要なところだけ Claude に肩代わりさせられないか？

順番に見ていきます。

## Codex は Gemini API を叩いていた

まず片方の実装。Codex CLI で `write-article` を回すと、ヒーロー画像生成は最終的にこのコマンドに落ちます。

```powershell
& 'C:\dev\workshop\docs\local\hero-automation-trials\gen-hero.ps1' `
    -PromptFile '<topic>\<...>_hero.prompt.txt' `
    -OutSrc     '<topic>\<...>_hero_src.png' `
    -Tier batch
```

`gen-hero.ps1` の中身は、要するに Google の Gemini Image API（Nano Banana）を HTTP で叩いて PNG を落とすラッパーです。

Google の画像モデル料金は公式ドキュメントに明記されています。https://ai.google.dev/pricing 。2026 年 7 月時点の実行モデル `gemini-2.5-flash-image`（旧称 Nano Banana 1）で 1 枚 $0.039、新しい `gemini-3.1-flash-image-preview`（Nano Banana 2、2026-02 リリース）で 1 枚 $0.0672。日本円換算で 6 円から 10 円/枚くらい。

1 記事あたりヒーロー 1 枚、本文挿絵 2〜4 枚、インフォグラフィック 1 枚。合計 4〜6 枚の画像を毎回生成する記事運用だと、月 20 本書けば 100 枚を超えます。**月あたり数百円から千円強の課金**が積み上がる計算です。

これは「お金を払っている感覚」がゼロで済んでいる箇所でも同じで、Codex 単独運用だと Web 経路のフォールバックすら選択肢に上がりません。理由は、Codex には「ログイン済みブラウザーを AI が操作する」ための仕組みが標準では入っていないからです。

## Claude は Web でタダだった

もう片方。Claude Code で同じスキルを動かすと、Step 2 に降りて **Chrome を実際に開いて Gemini Web の入力欄にプロンプトを打ち込み、生成完了を待って、ダウンロードボタンを押す**、という一連の GUI 操作を Claude 自身がやってくれます。

つまり Anthropic 純正の Chrome 拡張（Claude for Chrome）を経由して、**ログイン済みブラウザーの Web セッションをそのまま流用**しています。私は Google AI Plus に加入しているので、Gemini Web からの画像生成には無料枠と Plus のクレジットが割り当てられます。**ブラウザー経由の生成は追加課金なし**、というのがカラクリです。

同じ画像モデル（Nano Banana）にアクセスしているのに、経路が違うだけでコストが 0 円になる。API 版は「開発者向け・可計測」、Web 版は「エンドユーザー向け・生成枠込みの定額サブスク」という商品設計の違いがそのまま反映される形です。

## なぜ Claude だけがタダ経路を持てるのか

この差はランダムではなく、明確な理由があります。**Anthropic が「Claude for Chrome」という公式の Chrome 拡張を出して、Model Context Protocol（MCP）のサーバーとして自動登録される形にしている**からです。https://claude.com/claude-for-chrome にプロダクトページがあり、Anthropic 側の告知は https://www.anthropic.com/news/claude-for-chrome から辿れます。

Claude Code CLI 側は、この拡張がインストールされていれば、追加の設定なしで「ブラウザーを操作する」ツール群（`mcp__claude-in-chrome__navigate` / `computer` / `find` / `read_page` など）を使えるようになります。ユーザー側の作業はゼロ。

これがなぜ「他 CLI では真似しにくいか」というと、拡張が **Claude アカウントとのログイン紐付け前提**で動くように設計されているためです。Codex や Grok の CLI から同じ拡張を呼び出そうとしても、認証が通らない。汎用 MCP サーバー（`playwright-mcp` 等）を別途配線する必要があります。https://github.com/microsoft/playwright-mcp

つまり結論はこう:

- MCP は中立プロトコル（Anthropic 発だが公開規格）https://modelcontextprotocol.io/
- browser 自動化は原理的に MCP で誰でもできる
- しかし **Anthropic は「拡張を純正で提供 + ログインで即動く」の垂直統合をしている**
- Codex / Grok / Copilot は各自で汎用 MCP を配線する運用が必要

この差が、今回のような「skill を使ったら経路が分岐する」という現象を生みます。

## Model Context Protocol とは

一次情報を丁寧に置いておきます。

**Model Context Protocol（MCP）** は Anthropic が 2024 年 11 月に公開したオープン規格で、AI アプリケーション（Claude、その他 LLM ホスト）と外部データソース・ツールをつなぐための共通プロトコルです。https://www.anthropic.com/news/model-context-protocol にアナウンスがあり、仕様と実装は https://modelcontextprotocol.io/ から辿れます。

わかりやすい 3 行:

- ホスト（Claude Code CLI 等）は MCP クライアントを内蔵している
- ローカルまたはリモートに立てた MCP サーバーが「ツール群」を公開する
- LLM は自然言語でツールを呼び、サーバーが結果を返す

代表的な MCP サーバーには次のようなものがあります。参照実装のリポジトリは https://github.com/modelcontextprotocol/servers に集約されていて、2026-07 現在で 88.8k star に達しています。

- ファイルシステム操作
- Git / GitHub 操作
- Slack / Notion / Google Drive
- Postgres / SQLite など DB
- ブラウザー自動化（`playwright-mcp` / `puppeteer-mcp` など）

**Anthropic の Claude for Chrome 拡張は「MCP サーバー」として動きます**。つまり Claude Code CLI から見ると、他の MCP サーバーと同じ扱いで「browser 系ツール」を呼べる形。ここが技術的な要点です。

![MCP アーキテクチャ俯瞰図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-23_codex-vs-claude-mcp-browser-delegate/illustration-1.png)

MCP は一言でいえば「LLM 側の統一プラグイン仕様」です。ホスト側（Claude Code / Cursor / Codex CLI）が MCP クライアントを内蔵し、サーバー側は Slack でも Chrome でも Postgres でも同じ規格で公開できる。これが Anthropic が中立プロトコルとして公開した理由でもあります。

規格自体はプロプライエタリではなく、OpenAI も MCP を採用する意向を表明しています（2025 年後半以降、Agents SDK まわりで対応が進んでいる）。https://developers.openai.com/api/docs/guides/agents 。ただし前述のとおり、Anthropic 純正の Chrome 拡張は Claude 前提なので他 CLI からは使えません。

## Claude in Chrome 拡張の中身

もう一段踏み込みます。https://claude.com/claude-for-chrome によると、Claude for Chrome は 2025 年後半にリサーチプレビューとして出て、2026 年に一般公開されました。Chrome ウェブストアからインストールして Claude アカウントでログインすると、以下のことができるようになります。

- 現在開いているタブの内容を Claude に見せる
- 特定 URL に Claude が navigate する
- Claude が要素を find して click / type / scroll する
- ページの Accessibility ツリーを read_page として構造化取得できる
- スクリーンショットを取って画像として Claude に見せる

ホスト側（Claude Code CLI や Claude デスクトップ）から呼ぶツール名は `mcp__claude-in-chrome__*` で、例えばこの記事を書くのに使ったのは次のような呼び出しでした。

```
mcp__claude-in-chrome__navigate(url="https://gemini.google.com/images?hl=ja")
mcp__claude-in-chrome__find(query="prompt input")
mcp__claude-in-chrome__computer(action="type", text="<プロンプト>")
mcp__claude-in-chrome__computer(action="wait", duration=10)
mcp__claude-in-chrome__computer(action="left_click", coordinate=[1105, 310])
```

裏でやっているのは実際の Chrome のイベント。だからログイン状態も維持されるし、Google のアカウント情報も引き継がれます。**「無料枠の Gemini Web を Claude が操作する」がそのまま実現できる**のはここです。

セキュリティ面でも Anthropic は慎重に設計しています。拡張はサイト単位の権限（この URL は許可、この URL は不許可）を持ち、確認モーダルの発火設計や、Claude によるナビゲーションの許可 UI などが明示されています。詳細は Anthropic の 2025 年 8 月時点のアナウンスを参照。https://www.anthropic.com/news/claude-for-chrome

![Claude for Chrome の中身](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-23_codex-vs-claude-mcp-browser-delegate/illustration-2.png)

これが Claude Code の CLI から見ると `mcp__claude-in-chrome__navigate` `mcp__claude-in-chrome__computer` のように 1 段抽象化されたツール群として現れます。実体は本物の Chrome のイベントを操作しているため、ログイン済みセッションもそのまま流用できます。

## 他 CLI の browser 事情

比較を横並びで整理します。

### Codex CLI（OpenAI）

- リポ: https://github.com/openai/codex
- 標準では browser 自動化は入っていない
- MCP サーバーのサポートは実装されており、`~/.codex/mcp.json` に手動で配線すれば `playwright-mcp` 等を呼べる https://github.com/microsoft/playwright-mcp
- OpenAI は Agents SDK として "computer use" API を持っている https://developers.openai.com/api/docs/guides/agents 。ただ CLI とは別レイヤーで、CLI から直接エンドユーザーのブラウザーを叩くパイプラインは薄い
- Anthropic の Claude for Chrome 拡張を Codex から呼ぶことは、認証が通らないため実質不可

### Grok Build CLI（xAI）

- xAI が公開している CLI で、Claude Code 系よりカルパシー的にミニマル
- MCP 対応の記述は薄く、外部ツール接続の抽象化がまだ弱い
- 2026-07 時点で browser 経路の標準実装はなし
- 何かやりたければ shell 経由の PowerShell スクリプトや `curl` を叩く運用

### GitHub Copilot CLI

- リポ: https://github.com/github/gh-copilot（現行）
- CLI 単体でのブラウザー自動化は限定的
- VS Code 上の Copilot Chat であれば Chrome の DevTools MCP 拡張などと組み合わせて使う道がある
- CLI から MCP を呼ぶには config 配線が必要。標準では入っていない
- マニュアル: https://cli.github.com/manual/gh_copilot

### Cursor Agent CLI

- Cursor エディター本体は browser 操作の一部機能を持っている
- CLI 単体（`cursor-agent`）の browser 経路は 2026-07 時点で断片的
- MCP サーバーの追加は Cursor 側でサポートされている https://cursor.com/docs

### Claude Code CLI

- リポ: https://github.com/anthropics/claude-code
- Claude for Chrome 拡張が入っていれば **配線ゼロで即 browser 使える**
- 拡張がなくても `~/.claude/mcp.json` に汎用 MCP を書けば同じことはできる
- 「純正拡張がある」ことが Codex 等との実質的な差

![多 AI CLI の browser 対応マトリクス](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-23_codex-vs-claude-mcp-browser-delegate/fig-1.png)

この横並びで見ると、**「browser 対応が組み込まれているのは Claude だけ、他は汎用 MCP を手動配線する余地しかない」**という非対称がひと目でわかります。技術的に不可能なわけではなく、投資の差です。

## skill 設計が経路分岐を吸収した具体例

自分の `write-article` スキルは、この「Claude なら Web、他 CLI なら API」を明示的に受け入れる設計になっています。ヒーロー画像を作る参照ファイル `references/hero-pipeline.md` の冒頭にはこう書いてあります（要旨）:

> Step 2（既定）: Gemini Images をブラウザーで操作する経路。Claude Code + Claude for Chrome 拡張がある環境で、Google AI Plus のログイン済み Chrome を優先する
> Step 2B（フォールバック）: Web 経路が失敗した場合だけ `gen-hero.ps1` を呼ぶ。実行直前にユーザーへ課金確認する

つまり、**同じスキル・同じ md・同じ画像生成の目的に対して、実行環境の CLI ごとに経路が変わる**設計にしてあります。

Codex CLI で実行すると、Step 2 の実行手段（`mcp__claude-in-chrome__*` ツール）が存在しないので即 Step 2B に降ります。Claude Code で実行すると、Step 2 が通り、Web で無料で終わります。

インフォグラフィックの生成も同じ構造になっていて、`references/infographic-pipeline.md` にはこう書いてあります。

> エンジン A（既定）: NotebookLM のインフォグラフィック機能を Chrome で操作。無料 1 日 3 枚
> エンジン B（明示指定時のみ）: `make-infographic` skill 経由で Gemini API（Nano Banana 2）を叩く。1 枚 3〜6 円

このおかげで、私は Claude で書いているうちは NotebookLM の無料 3 枚枠を毎日使い切ることができ、Codex で書いているときだけ Nano Banana 2 の課金が走る、という運用になっています。

## 実際の経済性

雑にシミュレーションしてみます。個人ブログを月 20 本更新する運用を想定。

**1 記事あたりの画像枚数**

- ヒーロー: 1 枚（必須）
- インフォグラフィック: 1 枚（推奨）
- 本文挿絵: 2〜4 枚
- 合計: 4〜6 枚

**Codex 側で全部 API を叩く場合**

- Nano Banana 1: $0.039 × 平均 5 枚 × 20 本 = $3.9/月
- Nano Banana 2 に切り替えると: $0.0672 × 5 × 20 = $6.72/月
- インフォグラフィックだけ Nano Banana 2 なら: $3.9 - $0.039 + $0.0672 = $3.93/月

**Claude 側で Web 経路が通る場合**

- Nano Banana 1（Gemini Web 経路・Google AI Plus 契約分に吸収）: $0/月
- NotebookLM インフォ（無料 3 枚/日）: $0/月
- **合計: $0/月**

差額は月 500 円から 1000 円くらい。1 年で 6,000 円から 12,000 円。**絶対額としては小さいのですが、大事なのは「これが構造的な差」だという点です**。

実は同じ気づきをより早い時点で note に短くまとめていました。「AI に任せて 600 円。Gemini Web なら、ヒーロー画像までほぼ追加料金なしだった」。https://note.com/ishizakahiroshi/n/n065ea9354edf 。今回の記事はその「600 円 → 0 円」の発見を、多 AI CLI 時代の orchestration の観点まで広げた続編になります。

![経済性試算: 月 20 本記事の運用](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-23_codex-vs-claude-mcp-browser-delegate/fig-2.png)

金額そのものより「同じスキルなのに CLI 選択で経済性が構造的に変わる」ことに注目してほしいところです。特に skill 実装者としては、両経路を維持することで「使う CLI を選ぶ自由」を読者に残せます。

- Web 経路は月額サブスク（Google AI Plus / Anthropic Pro）で「無限に近い定額」
- API 経路は「使った分だけ従量課金」

書く量が増えるほど差が広がります。ここに気付いたのが、記事執筆スキルを 2 種類の CLI で並列運用した後でした。

## ひらめきの瞬間

会話を進めているうちに、思ったことがあります。

「これ、Codex 側で走っているセッションから、必要な瞬間だけ Claude を呼び出せたら、両方の得意分野を活かせるじゃないか」

Codex は Codex で書くの得意な文脈（例: 私が Codex に投げ慣れている設計判断や、Codex 固有の実装スタイル）を持っています。それをそのまま活かしつつ、**ブラウザー操作が要る瞬間だけ Claude 子セッションに肩代わりさせる**。

例えばこういう構図です。

- Codex 親セッション: 記事本文を書く（コード解説など Codex が得意）
- Claude 子セッション: 画像生成（Gemini Web） / NotebookLM / X 予約投稿 のような browser 系タスクを担当
- 完了したら Claude 子は結果（PNG パス、公開 URL など）を親に返す

これ、既に部品はある程度揃っている、というのが面白いところで。次で説明します。

![browser-delegate: 親子セッション間のオーケストレーション](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-23_codex-vs-claude-mcp-browser-delegate/illustration-3.png)

Codex は本文執筆に集中し、Claude 子は画像生成 / NotebookLM / X 予約投稿などの browser 系タスクだけを担当。結果は共有 board.md に書き戻して親に返す。役割分業の絵です。

## 既存の土台

`many-ai-cli` は v0.5.1 の時点で以下を持っています。

**1. `spawn-child` API**

親セッション（provider 問わず）が子セッションを生やせる REST エンドポイントが既に実装されています。

```
POST /api/sessions/:id/spawn-child
{
  "role": "designer",
  "provider": "claude",
  "model": "claude-opus-4-7",
  "prompt": "この記事の hero 画像を作って",
  "cwd": "./"
}
```

Codex 親から Claude 子を spawn する、が現時点でも技術的には可能。

**2. Orchestration board（`board.md`）**

子セッションが増えると `~/.many-ai-cli/orchestration/<id>/board.md` という共有 Markdown ファイルが生まれます。親の指示、子の完了報告、子から親への結果引き渡しがすべてここに集約される仕組み。

**3. git worktree の自動隔離**

親と子の作業ディレクトリが被らないように、子は自動で git worktree に隔離されます。ブランチが違うので競合しません。

**4. `## DONE <role> session=<child_id>` マーカー**

子が完了した瞬間に board.md に自動で書き込まれるマーカー。親はこれを検知して次のアクションに移れます。

つまり **spawn からタスク実行、完了通知、結果引き渡しまでのフレームワークは既に動いている**。あとは「browser 使いたい」ケースに特化して滑らかにするだけ、という状況です。

## 未実装の 4 ピース

とはいえ、実運用に持っていくには詰めるべき論点が 4 つあります。

### P1: browser 必要性の自動判定 + ルーティング提案

親セッションが「これ browser 要るな」と判定する導線が欲しい。案:

- ヒューリスティック検出: 親のプロンプトに `Gemini Web` / `NotebookLM` / `X 投稿` / `Google カレンダー` / `Chrome` / `browser` 等の語が出たら many-ai-cli 側で通知
- explicit マーカー: 親が `[BROWSER-DELEGATE] <task>` と書いたら board.md に子タスクとして自動登録
- skill 側で吸収: 各 skill の references に「browser 経路と API 経路の分岐」がある場合、`many-ai-cli` が provider 種別を参照して自動選択

### P2: セッション文脈の自動要約 + 引き継ぎ

親（Codex）の会話履歴を丸ごと子（Claude）に渡すと context が爆発します。**「browser で何をしたいか」に絞ってサマリを board.md に書く**仕組みが要る。

- 親の LLM に board.md 用のタスクサマリ生成を依頼
- サマリの粒度: 入力、期待する出力、制約、認証状態（既ログイン想定など）
- 子は board.md だけを context として受け取り、親の生履歴には触らない（責務分離）

### P3: 完了後の親への結果注入

Claude 子が browser で得た結果（DL 画像パス、ログイン後の URL、抽出テキスト、スクリーンショット）を **Codex 親の会話に自然に注入する**仕掛け。

- board.md に `## RESULT <role> session=<child_id>` セクションで書き戻す（DONE マーカーと対称）
- 親側は次ターンで `board.md` の RESULT を context に含めて応答
- ファイルパス系は「親の cwd 相対」に正規化して渡す

### P4: 課金 / 認証の使い分け最適化

- 無料経路優先: Claude 側 browser で無料枠を使う（NotebookLM 3 枚/日、Gemini Web、ChatGPT Plus の画像生成、X 予約投稿）
- 有料経路 fallback: Claude 子が unavailable なら Codex 親が API 経路に降りる（既存の hero-pipeline.md Step 2B のパターン）
- 課金追跡: Web 経路と API 経路の使用量を統計として `~/.many-ai-cli/logs/browser-delegate.jsonl` に記録

## 設計案 A/B/C

実装アプローチとして 3 つの案を考えました。

### case A: 明示的な `spawn-child` 拡張

親が既存の `spawn-child` API に `role: browser-delegate` を渡すと、子が自動で Claude provider に固定される（`provider: claude` 指定と等価）。ユーザー体験:

```
Codex 親: "この記事を NotebookLM でインフォグラフィックにして"
      → 内部で spawn-child role=browser-delegate provider=claude
      → Claude 子が browser 経路で処理
      → RESULT に PNG パス書き戻し
Codex 親: (RESULT を context に含めた次ターンで) "できました。パスは …"
```

Pros: 明示的で分かりやすい。デバッグしやすい
Cons: 親側がこの role を意識して指示を書く必要がある

### case B: 暗黙の routing hook

many-ai-cli 内部で「NotebookLM 系タスクが親のプロンプトに現れた」等のパターンを検知し、**自動で** Claude 子を spawn して board.md を作る hook を仕込む。ユーザー側は spawn を意識しない。

Pros: UX が滑らか
Cons: 誤発火のリスク（ユーザーが手動でやりたい時に spawn される）→ `config.yaml` で opt-in にするのが安全

### case C: skill 側で完全に吸収

skill の `references/*.md` 内で「provider ≠ claude なら spawn Claude 子」の明示ロジックを書く。many-ai-cli コア側の変更は最小。

Pros: 実装量最小・skill 単位で挙動制御しやすい
Cons: 全 skill に配線が必要・記述の重複

**推奨**: まず case C（skill 側で 1〜2 skill 実装 → dogfood）→ 効果検証後に case A（コア API 拡張）→ さらに UX 磨きが要れば case B（暗黙 hook）の順で段階導入。

![設計案 A/B/C の比較](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-23_codex-vs-claude-mcp-browser-delegate/fig-3.png)

段階導入の理由は 3 つ。（1）skill 側の実装だけならコア変更ゼロで検証できる、（2）dogfood で「本当に欲しかった機能」が分かる、（3）UX の磨き（case B）は最後にやれば十分。

## 業界動向: MCP エコシステムの現状

視野を少し広げます。MCP 自体がどこまで広がっているか、事実だけ並べます。

- **参照サーバーの数**: https://github.com/modelcontextprotocol/servers に集約されているのが 30+ 種類（Slack、Notion、GitHub、Postgres、Google Drive、ファイルシステム、ブラウザー、SQLite、Puppeteer、Fetch など）
- **リポの star 数**: 参照実装リポで 88.8k（2026-07 時点）。エコシステムとしては急拡大中
- **主要ホスト側の対応**:
  - Claude Code / Claude Desktop: MCP フル対応（発案元）
  - Cursor: MCP サーバー追加 UI を持つ https://cursor.com/docs
  - Codex CLI: MCP サーバー設定を `~/.codex/mcp.json` で書ける
  - Grok Build CLI: 対応記述薄い
  - GitHub Copilot: VS Code 拡張経由で MCP を扱う道はある
- **仕様の中立性**: 規格は https://modelcontextprotocol.io/specification に公開。プロプライエタリではない
- **セキュリティ**: 各 MCP サーバーが「信頼できるサーバーか」を確認する仕組みは各ホスト側で実装される（Claude Desktop なら明示的な承認、Claude Code は `~/.claude/mcp.json` の設定ファイルで管理）

つまり **プロトコルとしては業界標準に近づいているのに、「純正 Chrome 拡張」というインフラを持っているのは Anthropic だけ**、という非対称が生まれています。

これは各社の戦略の違いを反映していると思います。

- Anthropic: 「Claude を家の隅々まで住まわせる」路線。デスクトップアプリも Chrome 拡張も VS Code 拡張も自前で出す
- OpenAI: 「API 経由でどのアプリからも呼べる」路線。GPT を SaaS のバックエンドとして売る
- xAI: 「Grok は Twitter/X との統合が主戦場」。ローカル PC の browser にはあまり投資してない
- Google: Gemini Web を Chrome にネイティブ統合。ただし Codex や Claude からは操作対象になる側

多 AI CLI 運用者から見ると、この非対称が **「同じ skill を Claude で走らせるか Codex で走らせるかで結果が変わる」** 現象として現れる。今回の題材の核心はここです。

## 予測

ここから先は私見なので、外れる可能性込みで書きます。

### 3 ヶ月後（2026 年 10 月頃）

- OpenAI が Claude for Chrome 相当の拡張をリリースする可能性は低い。理由: OpenAI の戦略は「API + SaaS プラットフォーム」で、エンドユーザーの browser を直接持たせるより ChatGPT アプリの中で完結させる方向
- 代わりに Codex CLI で **playwright-mcp 経由のブラウザー自動化**が「よくあるレシピ」として広まる可能性が高い
- xAI（Grok）は browser 系にあまり動かない
- **`many-ai-cli` に browser-delegate 機能が入るなら、v0.6.0 相当の目玉として仕上げられる**（v0.5.2 は他の後処理で埋まる想定）

### 6 ヶ月後（2027 年 1 月頃）

- MCP エコシステム全体で「browser 系サーバー」が最も star を集めるジャンルになっている可能性
- OpenAI と Anthropic の間で MCP プロトコルの拡張仕様（認証、権限、コスト計測）で議論が起きる可能性
- Google Gemini CLI 側が独自の Web 統合を打ち出してくる可能性はあり
- 個人開発者にとっては「Claude で無料、Codex で API 課金」の構造が体感として広く知られるように

### 1 年後（2027 年 7 月頃）

- 多 AI CLI ラッパーが数種類乱立していて、その中の 1 つに **「provider ごとの経路最適化」**が標準機能として入っている
- 「browser を持たない CLI」から「browser を持っている CLI」への delegate は業界一般の設計パターンになっている（かもしれない）
- 個人 OSS で `many-ai-cli` がその流れの参照実装として認識される、というのが私の希望

**予測 3 つはハズしていい**という前提で書いています。とはいえ MCP が中立プロトコルであることと、Anthropic の Chrome 拡張が既に一般公開されていることを踏まえると、「Claude だけタダ」の非対称は今後 1 年は続くと考えています。

![予測タイムライン: 3 ヶ月・6 ヶ月・1 年後](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-23_codex-vs-claude-mcp-browser-delegate/fig-4.png)

このタイムラインはあくまで私見で、外れる可能性が高い箇所も多々あります。特に OpenAI が Chrome 拡張を出す/出さないの判断は完全に主観です。それでも「1 年は非対称が続く」だけは高い確度で言えると思っています。

## リスクと制約

冷静な視点も置いておきます。この構想の弱点。

### 認証/セキュリティのリスク

- Claude for Chrome 拡張は「開いている Chrome タブへの操作権」を Claude に渡すことになる
- 悪意ある Web ページに Claude が誘導されると、ログイン済みセッションを使って何かされる可能性
- Anthropic は権限モデル（サイト単位の許可、明示的な確認）を積んでいるが、完全ではない
- 特に **「Codex から delegate された Claude 子」が browser 経由で不注意にログインセッションを漏らす可能性**は本気で設計する必要がある

### プロトコル依存

- MCP は Anthropic 主導。標準として広がっている最中だが、将来仕様変更で backward compat が壊れる可能性は否定できない
- Chrome 拡張は Chrome の拡張 API に依存。Manifest V3 は既に来ているし、今後の Chrome 側変更で使えなくなる機能が出る可能性

### プロバイダー依存の集中

- Web 経路が使えるのは各サービスの「無料枠」に依存
- Google AI Plus、Anthropic Pro、ChatGPT Plus のサブスクを維持していないと Web 経路の無料生成すらできない
- サブスク価格が上がる or 無料枠が縮小されると経済性が崩れる

### 実装コスト

- `many-ai-cli` に browser-delegate 機能を入れると、コア規模がまた膨らむ
- 保守負担、regression リスク、テスト面積がすべて増える
- 個人 OSS の許容範囲を超える可能性

こういうリスクを踏まえた上でも「面白いし需要はある」というのが私の判断ですが、これは主観です。

## 今日の帰結

長くなったので、まとめます。

- 記事作成スキルを 2 種類の CLI で並列運用したら、片方は無料 Web、片方は課金 API になっていた
- 原因は Anthropic が Chrome 拡張を純正で出しているエコシステムの厚み
- MCP プロトコル自体は中立で、他 CLI でも汎用 MCP を配線すれば同じことはできる
- ここに「Codex 親から Claude 子に browser 部分だけ delegate する」orchestration の空きスペースがある
- 自作の `many-ai-cli` に browser-delegate を pending として起票した（次リリース候補）
- 3 ヶ月〜1 年の予測はハズしていい前提。ただし「Claude だけタダ」の非対称は 1 年は続く見込み

## many-ai-cli はこんなときに刺さります

- Claude Code・Codex・Copilot・Cursor・Grok を同時に走らせて、状態を 1 画面で監視したい方
- 各 CLI の承認待ちをまとめて 1 タブで捌きたい・ターミナルの往復を消したい方
- 離席中もスマホから承認したい方
- そして将来的には、**Codex で書きながら Claude に browser を肩代わりさせる、みたいな運用**を待っている方（本記事の題材）

いずれかに心当たりがあれば、`npm i -g many-ai-cli` で 1 分で試せます。設定ゼロで動きます。

- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/many-ai-cli
- npm: https://www.npmjs.com/package/many-ai-cli

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## おわりに

前作の note 記事も、良ければどうぞ。https://note.com/ishizakahiroshi （公開後に URL 差し替え予定）。「gh run watch の tail 表示を信じたら Validate が赤いままタグを push していた話」でした。この記事はその続き、というか脱線から生まれた副産物です。

3 万字書いてみて分かったのは、**この題材はまだ書ききれていない**ということでした。実装が進めば「実際に browser-delegate を書いてみたら詰まった話」も書けそう。次に書きたいネタが目の前に転がっている、というのは幸せな状態です。

小さく、これからも続けます。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-07-23_codex-vs-claude-mcp-browser-delegate/
