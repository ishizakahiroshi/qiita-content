---
title: >-
  Claude Code / Codex / Cursor / Copilot / OpenCode で同じ Agent Skills を共有する。正本 1
  箇所 + リンクの設計
tags:
  - AI
  - cursor
  - GitHubCopilot
  - ClaudeCode
  - AgentSkills
private: false
updated_at: '2026-07-10T11:01:40+09:00'
id: 6821655d5af59a32e50c
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![スキル棚は、一つでいい ヒーロー画像](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-10_cross-cli-skill-shelf_hero.png)

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-10_cross-cli-skill-shelf_infographic.png)

この記事の音声版（AI ラジオ解説・17 分）もあります:

https://youtu.be/Ei3EB3W4tvM

手元の AI コーディング CLI、数えたら 6 本ありました。Claude Code、OpenAI Codex CLI、Cursor Agent、GitHub Copilot CLI、OpenCode、Grok Build。そしてこの 1 年で、どれもが **Agent Skills**（`SKILL.md` を持つフォルダ）を扱えるようになってきました。

便利になった一方で、skill の「置き場」がバラバラという問題が出てきます。この記事は、それを **正本 1 箇所 + リンク（Windows ならジャンクション）** で解決したときの設計判断のメモです。

## 結局、こうします（最初にレシピ）

1. skill の正本フォルダを 1 つ決める（git 管理推奨）
2. 常用する各 CLI の **公式 skill スキャンパス** を調べる
3. コピーせず **リンク** する（Windows: ジャンクション / macOS・Linux: シンボリックリンク）
4. 製品に **同梱されている skill ディレクトリは上書きしない**
5. 各 CLI を新セッションで起動して、skill 一覧に出ることを確認する
6. 「編集は正本だけ」をルールとしてメモに残す

Windows のジャンクション張りはこれだけです（管理者権限は不要）:

```powershell
$src = 'C:\dev\skills'   # 正本（git 管理のリポジトリ）
foreach ($rel in @(
  '.claude\skills',
  '.agents\skills',
  '.cursor\skills',
  '.config\opencode\skills',
  '.copilot\skills'
)) {
  $p = Join-Path $env:USERPROFILE $rel
  if (Test-Path $p) { continue }   # 既存があれば触らない
  $parent = Split-Path $p -Parent
  if (-not (Test-Path $parent)) { New-Item -ItemType Directory -Path $parent -Force | Out-Null }
  New-Item -ItemType Junction -Path $p -Target $src | Out-Null
}
```

以下は「なぜこうしたか」の話です。

## CLI が増えるたびに、skill が散らばっていく

起きていた症状は単純です。

- Claude Code 用に書いた HTML 資料生成の skill が、Codex CLI からは見えない
- Cursor にだけ skill が増えていく
- 同じ手順書を 5 箇所にコピペして、どれが正本か分からなくなる
- 片方だけ更新して、もう片方が古いまま放置される

最後のやつが一番痛い。手順書が 2 つあって中身が微妙に違うとき、人は必ず古い方を引きます。

## 最初の答えは「各ツールの公式パスにコピーすればいい」だった

一見それで済みそうに見えます。各 CLI のドキュメントに「skill はここに置け」と書いてあるのだから、そこへ置けばいい。

でも、コピーはメンテで死にます。skill を 1 行直すたびに 5 箇所へ配る作業が発生して、3 週間後の自分は確実にどこかを配り忘れる。ここで一度立ち止まりました。

![コピー運用と正本+リンク運用の対比図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-10_cross-cli-skill-shelf_fig1.png)

上の図のとおり、コピー運用は編集のたびに N 箇所への同期作業が生まれ、時間とともに「どれが正しいか分からない棚」が増えていきます。正本 + リンク運用なら編集先は常に 1 箇所で、各 CLI は同じ実体を見ているだけです。

## そもそも skill って何なのか、を見直した

きっかけは Grok Build でした。Claude Code 用に書いた skill が、Grok のセッションでそのまま動いたんです。最初は「Claude のランタイムを借りてるのか?」と思ったのですが、違いました。

Grok は既定で Claude 互換の設定（`compat.claude.skills = true`）を持っていて、`~/.claude/skills` をスキャンし、タスクにマッチしたら `SKILL.md` を読んで手順どおり実行しているだけ。つまり **同じディスク上の手順書を、別の CLI が読んで実行している** だけでした。

考えてみれば、skill の中身はこういう束です。

```text
<skill-name>/
  SKILL.md          # 起動条件・手順・制約（実質プロンプト）
  templates/        # 任意: 雛形
  references/       # 任意: 詳細ガイド
  scripts/          # 任意: 実行スクリプト
```

