---
title: Grok Bot のサンドボックス内でしか再現しないバグを、手元から詰める。Tailscale SSH を通すまで人力の伝書鳩をしていた話
tags:
  - tailscale
  - GrokBot
  - AIエージェント
  - debug
  - Terminal
private: false
updated_at: '2026-09-02T00:16:22+09:00'
id: 6a9edc768efea629bf07
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![ヒーロー（記事トップ）](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_tailscale-ssh_hero.png)

![記事全体の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_tailscale-ssh_infographic.png)

## 主題: Grok Bot が持っているマシンの中を、手元からどう調べるか

前回、Grok Bot に付いてくるクラウド PC（Debian 13 / 8 コア / 16GB）へ自作 CLI を入れた記事を書きました。そこでバグを 1 つ踏みました。

厄介だったのは、**バグが出たマシンが Grok Bot の持ち物である**ことです。入口は Grok Bot のクライアント経由だけで、RDP も SSH も無い。手元の開発環境からはファイルもプロセスも見えません。**そのマシンの中を調べられるのは Grok Bot だけ**です。

結果、こうなりました。

1. 手元の Claude Code がコードを読み、調査手順をテキストで出す
2. **人間がコピーして Grok Bot へ貼る**
3. Grok Bot が実行し、結果をテキストで返す
4. **人間がコピーして Claude Code へ貼る**

1 往復あたり人間のコピペが 2 回。運んでいる中身は `ls -l` と `head -c 4096 | xxd` の出力です。

そしてもう 1 つ。**前回の記事で書いた原因は誤りでした。** 「Claude Code が `CSI 6n`（DSR）を投げていて、表示専用の xterm.js が応答を返せないから固まっている」と書きましたが、そうではありませんでした。

https://qiita.com/ishizakahiroshi/items/9818ab40e7a30e42cba7

実測でこうなりました。

- Claude Code の実 PTY ログに `ESC[6n` は **0 個**。同じ環境の Codex には 1 個あり、その Codex は応答が返らないまま正常に描画している
- Claude Code のバイナリを全走査しても部分文字列 `[6n` は 0 件（対照の `[?1049h` は 3 件検出できるので、走査自体は効いている）
- ログにはスプラッシュの描画バイトが最後まで揃っている。**固まっていない**

真因は自分のコード側でした。**Web UI の表示フィルタが、単独 CR の後ろへ無条件に `ESC[K`（EL: 行末まで消去）を挿していた**こと。Claude Code は alt screen 上で「1 行描く → `\r` で行頭へ戻る → カーソル下移動」と描くので、挿入した EL が描いたばかりの行を毎回消していました。

**この 2 つは無関係ではありません。** Grok Bot のマシンの中でしか再現しないから、生ログを 1 度も見ないまま推論だけで原因を書くことになったからです。以下、その経緯と、Tailscale SSH で伝書鳩を畳むまでを実測値と一緒に並べます。

## many-ai-cli というローカルダッシュボードを作っています

自作で many-ai-cli という AI ツールを作っています。複数の AI コーディング CLI を並列で走らせ、承認をブラウザ 1 タブに集約。スマホからでも。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/many-ai-cli

同じ悩みを持っている方は、下記で入ります。

```bash
pnpm add -g many-ai-cli
# 代替（同じ npm レジストリ）
bun install -g many-ai-cli
npm install -g many-ai-cli
```

入れたら 1 回だけ `many-ai-cli setup` を実行します。グローバル bin をシェルがまだ拾っていない場合は `pnpm exec many-ai-cli setup` でも同じショートカットが作られます。

アップデートは、入れたときと同じ経路で上げます。

```bash
pnpm add -g many-ai-cli@latest        # npm レジストリ経由で入れた場合
winget upgrade ishizakahiroshi.many-ai-cli   # Windows / winget で入れた場合
brew upgrade --cask ishizakahiroshi/tap/many-ai-cli   # macOS / Homebrew で入れた場合
```

起動後の見え方は OS で違います。

