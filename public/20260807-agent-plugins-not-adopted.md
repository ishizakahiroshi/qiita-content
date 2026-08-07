---
title: "Agent Plugins を自作ツールに取り込むか検討して、見送りました。配布モデルが逆向きだった"
tags:
  - AI
  - OpenAI
  - ClaudeCode
  - MCP
  - 個人開発
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![標準に乗らない Agent Plugins を見送った理由](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-07_agent-plugins-not-adopted_hero.png)

金曜の昼、X に OpenAI Developers のポストが流れてきました。Agent Plugins という新しい標準が出たらしい。表示回数 157 万。技術ステアリングコミッティに Amazon、Cursor、Microsoft、OpenAI、Vercel が並んでいます。

https://x.com/OpenAIDevs/status/2085398373511918022

AI コーディング CLI を 6 本並べて使っている身としては、見出しだけで手が止まりました。「一度書けば、対応するどのエージェントクライアントでも使える」。まさに去年から地味に困っていたところです。

で、自作ツールに取り込むか半日かけて検討して、見送りました。この記事はその判断の記録です。OpenAI を悪く言う話ではありません。むしろ標準の中身は素直に良くて、ただ自分の位置から見ると必要になる場面が来ていなかった、というだけの話です。

## 複数の AI CLI を 1 画面で承認するツールを作っています