エージェントは起動時に「名前と説明の短い一覧」だけをコンテキストに載せ、必要になったときに SKILL.md 全文を読んで従う。実行主体はその CLI 自身の Read / Write / Shell ツールです。中身は Markdown とファイル資産でしかない。

だとすると、製品ごとの差は実質 3 つに絞られます。

- どのディレクトリをスキャンするか
- 呼び出し方の UI（スラッシュコマンド・専用ツール・暗黙マッチ）
- skill 内が前提にしている専用ツールの有無

このうち 2 番目と 3 番目は skill の書き方で吸収できます。残るのは 1 番目、**置き場の差だけ**。コピー運用の前提が、ここで崩れました。

## 各 CLI が skill を探す場所（2026-07 時点）

ユーザースコープ（ホームディレクトリ側）の主なスキャンパスを整理するとこうなります。

| CLI | ユーザースコープの主なパス |
|---|---|
| Claude Code | `~/.claude/skills/` |
| Codex CLI | `~/.agents/skills/`（同梱 skill は `~/.codex/skills/.system`） |
| Cursor Agent | `~/.agents/skills/`・`~/.cursor/skills/`（Claude / Codex 互換パスも読む） |
| GitHub Copilot CLI | `~/.copilot/skills/`・`~/.claude/skills/`・`~/.agents/skills/` |
| OpenCode | `~/.config/opencode/skills/` |
| Grok Build | `~/.grok/skills/` + compat 設定で `~/.claude/skills/` |

各社の呼び方は違っても、「SKILL.md + フォルダ」というフォーマット自体は事実上の交差標準に収束しつつあります。ただしスキャンパスは製品更新で変わりうるので、この表は鵜呑みにせず各公式 docs を正としてください（本記事の表も「配線を決めたときの地図」くらいの扱いです）。

![各 CLI から正本フォルダへの配線マップ](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-10_cross-cli-skill-shelf_fig2.png)

図にすると、各 CLI の公式パスからリンクの矢印が 1 つの正本フォルダに集まる形になります。編集は正本だけ、各 CLI は自分の公式パス越しに同じ棚を見る、という配線です。

## 置き場だけが違うなら、正本 1 つをリンクで見せればいい

そこで冒頭のレシピに至ります。正本フォルダを 1 つ決めて git 管理し、各 CLI の公式パスにはジャンクション（macOS / Linux なら symlink）を張る。Windows のジャンクションは管理者権限なしで作れて、ディレクトリ単位で「同じ中身」を複数パスから見せられます。

ただし 2 つだけ、やってはいけないことがあります。

**1 つ目、中身のある既存ディレクトリを無確認で置換しない。** 特に Codex CLI の `~/.codex/skills` には同梱 skill（`.system` 配下）が入っています。ここを丸ごと差し替えると同梱機能が隠れて消えます。Codex には `~/.agents/skills` という別の公式経路があるので、そちらへ張るのが安全です。

**2 つ目、この配線を公開リポジトリの README に書かない。** ホーム側のリンク構成は個人環境の話です。チームで skill を共有したいなら、リポジトリ内の `.claude/skills` / `.agents/skills` に置くのが正攻法で、ホームのリンクは個人の横断棚に留めます。

## 横断しやすい skill の書き方

棚を揃えても、skill 本体が特定製品べったりだと他の CLI で詰まります。書き方の側で効いたのはこのあたりでした。

- 手順を「このファイルを Read して従う」形で書く（特定製品の専用ツール名に依存しない）
- ファイル名は `SKILL.md` の大文字必須（小文字だと認識しない CLI があります）
- 起動語・トリガー条件を description に書いておく（暗黙マッチ用）
- テンプレや参照ファイルは skill フォルダからの相対パスで指す

逆に「期待しすぎないこと」も決めておきました。同じ手順書が読めることと、UX が完全に一致することは別です。暗黙マッチの精度は CLI ごとに違うし、skill 内のシェルスクリプトが別 CLI のサンドボックスで動く保証もない。あと地味なところで、リンクを張った直後の起動済みセッションは skill 一覧を更新しません。確認は必ず新セッションで。

## 学んだこと

- skill は製品の魔法ではなく、ディスク上の手順書。だから正本は 1 つでいい
- 各 CLI の差は最終的に「スキャンパス」に集約される。差分は小さな表で持てば怖くない
- 製品同梱の隠れた資産は壊さない。「空でないディレクトリは置換しない」を機械的なルールにする
- チーム共有はホームのリンクではなく、リポジトリ内 skill で

公式のスキャンパスは製品更新で変わるので、年 1 回くらいは見直すつもりです。Copilot CLI のグローバル発見あたりはまだ環境差がありそうで、ここは様子見。それでも、「どれが正本だっけ」と考える時間がゼロになった効果は思った以上でした。

棚は一つ。編集も一箇所。これだけの話に、思ったより長く遠回りしました。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
