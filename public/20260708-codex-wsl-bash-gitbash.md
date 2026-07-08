---
title: Windows の Codex CLI が WSL bash を勝手に使う問題を、Git Bash に固定して解決する
tags:
  - Windows
  - gitbash
  - WSL
  - OpenAI
  - codex
private: false
updated_at: '2026-07-09T00:08:09+09:00'
id: acec7f7c24db713cfcbe
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-08_codex-wsl-bash_hero.png)

:::note info
この記事は **Windows で Codex CLI を使っている人向け**です。macOS / Linux では起きない問題なので、該当しない方はここで閉じて大丈夫です。
:::

動作環境:

- Windows 11 Pro (x64)
- WSL2 インストール済み(**これが発症の前提条件**。WSL を入れていない環境では起きません)
- Git for Windows インストール済み(Git Bash が PATH に通っている)
- codex-cli 0.140.0 系(npm レジストリ版を pnpm でグローバルインストール)

Windows で Codex CLI に .sh スクリプトを実行させたら、頼んでもいない WSL が立ち上がって失敗しました。結論から書くと、`~/.codex/config.toml` にこれを足せば直ります。

```toml
[windows]
agent_shell = "git-bash"
```

以下、何が起きていたのかと、設定の中身、効いたかどうかの確認方法です。

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-08_codex-wsl-bash_infographic.png)

## 何が起きたか

業務システムのデプロイ用シェルスクリプト(.sh)の実行を Codex に任せたところ、Git Bash がインストール済みで PATH も通っているのに、Codex は WSL の bash を起動しました。WSL 側は Windows 側ファイルの CRLF 改行をそのまま解釈するので、スクリプトは途中で失敗。Codex 自身は「Git Bash + LF の一時コピーで再実行して成功しました」と器用にリカバリーしてくれたのですが、そもそも WSL を立ち上げてほしくない。WSL は本当に必要なときだけ手動で使う運用にしているためです。

厄介なのは、普段のコマンドが PowerShell で実行されることです。日常使いでは何も起きないので気づかない。「.sh を実行するために bash を探す」という特殊経路に入ったときだけ、WSL を掴みにいく挙動でした。

## 原因は PATH を見ていないこと

公式リポジトリに、まさにこの報告がありました。

> Codex completely bypasses PATH resolution and directly uses WSL bash (version 5.1.16)