Windows では `setup` がデスクトップに「MANY-AI-CLI」ショートカットを作ります。押すとトレイに常駐し、トレイメニューの「Hub を開く」でブラウザが開きます。停止は同じメニューの「Hub を停止」。

macOS と Linux では「Many AI Hub Start」と「Many AI Hub Stop」の 2 つ（`.command` / `.desktop`）が作られます。Start を押すとコンソール窓がブラウザと一緒に開きますが、そのコンソールが Hub サーバー本体なので閉じずに最小化してください。

ターミナルから直接なら全 OS 共通で `many-ai-cli serve --open` です。

実環境で動作を確認できているのは Windows で、ネイティブの Linux と macOS はまだ検証が十分ではありません。本記事は、その未検証の環境を借り物のサンドボックスで踏んだときの後始末です。

### 前回の記事

Grok Bot に付いてくるクラウド PC（Debian 13 / 8 コア / 16GB）へ、この CLI を入れて動かした回です。本記事はその続編で、あの記事の結論の訂正でもあります。

https://qiita.com/ishizakahiroshi/items/9818ab40e7a30e42cba7

## 環境と、何が起きていたか

- 対象マシン: **Grok Bot のサンドボックス**（Linux / kernel 6.12.94+）。所有者は自分ではなく、入口は Grok Bot のクライアント経由のみ
- many-ai-cli 0.7.0、Claude Code 2.1.251（native binary）
- 症状: Web UI から spawn した Claude のペインだけ真っ黒。Codex と Grok は同じ Hub で正常描画
- 同じマシンで `claude` を端末から素で起動すると正常に描画する
- 手元の Windows Hub では再発しない

`systemd` が無く、箱を作り直すと `apt` で入れたものは消えます。常駐サービスを置く前提の環境ではありません。この「借り物の箱」という性質が、以下の遠回りの前提になります。

![2 台の AI の間で人間がテキストを運んでいる場面](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_tailscale-ssh_illust1.png)

調べているのは機械、実行するのも機械。運んでいるのだけが人間、という構図でした。

## 生ログが無い状態では、AI は推論しか出せない

前回の結論はこの伝書鳩フェーズで出たものでした。**Grok Bot のマシンに手が届かない側（＝自分）は、Grok Bot が返してきたテキストしか材料を持っていません。** そこで Grok Bot 側へ確認したところ、根拠になる生 PTY ログを **1 度も見ていません**でした。`session_enabled: false` で `~/.many-ai-cli/logs/sessions/` 自体が存在しなかったからです。

そこから組み立てられた説明はこうでした。

- Hub の xterm は `disableStdin: true` で応答を返す口が無い
- 同じマシンの xfce では同じ argv/env で描画する。Windows Hub も描画する。Linux Hub だけ黒い
- `/proc` を見ると子は raw モード、PTY もあり、winsize も正常、ただし wrap の IO 量が Codex/Grok より極端に少ない
- よって「端末問い合わせ待ち」ではないか

筋は通っています。自分も納得しました。**が、これは観測ではなく推論です。**

そこで `session_enabled: true` にして再現してもらいました。ここで話が一変します。

## 決め手は「ログにバイトはあるのに画面が空」

取れた Claude のログは 5866 バイト。中身にはスプラッシュの描画データが完全に入っていました。`cat -v` で先頭を見ると、こうなっています。

```text
ESC 7 ESC[r ESC 8 ESC[?25h ESC[?25l ESC[?2004h ESC[?1004h ESC[?2031h ESC[>0q ESC[c
ESC[?1049h ESC[2J ESC[H ... ▐▛███▜▌ ESC[12G Claude ESC[19G Code ESC[24G v2.1.251 CR
ESC[1B ▝▜█████▛▘ ESC[12G Opus 5 ... CR ESC[1B ...
```

**出力はある。画面だけが空。** この組み合わせで、疑う場所が AI 側から自分の表示パイプラインへ移りました。

ちなみに `ESC[c`（Primary DA）は 1 回出ています。前回の調査で自分は「DSR も DA も発行していない」と書きましたが、DA については誤りでした。バイナリ全走査で生 `ESC [ c` が 1 件だけ出たのを「埋め込みデータ領域だろう」と切り捨てたのが原因です。**全走査で 1 件だけ出たものをノイズと決めるなら、その 1 件を開いて確かめるべきでした。** ただし Claude はその応答を待たずに描き切っているので、無応答は主因になりません。