自作で [many-ai-cli](https://github.com/ishizakahiroshi/many-ai-cli) というローカル Web ダッシュボードを作っています。複数の AI コーディング CLI を並列で走らせ、承認をブラウザ 1 タブに集約。スマホからでも。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/many-ai-cli

同じ悩みを持っている方は、下記で入ります。

```bash
npm install -g many-ai-cli
```

この話には前があります。AI コーディング CLI が 6 本並立して Agent Skills（`SKILL.md`）の置き場がバラバラになったので、正本フォルダを 1 つ決めて各 CLI の公式パスにジャンクションで見せる設計に落ち着いた、という記事を 1 か月前に書きました。

前回の記事: [Claude Code / Codex / Cursor / Copilot / OpenCode で同じ Agent Skills を共有する。正本 1 箇所 + リンクの設計](https://qiita.com/ishizakahiroshi/items/6821655d5af59a32e50c)

今回は、その 1 か月後に業界標準が出てきたときの答え合わせにあたります。

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-07_agent-plugins-not-adopted_infographic.png)

## Agent Plugins は、驚くほど小さい仕様だった

まず中身を読みました。仕様本体はここです。

https://agent-plugins.org/specification

読み終えて最初に思ったのは「小さいな」でした。褒め言葉です。

- `plugin.json`（必須。必須フィールドは `$schema` と `name` だけ）
- `skills/<name>/SKILL.md`（Agent Skills 仕様に準拠。任意）
- `mcp.json`（MCP サーバー定義。任意）
- `com.example.client/` 形式の逆ドメイン名ディレクトリで、クライアント固有の拡張

ディレクトリはこれだけです。

```
my-plugin/
├── plugin.json
├── skills/
│   └── summarize/
│       └── SKILL.md
├── mcp.json
└── com.example.client/
```

stdio の MCP サーバーには `PLUGIN_ROOT` と `PLUGIN_DATA` を必ず渡す、`args` と `env` と `cwd` の中では `${PLUGIN_ROOT}` を展開する、というあたりまで決まっています。個別コンポーネントの失敗が他をブロックしない、という耐障害の書き方も丁寧でした。壊れた `SKILL.md` は skip、壊れた MCP エントリも skip、プラグイン全体は生き残る。

仕様として素直です。ここに文句はない。

## 「一度書けばどこでも」に、一瞬心が動いた

自分の困りごとは去年からずっと同じで、Claude Code と Codex と Cursor と Copilot と OpenCode と Grok を同時に使っていると、**同じ手順書をそれぞれの流儀で置き直す羽目になる**ことでした。

skill の置き場がバラバラ。MCP の設定形式もバラバラ。片方を直すともう片方が古くなる。

だから「共通形式で 1 個作れば全部に届く」という売り文句は、正直かなり刺さりました。Vercel の告知記事も読みました。

https://vercel.com/blog/introducing-agent-plugins

Google の開発者ブログも同日に出ています。

https://developers.googleblog.com/agent-plugins-package-your-skills-tools-and-more/

ここまでは完全に前のめりでした。問題は、自分の手元を見直したところからです。

## Skills の共有は、1 か月前に自分で解いていた

冷静になって気づきました。Agent Plugins の 2 つのコンポーネントのうち、**Skills 側は自分の環境ではもう解決済み**です。

正本のフォルダを 1 つ決めて git 管理下に置き、各 CLI が読むパスへディレクトリジャンクションで同じ実体を見せています。Windows の `New-Item -ItemType Junction` なので管理者権限も要りません。

```powershell
$src = 'D:\dev\workshop\skills'
foreach ($rel in @(
  '.claude\skills',
  '.agents\skills',
  '.cursor\skills',
  '.config\opencode\skills',
  '.copilot\skills'
)) {
  $p = Join-Path $env:USERPROFILE $rel
  if (Test-Path $p) { continue }
  $parent = Split-Path $p -Parent
  if (-not (Test-Path $parent)) { New-Item -ItemType Directory -Path $parent -Force | Out-Null }
  New-Item -ItemType Junction -Path $p -Target $src | Out-Null
}
```

各 CLI は Agent Skills 形式（`<name>/SKILL.md`）を自分のスキャン先から読むだけなので、**プラグイン標準に対応しているかどうかに関係なく届きます**。

そして運用としては、こちらの方が上でした。

- install と update のサイクルが無い。正本を 1 か所直せば、次のセッションから全部の CLI に反映される
- 正本が git 管理下なので、標準のバージョニングに頼らずにバージョン管理できる
- クライアント側の標準対応を待つ必要がない

プラグイン方式だと、1 か所直したあとに各クライアントのキャッシュを更新して回ることになります。`~/.codex/plugins/cache/` とか `~/.copilot/installed-plugins/<marketplace>/<plugin>/` とか、置き場はクライアントごとに違う。

正直、ここで熱が半分下がりました。

![新しい標準の告知を見たあと、自分の棚を見返している場面](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-07_agent-plugins-not-adopted_illustration-looking-back-at-shelf.png)

自作の仕組みが先に解いていた、というのは気分がいい話ではあるんですが、同時に「じゃあ何のために半日読んだんだ」という気持ちもあって、複雑でした。

## 残る半分の MCP も、載せるものが無かった

Skills 側が空振りなら、MCP 側はどうか。

`mcp.json` を標準形式で配れるのは確かに便利です。今はクライアントごとに設定の書き方が違うので。

ただ、**many-ai-cli は MCP サーバーを持っていません**。PTY をラップして端末出力を読み、承認待ちを検出して Web UI に出すツールなので、そもそも MCP として提供するものが無い。

つまり Agent Plugins の 2 コンポーネントが、両方とも空振りでした。Skills 側は自前で解決済み、MCP 側は載せるものが無い。仕様の中身が丸ごと余る。

この時点で「取り込まない」に大きく傾きました。

## 決定打は、配布モデルが逆を向いていたこと

とどめはこれでした。

Agent Plugins は「**ユーザーが各クライアントに install するもの**」を想定した形式です。marketplace のコマンドか git URL で入れて、クライアント固有のキャッシュに展開される。

一方 many-ai-cli の承認ルールは、Hub がセッション起動時に自動で注入して、そのファイルを参照するセッションが 0 になったら自動で消します。ユーザーの手順はゼロです。

![配布の向きが逆になっている図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-07_agent-plugins-not-adopted_fig1.png)

上の図のとおり、プラグイン標準はユーザーが各クライアントへ install する向きに流れます。many-ai-cli は逆で、Hub が起動時に注入して終了時に消す向きに流れる。矢印の出どころが違うので、同じ荷物を運んでいても乗り換えられません。

これを plugin 化すると、ユーザーは「many-ai-cli を入れる」に加えて「使う CLI ごとに plugin を install する」が増えます。体験としては、はっきり退化する。

かといって many-ai-cli が各クライアントのプラグインディレクトリへ直接ファイルを置きに行くと、今ある「ユーザーの `CLAUDE.md` を書き換える」問題が、「クライアント別のパス 6 種類に、非公式なやり方で置く」問題に化けるだけです。もっと悪い。

**すでに解けている問題を、手数の多い形に戻す提案**になっていた。標準そのものの出来とは無関係に、自分の立ち位置から見るとそうなっていました。

## 仕様の未成熟も、後押しにはなった

念のため書いておくと、v1.0.0 の時点では次が入っていません。

- hooks
- slash commands
- subagents
- **配布とインストールとレジストリの方法**

承認検出まわりで欲しかったのは hooks と配布の標準化なので、そこは丸ごと空でした。

それから、Anthropic は参加していません。Claude Code は独自の `.claude-plugin/` と marketplace のままです。Codex も Copilot も、現行の実装は今回の標準より前のものでした。発表が昨日なので当然ではあります。

「一度書けばどこでも」が実際に効くのは、早くて来年だと思っています。これは批判ではなく、単に標準というものはそういう速度で普及する、という話です。

## 見送るが、見張ってはおく

というわけで、pending として記録に落としました。着手条件を先に決めておかないと、半年後にまた同じ半日を使うことになるので。

- hooks（または同等のクライアント介入点）が標準に入る
- 配布とインストールの方法が標準化され、「1 コマンドで全 CLI に入る」が成立する
- Claude Code が標準に乗る。乗らないなら、対象にしている 6 プロバイダのうち 4 つ以上が標準形式を読めるようになる

このどれかが欠けているうちは保留のままにします。

判断そのものより、**判断した理由を残したことの方が効く**気がしています。次に似たニュースが来たとき、「配布の向きは自分と同じか」を最初に見ればいい、という物差しが 1 本増えました。

## で、このジャンクションの考え方をツール側にどう還元するか

ここはまだ決めていません。

many-ai-cli のユーザーは、定義上「複数の AI CLI を並列で使っている人」です。つまり skill 棚がバラバラになる問題を、ほぼ全員が持っている。だから何かしら還元したいとは思っています。

選択肢は 3 つ考えました。

1 つ目は、ドキュメントにトピックとして書く。「many-ai-cli を使うなら、skill 棚もこうやって共有すると楽ですよ」という運用の tips として README か docs に足す。実装ゼロで、明日できる。

2 つ目は、Hub の UI に**読み取り専用の診断**を出す。各 CLI のスキャン先を見て、同じ実体を指しているか、バラバラか、未設定かを表示して、設定用のコマンドをコピペできる形で出す。実行はユーザーの手に残す。

3 つ目は、UI からジャンクションを自動作成できるようにする。一番親切に見えますが、これは怖い。

3 つ目が怖い理由は具体的にあって、既存の `~/.claude/skills` に中身がある人のフォルダを置き換えるのは破壊的です。実際、Codex の `~/.codex/skills` には製品同梱の `.system` が入っていて、丸ごと差し替えるとシステム skill が隠れます。自分は 1 か月前にこれを踏みかけて、`~/.agents/skills` 経由で届ける形に逃げました。他人の環境で同じことを自動でやる勇気は、今のところ無い。

あと、many-ai-cli は「承認と監視」が本分です。ファイルシステムを書き換える機能を足すと責務が膨らむし、承認ルールの注入で既にファイル書き換えの信用は使い切っている気がしています。

なので今は「1 → 2 → 3 は作らない」の順で考えています。ただ、これが正しいかは自信がありません。使う人からすると「コマンドをコピペさせるくらいなら押させてくれ」かもしれない。

## many-ai-cli はこんなときに刺さります

- Claude Code・Codex・Copilot・Cursor・Grok を同時に走らせていて、どれが承認待ちなのか分からなくなる人
- ターミナルのウィンドウを行ったり来たりして、承認ボタンを探すのに疲れている人
- 席を外している間も、スマホから承認だけ済ませて先に進めたい人

いずれかに心当たりがあれば、`npm install -g many-ai-cli` で 1 分で試せます。設定ゼロで動きます。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/many-ai-cli

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## あわせて読みたい

- [Claude Code の設定を GitHub に置けないので、Windows のリンクで実体だけ Google ドライブへ逃がした](https://qiita.com/ishizakahiroshi/items/5dd3504c7c180c052b56)（今回と同じ「実体 1 か所 + リンク」の考え方を、skill ではなく設定ファイルに当てた回です）
- [4 つの AI と一緒に、many-ai-cli v0.5.0 を出した話](https://qiita.com/ishizakahiroshi/items/a3f40f08b2185b89c887)（今回見送りを決めた many-ai-cli 本体が、どういう作られ方をしているかの回です）
- [30 個のスキルを積んだ Claude Code に /doctor をかけて、棚卸ししてみた](https://qiita.com/ishizakahiroshi/items/c114346e08dc1f382b0e)（共有した skill 棚が育ちすぎたあとに何が起きるか、という続きの話です）

## おわりに

新しい標準が出ると、乗らないと置いていかれる気がします。今回もそうでした。

でも半日読んで分かったのは、標準が解こうとしている問題を、自分はもう別のやり方で解いていたということでした。しかも運用としてはそっちの方が軽かった。

だからといって「標準は要らない」とは全く思っていません。他人に配るときや、環境を跨いで持ち運ぶときには、ジャンクションでは絶対に足りない。ただ、それが必要になる場面が自分にまだ来ていなかった、というだけです。

乗るかどうかを決める前に、まず自分がどこで困っているかを書き出す。今回はそれで済みました。次も同じでいけるかは分かりませんが。

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