[Windows: Codex bypasses PATH resolution and incorrectly uses WSL bash instead of user-configured MSYS2 bash #3159](https://github.com/openai/codex/issues/3159)

Codex は bash を PATH から解決せず、WSL bash を直接使うという報告です(2025 年 9 月)。報告者の環境も、MSYS2 系の bash 5.2.37 を入れているのに WSL の 5.1.16 が使われるという、今回とまったく同じ構図でした。issue 内では次の対応が提案されています。

> 1. Respect PATH resolution: Use the bash that `which bash` or `where bash` returns
> 2. Add bash path configuration: Allow manual override in `config.toml`
> 3. Remove hardcoded WSL preference

さらに「そもそもシェルを設定で選ばせてほしい」という要望が別 issue になっていて、

> On Windows, Codex hardcodes PowerShell as the agent shell. (中略) Users with Git Bash installed have no way to switch.

[feat: configurable Windows agent shell (powershell/git-bash) #16717](https://github.com/openai/codex/issues/16717)

ここで提案された `[windows] agent_shell` 設定が実装され、両 issue とも Closed になっています。つまり「直せる状態になったばかり」で、設定を知らないと旧来の挙動を踏む、という時期でした。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-08_codex-wsl-bash_fig1.png)

図にすると、設定なしの Codex は .sh を見て bash を自前解決し、`C:\Windows\System32\bash.exe`(WSL ランチャー)に到達してしまいます。`agent_shell = "git-bash"` を入れると、この解決を飛ばして最初から Git Bash に向かいます。

## 対策 1: config.toml で Git Bash に固定する

`~/.codex/config.toml` に 2 行足すだけです。

```toml
[windows]
agent_shell = "git-bash"
```

値は `"power-shell"` か `"git-bash"`。未対応の古いバージョンでは未知キーとして無視されるだけなので、入れておいて損はありません。手元の `codex-cli 0.140.0` 系では効きました。

## 対策 2: グローバル AGENTS.md にも書いておく(保険)

設定が効かない旧バージョンに備えて、`~/.codex/AGENTS.md`(全プロジェクト共通で読み込まれる指示ファイル)にも禁止事項を明文化しておきます。

```markdown
## Windows Shell Selection (WSL 禁止)

- `bash` を裸で呼ばない（C:\Windows\System32\bash.exe = WSL ランチャーに解決される事故が起きる）
- bash が必要な時は必ず Git Bash のフルパス（C:\Program Files\Git\bin\bash.exe）で呼ぶ
- `wsl` / `wsl.exe` を実行しない
- 実行前に `uname` の結果が `Linux`（= WSL）だったら即中断して Git Bash に切り替える
```

モデルへの自然言語指示なので、確実性は設定キーより一段落ちます。ただ「なぜダメか」の理由(CRLF でデプロイが失敗した実害)も添えておくと、従ってくれる率が上がる印象です。

## 効いたかどうかの確認

新しいセッションで一言聞くだけです。

```text
> .sh を実行する時に使う bash で uname -a を実行して

MINGW64_NT-10.0-26200 xxxx 3.6.7 x86_64 Msys
```

`MINGW64_NT`(= Git Bash / MSYS)が返れば固定成功。`Linux` が返ったら、まだ WSL に落ちています。

## ちなみに他の AI エージェントはどうなのか

同じ Windows マシンで使っている他の AI コーディングエージェントも気になって調べました。すると「Codex 特有の癖」だと思っていたこの問題、実はそうでもありませんでした。

問題が起きない組:

- **Claude Code**: Bash ツールがハーネス側で Git Bash に固定されていて、モデルの判断で WSL に落ちる経路がそもそもない
- **Copilot CLI**: Windows ネイティブでは PowerShell 実行(逆に Git Bash を強制する公式設定がない、という不満が [community discussion](https://github.com/orgs/community/discussions/189318) に出ています)
- **Grok CLI**: PowerShell 用のネイティブインストーラーが配布されていて、これも PowerShell 実行

同じ穴に落ちている組:

- **Cursor(Agent)**: 「既定の統合ターミナルを Git Bash にしているのに、Agent のコマンド実行だけ WSL で走る」という今回とそっくりの報告が[公式フォーラムの Bug Reports](https://forum.cursor.com/t/agent-command-execution-uses-wsl-instead-of-the-windows-default-integrated-terminal-git-bash/160196) に上がっています
- **opencode**: `SHELL` 環境変数を Git Bash 指定(`OPENCODE_GIT_BASH_PATH`)より優先してしまい WSL bash を掴む、という報告([#8396](https://github.com/anomalyco/opencode/issues/8396))があり、[Git Bash / cmd / PowerShell を意識したシェル選択の改善提案](https://github.com/anomalyco/opencode/issues/16479)も出ています

つまり「bash を積極的に使おうとするエージェントが、Windows で bash を探すと WSL ランチャーに当たる」という構図は、Codex に限らないこの環境の共通の落とし穴でした。PowerShell に閉じている Copilot や Grok が安全なのは、裏を返せば bash を使いにいかないからです。

## まとめ

- 現象: Codex CLI が .sh 実行時に PATH を無視して WSL bash を起動する([#3159](https://github.com/openai/codex/issues/3159))
- 対策: `~/.codex/config.toml` に `[windows] agent_shell = "git-bash"`([#16717](https://github.com/openai/codex/issues/16717) で実装)、保険でグローバル AGENTS.md にも WSL 禁止を明記
- 確認: `uname -a` が `MINGW64_NT` を返せば OK
- 同じ構図は Cursor(Agent) や opencode でも報告されていて、Windows + AI エージェント共通の落とし穴

WSL は残したい、でも AI に勝手に起動されたくない。そういう運用の人の参考になれば。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