## 真因: CR の後ろに EL を挿していた

`web/src/app/terminal.ts` の表示フィルタが、**直後が LF でない CR（単独 CR）の後ろへ無条件に `ESC[K` を挿入**していました。進捗バーのような同一行の上書きで、短い文字列が古い文字列の末尾を残す問題への対策です。provider 分岐はなく全セッションに掛かります。

Claude Code は alt buffer 上で次の順に描きます。

1. 1 行ぶん描く（`ESC[12G` などで列を移動しながら）
2. `\r` で行頭へ戻る
3. `ESC[1B` で 1 行下へ
4. 1 に戻る

取得したログで **単独 CR が 38 個、LF が 0 個**。つまり挿入された EL が、いま描いたばかりの行を毎回消していました。

![PTY からペインまでの経路と EL の挿入点](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_tailscale-ssh_qiita_fig1.png)

PTY から出たバイトはフィルタを 5 段通って xterm.js に着きます。問題の EL 挿入は最後の段にありました。

再現は `@xterm/headless` に実 PTY 173x22 で流し込んで、描画後の可視文字数を数えました。

| ログ | 素の PTY 出力 | 現行の表示経路を通したあと |
|---|---|---|
| claude s1（Hub spawn） | 674 文字 / 11 行 | **73 文字 / 1 行** |
| claude s4（xfce から手動 wrap） | 641 文字 / 10 行 | 80 文字 / 3 行 |
| codex s2 | 658 文字 | 658 文字（単独 CR 0 個） |

**Windows で露見しない理由**も測りました。手元の Windows セッションログ 6 本（10〜36 万バイト）は **いずれも単独 CR が 0 個**です。ConPTY が画面を作り直して出すため、消す対象がそもそも流れてきません。同じコードが同じように動いていて、被害が出るのは片方だけでした。

![前回の説明と実際の原因の対比](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_tailscale-ssh_qiita_fig2.png)

前回は「AI が応答待ちで固まっている」と書きましたが、実際は「描き切ったあとに消されていた」でした。症状の見た目は同じでも、直す場所が正反対です。

## 修正: alt screen では EL を挿さない

この防御は **main buffer の scrollback に残留が化石化する**ことへの対策で、scrollback を持たない alt buffer では不要です。同じ理屈は cursor-hide フィルタ側に既に入っていたので、そこと形を揃えました。provider 名での分岐は要りません。

```ts
// web/src/app/cr-erase-filter.ts
export function filterBareCarriageReturnPure(
  bytes: Uint8Array,
  state: CarriageReturnFilterState,
): { out: Uint8Array; state: CarriageReturnFilterState } {
  // ... ?1049h / ?1049l をストリーム内で自前追跡して altScreen を持ち回る
  if (combined[i] === 0x0d) {
    out.push(combined[i]);
    if (combined[i + 1] !== 0x0a && !altScreen) {
      for (const b of eraseLineSeq) out.push(b); // ← alt screen 中は挿さない
    }
  }
}
```

`?1049h` / `?1049l` がチャンク跨ぎで分割されると alt 判定が反転して挙動が逆になるため、確定するまで carry へ送ります（末尾 CR の carry と同じ扱い）。fixture は 10 件書きました。alt 中に 1 バイトも増えないこと、alt を抜けたら再開すること、main buffer 側の防御が落ちていないこと、分割チャンクでの判定、末尾 CR の carry です。

## Grok Bot のサンドボックスへ、手元から直接入れるようにする

ここまでは、まだ人力で往復していた時間帯の話です。前回の記事の最後に「このサンドボックスに Tailscale は入る」と書いていたので、それを使いました。

Tailscale で tailnet には載っていました。`tailscale ping` は通ります。それなのに 22 番は接続タイムアウト。理由は 2 つありました。

1. 対象ノードで Tailscale SSH が有効でなかった（`tailscale set --ssh` 未実行）
2. **tailnet の ACL が別マシン宛ての許可しか持っておらず、当該ノード宛てのパケットが全部落ちていた**

2 が見落としやすいところでした。`tailscale ping` は疎通するので「つながっている」と錯覚します。ACL に足したのはこれだけです。

```json
    // 対象ノードへ到達可。22 = Tailscale SSH、443 = tailscale serve
    {
      "action": "accept",
      "src":    ["<自分のアカウント>"],
      "dst":    ["<マシン名>:22,443"],
    },
  ],
  // Tailscale SSH の許可。acls とは別セクションで、無いと
  // 「SSH は有効だが誰も接続できない」状態になる
  "ssh": [
    {
      "action": "accept",
      "src":    ["autogroup:member"],
      "dst":    ["autogroup:self"],
      "users":  ["autogroup:nonroot"],
    },
  ],
```

![伝書鳩の経路と、SSH で直結したあとの経路](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_tailscale-ssh_qiita_fig3.png)

人間が 2 回コピペしていた 1 往復が、そのまま 1 本のコマンドになります。

![往復が消えて 1 本の線でつながる場面](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_tailscale-ssh_illust2.png)

経路が 1 本になると、調査と修正と検証が同じターンの中に収まります。

保存した直後から、**手元の AI が Grok Bot のサンドボックスへ直接入れるようになりました。** そこから先は 1 ターンです。

```bash
ssh <user>@<tailnet の IP> 'cd ~/dev/github/public/many-ai-cli && git pull --ff-only && \
  (cd web && bun run build) && CGO_ENABLED=0 go build -o dist/linux/many-ai-cli ./cmd/many-ai-cli'
```

Hub を上げ直し、セッションは Hub の API から起こしました。**token をシェル変数で受けてそのまま使い、標準出力へは出しません**（会話ログにもスクロールバックにも残さないため）。

```bash
T=$(sed -n 's|.*Open:[^h]*http://127.0.0.1:[0-9]*/?token=\([^ ]*\).*|\1|p' /tmp/hub.log | head -1)
for p in claude codex grok; do
  curl -s -o /dev/null -w "$p HTTP=%{http_code}\n" \
    -X POST "http://127.0.0.1:47777/api/spawn?token=$T" \
    -H "Content-Type: application/json" -H "Origin: http://127.0.0.1:47777" \
    -d "{\"provider\":\"$p\",\"cwd\":\"/home/<user>/dev/github/public/many-ai-cli\"}"
done
```

Hub は Host / Origin を `127.0.0.1:<port>` で検証するので、リモートで叩くときも loopback 宛てにします。ここを外すと `host not allowed` で 403 になります。

Tailscale そのものの導入と概念は、まさおさんのこの動画が分かりやすいです。ACL で詰まったときに見返しました。

https://www.youtube.com/watch?v=ADYYGLTV6oI

## ついでに見つかった別件: NO_COLOR の継承

直接入れるようになって、生ログを自分で数えられるようになりました。そこで別の問題が出ます。**そのマシンのセッションは全部モノクロ**でした。

| セッション | SGR 総数 | うち色指定 |
|---|---|---|
| Linux claude | 0 | 0 |
| Linux codex | 478 | 0 |
| Windows claude（同日） | 25871 | 15132 |

原因は、AI エージェントのハーネスが `TERM=dumb NO_COLOR=1 FORCE_COLOR=0` を持っていて、それが `os.Environ()` 経由でそのまま子 CLI へ渡っていたことでした。wrap は `TERM` と `COLORTERM` は上書きしていましたが、`NO_COLOR` は素通しでした。

`FORCE_COLOR=3` と `CLICOLOR_FORCE=1` を足したところ、**Claude だけ**色が戻り（色指定 0 → 91 個）、Codex は SGR 689 個すべて非色、Grok も SGR 40 個すべて非色のままでした。`FORCE_COLOR` を先に見る実装（Claude Code 同梱の bun ランタイムの `getColorDepth`）では上書きが効き、そうでない実装では継承した `NO_COLOR` が勝っていたわけです。

そこで `NO_COLOR` をキーごと落としました。`NO_COLOR` の仕様では空文字は未設定と同じ扱いですが、**「値が空か」ではなく「変数が存在するか」だけを見る色判定もある**ため、空文字を渡すのではなくキーごと除きます。

結果、Codex は色指定 352 個、Grok は 11666 個。両方直りました。

## 設定は、config を書ける人だけのものにしない

ここで設計の判断が要りました。`NO_COLOR` は端末の能力ではなく **人の意思表示** です。`TERM` の上書きは「このペインは 256 色出せる」という能力の訂正なので筋が通りますが、`NO_COLOR` を黙って無視するのは 1 段踏み込んでいます。目の都合で色を切っている人もいます。

最初は `hub.force_color: false` という真偽値を config.yaml に足しました。ここで 2 つ問題が出ます。

**1 つ目。隠しフォルダの YAML を字下げ込みで直せる人しか降りられない。** 困っている本人は「色が付いて見づらい」としか思っていないので、`force_color` という語にも辿り着けません。逃げ道と呼べません。

**2 つ目。真偽値では嘘になる。** `false` は「上書きをやめる」だけで、起動元に `NO_COLOR` が無ければ色は消えません。画面に「色を付けない」と書く以上、確実に消える口が必要です。

なので 3 択にして、設定画面のプルダウンへ出しました。

| 値 | 子 CLI へ渡すもの |
|---|---|
| `force`（既定） | `FORCE_COLOR=3` と `CLICOLOR_FORCE=1` を渡し、継承した `NO_COLOR` を落とす |
| `inherit` | 起動元の環境をそのまま渡す |
| `off` | `NO_COLOR=1` と `FORCE_COLOR=0` を渡し、継承した `CLICOLOR_FORCE` を落とす |

`off` で `FORCE_COLOR=0` まで明示するのは、`NO_COLOR` だけでは `FORCE_COLOR` を先に見る実装に負けるからです。`TERM` / `COLORTERM` の上書きは 3 方針とも維持します（能力の訂正であって好みの上書きではないため）。

## many-ai-cli はこんなときに刺さります

- AI の CLI を複数同時に走らせていて、承認待ちを取りこぼす人
- 作業マシンから離れているあいだも、進捗をスマホで確認したい人
- 複数の AI を行き来していて、どれがどこまで進んだか分からなくなる人
- リモートのマシンで動かしている AI のペインを、手元のブラウザで見たい人

いずれかに心当たりがあれば、`pnpm add -g many-ai-cli` と `many-ai-cli setup` の 2 コマンドで試せます。設定ファイルを書く必要はありません。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/many-ai-cli
- npm: https://www.npmjs.com/package/many-ai-cli

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## おわりに

今回持ち帰ったのは 3 つです。

**AI が持つマシンの中は、こちらから観測できない。** Grok Bot のようにエージェントへ 1 台ずつクラウド PC が付く形が普通になると、その中で起きた不具合は「エージェントが返してくるテキスト」しか手がかりが無くなります。今回の遠回りは全部そこから来ました。観測経路を最初に 1 本通しておくのが、いちばん効きます。

**生ログが無いところから出てくるのは推論であって観測ではない。** そして推論は筋が通っているぶん、納得してしまう。前回は納得して記事にまで書きました。

**全走査で 1 件だけ出たものを、ノイズと決めない。** `ESC [ c` を切り捨てたのがそれでした。1 件なら開いて確かめるほうが早い。

エージェント側のサンドボックスへ手元から入る道を 1 本通しておくと、試せる範囲がかなり広がりそうです。もっと色々遊んで研究してみます。

## あわせて読みたい

Grok Bot のクラウド PC にこの CLI を入れた回です。本記事はその続編にあたります。
https://qiita.com/ishizakahiroshi/items/9818ab40e7a30e42cba7

観測用のコードをリリース成果物へ混ぜないための build tag と成果物検査の話です。
https://qiita.com/ishizakahiroshi/items/7871f232ac63e2d19a37

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-09-01_grokbot-pc-tailscale-ssh/

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
