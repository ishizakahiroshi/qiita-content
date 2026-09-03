---
title: AI エージェントに開発マシンの鍵を渡していいのか。借り物のサンドボックスを実測して、9 案を比べた
tags:
  - AIエージェント
  - Security
  - Docker
  - GitHub
  - 開発環境
private: false
updated_at: '2026-09-03T13:28:42+09:00'
id: 52030bd9ae36f77124a7
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![ヒーロー（記事トップ）](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-03_ai-key-boundary_hero.png)

![借りている箱と自分のマシン。境界はどこにあったか（記事全体の要約）](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-03_ai-key-boundary_qiita_infographic.png)

## 主題: 借りているサンドボックスと、自分の開発マシン。境界はどこにあるのか

先に結論だけ置きます。

AI エージェントの作業場として借りている Linux サンドボックスから、Claude Max と ChatGPT のサブスクリプション認証を引き上げました。理由は規約違反だったからではありません。**自分が管理していない実行環境に、失うと痛いアカウントの認証を置いていたから**です。

その結果、**箱の中で複数の AI を並べて動かす使い方は設計対象から外しました。** ただし失ったのはそれ 1 つで、複数の AI を使うこと自体は残ります。時間帯が夜から昼へ移るだけです。

そして調べている途中で、もっと大きな穴に気づきました。借りている箱の話をずっとしていたのに、**自分の開発マシン側には境界が 1 本も引かれていなかった**。AI エージェントのアプリが「このコンピューターで実行」を常に許可の設定で持っていて、そのマシンには SSH 鍵もサーバーの接続情報も顧客のナレッジベースも全部あります。

この記事は、その一日の記録です。実測のコマンド出力と、規約の条文と、9 つの選択肢の比較まで。結論はまだ出ていません。試行錯誤の途中です。

そして最後に、書いている当日の朝に流れてきた 1 本のポストの話をします。同じ問題を、ツールを作っている側が先に解きにきていました。

### 前回の記事

この記事は前回の続きです。

前回は「AI と AI の間で手紙を運ぶのをやめて Tailscale SSH で直接つないだら、本当の原因が出てきた」という話でした。その最後に「Grok Bot のサンドボックスへ、手元から直接入れるようにする」と書いています。

- Qiita 版: https://qiita.com/ishizakahiroshi/items/6a9edc768efea629bf07
- note 版: https://note.com/ishizakahiroshi/n/n987039361c75

今回はその続きで、**入れるようになったので中を調べたら、想定と違うものが次々に出てきた**話です。

## その前に。なぜそこまで調べたのか

先に動機を書いておきます。ここが無いと、以下がただ神経質なだけの話に見えます。

**借りているこの環境を、使い倒したい。**

理由は単純で、こちらは寝るからです。借りている作業場は 24 時間動きます。自分は寝ます。手元のパソコンも落とすことがあります。だったら、寝ている間も向こうで手を動かしていてほしい。朝起きたら何か進んでいてほしい。

昼は昼で、別の使い方をします。自分が直接入って作業することもあれば、手元で動いている AI に「向こうを見てきて」と頼むこともある。手元の AI から向こうの AI へ仕事を投げることもある。実際、この検討そのものが「手元の AI が向こうへ入って調べる」やり方で進みました。

つまり **1 つの環境を、時間帯も入口も違う何通りかの方法で使い倒したい**わけです。

ところが使い倒そうとするほど、置くものが増えます。設計書を向こうにも置きたい。ログインも通しておきたい。手元のファイルも触れるようにしたい。夜も動かすなら、止まらないように常時つないでおきたい。

**便利にするための動作が、そのまま「鍵と資料をあちこちに置く」という動作になります。** そこに気づいたので、置く前に中を見ておこうと思いました。

以下はその記録です。

## そもそも何をしようとしていたか

具体的にやりたかったことは単純でした。

各リポジトリの `docs/local/` に置いている設計書と作業メモを、Windows の開発マシンと、借りている Linux サンドボックスの両方から読めるようにしたい。このディレクトリは gitignore してあるので GitHub には載っていません。中身に顧客企業名や担当者名や社内のホスト名が構造的に含まれるからです。

最初に立てた計画はこうでした。Windows 側に共有フォルダを作って SMB で公開し、Tailscale 越しにサンドボックスからマウントする。これなら転送操作を意識せずに同じファイルを両方から触れます。

計画書には「Grok Bot のサンドボックスは 24 時間稼働させる」「Windows は基本常時稼働」と前提が書いてありました。

その前提を、実機で 1 つずつ確かめるところから始めました。ここが今日いちばん効きました。**書いた時点の前提は、書いた時点のものでしかない。**

## 前提 1: Windows はスリープしない、が理由が違った

まず開発マシンの電源設定を見ました。

```powershell
powercfg /q SCHEME_CURRENT SUB_SLEEP STANDBYIDLE
```

AC 電源のスタンバイ待機時間が `0x00000384`（900 秒）と出ました。15 分でスリープに入る設定です。それを「Windows が止まる時間帯がある」として報告しました。

指摘が返ってきました。モニタ OFF だけじゃないのか、と。

確かめ直しました。

```powershell
powercfg /a
```

```
以下のスリープ状態はこのシステムでは利用できません:
    スタンバイ (S1)  システム ファームウェアはこのスタンバイ状態をサポートしていません。
    スタンバイ (S2)  ハイパーバイザーはこのスタンバイ状態をサポートしていません。
    スタンバイ (S3)  ハイパーバイザーはこのスタンバイ状態をサポートしていません。
    休止状態         システム ファームウェアは休止状態をサポートしていません。
    スタンバイ (S0 低電力アイドル)  システム ファームウェアはこのスタンバイ状態をサポートしていません。
```

このマシンは Hyper-V のゲストで、**スリープ状態がひとつも利用できません**。900 秒という設定値は入っていますが、発火する先が存在しないので効きません。10 分の VIDEOIDLE はモニタ OFF だけです。

設定値を読んで「15 分で止まる」と報告したのが誤りでした。**設定が入っていることと、それが効くことは別**です。この日、同じ形の間違いをあと 3 回やります。

## 前提 2: サンドボックスの正体は、想定と違った

計画書は「Grok Bot の Linux サンドボックス」としか書いていませんでした。中身を見に行きます。

```bash
ps -p 1 -o comm=          # tini
cat /proc/1/cgroup        # 0::/agent
ls -la /.dockerenv        # 存在する
findmnt -no FSTYPE,SOURCE /   # overlay overlay
```

Docker コンテナでした。PID 1 は tini、cgroup は `/agent`、ルートは overlay 1 枚。

永続ボリュームを探します。

```bash
findmnt -rno TARGET,FSTYPE,SOURCE | grep -v overlay
```

非 overlay のマウントは `/etc/resolv.conf` と `/etc/hostname` と `/etc/hosts` の 3 本だけでした。これは Docker が常に差し込むものです。つまり**永続ボリュームは 1 つもありません**。ホームも作業ディレクトリも overlay の書き込み層の上にあります。

systemd もありませんでした。

```bash
command -v systemctl cron crond    # 何も返らない
ls /etc/cron.d                     # No such file or directory
```

定期実行の口がゼロです。あとで「夜間に定期実行したい」という要件が出てくるので、ここが効きます。

![借りている箱の中身。永続するものが 1 つもない](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-03_ai-key-boundary_qiita_fig1.png)

図にすると、箱の中身は書き込み層 1 枚の上に全部が乗っているだけです。外から差し込まれているのは Docker が常に付ける 3 本で、永続ボリュームは 0 本。箱が終われば、置いたものは残りません。

## 前提 3: 箱の寿命を握っているのは自分ではなかった

プロセス一覧を眺めていて、見慣れないものがありました。

```
22  sand-exit-watch
```

Python スクリプトでした。中身を読みます。

```python
def end_box_for_owner(self, owner_exit_code):
    self.window.flush()
    if self.supervise_owner is not None:
        print(
            "[start-sand-box] a box lifecycle owner exited (status %d); ending the box"
            % owner_exit_code,
            file=sys.stderr, flush=True,
        )
        sys.exit(1)
```

`box lifecycle owner が exit したら box を終わらせる`。owner は 2 つのプロセスで、どちらも自分が起動したものではありません。

そして既知バイナリの一覧にこう並んでいました。

```python
KNOWN_BINARIES = frozenset([
    "cursor", "cursor-nightly", "cursor-lab",
    "chrome", "node", "exec-daemon", "xvfb", ...
])
```

コメントには `sand/src/host/ports/telemetry.ts` というホスト側の参照。

**Cursor のクラウドサンドボックスでした。** tailnet 上のノード名が `cursor` だったのも、そのままの意味だったわけです。Grok Bot という製品が、Cursor のサンドボックス基盤の上に載っている構成です。

ここで計画書の前提が 1 つ崩れます。「サンドボックスは 24 時間稼働させる」は、**こちらが決められる設定ではありません**。箱の寿命は提供元のバックエンドが握っていて、しかも「再起動」ではなく「終了」です。

## 第一候補が、実装不能だと分かる

計画書の第一候補は SMB マウントでした。試せるか確かめます。

```bash
grep -E "cifs" /proc/filesystems     # 何も返らない
ls -l /sbin/mount.cifs               # No such file or directory
sudo -n modprobe cifs                # sudo: modprobe: command not found
ls /lib/modules                      # No such file or directory
```

`modprobe` コマンド自体が無く、`/lib/modules` も存在しません。**カーネルモジュールを読み込む手段がない**ので、`cifs-utils` を apt で入れても状況は変わりません。

一方で FUSE は使えました。

```bash
ls -l /dev/fuse                                  # crw-rw-rw- 存在する
grep fuse /proc/filesystems                      # nodev fuse
sudo -n mount -t tmpfs none /mnt/probe && echo ok  # ok
```

`/dev/fuse` があり、tmpfs のテストマウントも通ります。つまり rclone の FUSE マウントや sshfs なら道はあります。

そしてもうひとつ、確かめておくべきことがありました。

```bash
timeout 5 bash -c "echo > /dev/tcp/<開発 VM の tailnet IP>/445" && echo tcp445_open
```

`tcp445_open`。まだ共有設定を何もしていない段階で、**Windows の 445 が tailnet から到達可能**でした。これは意図した設定なのか、確認が要る状態です。

計画書の第一候補は、書かれたままの形では成立しませんでした。ここまでで、計画の前提のうち 3 つが実測で崩れています。

## 共有したかったものは、思っていたものではなかった

方式の話に入る前に、規模を測りました。以下の数字はすべて自分の環境を数えたもので、検討シートの公開版にも同じ値を載せています。

https://ishizakahiroshi.com/articles/2026/2026-09-03_ai-local-permission-boundary/review-masked.html

```powershell
Get-ChildItem '<開発ツリー>\github\public' -Directory | ForEach-Object {
  $dl = Join-Path $_.FullName 'docs\local'
  if (Test-Path $dl) {
    $f = Get-ChildItem $dl -Recurse -File -EA SilentlyContinue
    [pscustomobject]@{ repo=$_.Name; files=$f.Count; KB=[math]::Round(($f|Measure-Object Length -Sum).Sum/1KB,0) }
  }
}
```

公開リポ 23 本に `docs/local` があり、合計 **7,848 ファイル / 1.3GB**。

多すぎます。内訳を見ました。

```
repo                 files       KB
----                 -----       --
manabi-map            5860  1312036
many-ai-cli           1425    16556
always-pinned           18     1831
...
```

1 本のリポジトリだけで 5,860 ファイル 1.28GB。さらに掘ります。

```
拡張子別
  .pdf   855 本  622MB
  .csv   951 本  228MB
  .sql   525 本  143MB
  .dump   27 本   97MB
  .bin   300 本   73MB
  .md    789 本    7MB
```

`docs/local` という名前の下にあるものの大半は、設計書ではなく資料とデータの置き場でした。

（この節の数字はすべて自分の環境を数えたものです。同じ値は検討シートの公開版にも載せています。https://ishizakahiroshi.com/articles/2026/2026-09-03_ai-local-permission-boundary/review-masked.html ）

archive と backups を全リポで除くと **770 ファイル / 61.7MB**。さらにその 61.7MB のうち 37.6MB はデータベースのダンプ 5 本。

**本当に共有したかったものは、Markdown 481 ファイル / 6.1MB でした。**

1.3GB を共有する設計を考えていたのに、実体は 6MB。ここで方式の選択肢が一気に変わります。6MB なら git に入ります。

![1.3GB の内訳。共有したかったのは 6.1MB だった](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-03_ai-key-boundary_qiita_fig2.png)

上の図の帯が容量の内訳です。半分近くが PDF で、Markdown は右端の細い一本しかありません。そこから archive と backups、DB ダンプの順に外していくと、残ったのは 481 ファイル 6.1MB でした。

## 「全部そのまま共有する」と決めた後に、内訳が分かった

順番が逆になったのが良くなかった点です。

最初に「全リポの `docs/local` をそのまま共有する」という方針を確認していました。手間ゼロだが、顧客企業名や担当者名を含む md が外部の AI 事業者へ渡りうる前提を受け入れる、という説明で合意していました。

そのあとに内訳が分かりました。DB ダンプ 27 本と SQL 525 本と PDF 855 本が入っていることは、合意した時点では見えていませんでした。

「顧客名を含む md を受け入れる」判断と「DB ダンプを受け入れる」判断は、まったく別の重さです。

**規模を測る前に、共有する範囲を決めていた。** 順序を逆にすべきでした。

## 箱が何時間生きるかを測る。ただし測り方を間違えた

「箱は 24 時間稼働する」が仮定でしかないと分かったので、実測することにしました。

最初に考えたのは、Windows 側から定期的にサンドボックスへ到達性を確認して記録する方法でした。これは即座に指摘されました。

> Windows 側から測ると Windows を止められないじゃないか

そのとおりです。測りたかったのは「開発マシンを落としている間もサンドボックスが生きているか」で、Windows から測る限りその条件を作れません。**計測を設計するときに、測りたい条件下でその計測自体が生きているかを見ていませんでした。**

正しい形はこうです。箱自身が自分の生存を刻んで、それを箱の外へ逃がす。箱が死ねば刻みも止まるので、最後の刻印の時刻が死亡時刻になります。

逃がし先は GitHub にしました。

```sh
#!/bin/sh
set -u
REPO="$HOME/sandbox-heartbeat"
LOG="$REPO/heartbeat.log"
INTERVAL=300
cd "$REPO" || exit 1
while :; do
  ts=$(date -u +%Y-%m-%dT%H:%M:%SZ)
  up=$(cut -d" " -f1 /proc/uptime)
  pid1=$(ps -o etime= -p 1 | tr -d " ")
  hub=$(pgrep -c -f "many-ai-cli wrap" 2>/dev/null || echo 0)
  load=$(cut -d" " -f1-3 /proc/loadavg | tr " " ",")
  printf "%s pid1_etime=%s uptime_s=%s hub_procs=%s load=%s\n" \
    "$ts" "$pid1" "$up" "$hub" "$load" >> "$LOG"
  git add heartbeat.log >/dev/null 2>&1
  git commit -q -m "heartbeat $ts" >/dev/null 2>&1
  git push -q origin HEAD >/dev/null 2>&1
  sleep "$INTERVAL"
done
```

cron が無いので tmux の中で flock 越しに回します。

```bash
tmux new-session -d -s heartbeat "flock -n /tmp/heartbeat.lock $HOME/heartbeat.sh"
```

各行に `pid1_etime`（箱の連続稼働時間）を入れてあるのが要点です。記録が途切れたとき、後から入った箱の稼働時間がその空白をまたいでいれば「死んだのはループのほうで箱は生きていた」と分かります。

副産物として「深夜に箱から GitHub へ push が通るか」も同時に確かめられます。外向き通信が提供元のトンネルを経由しているので、そこも未検証でした。

## 権限モデルを実測する。組織リポの読み取りロールが効いた

共有の設計で、読む場所と書く場所を分けたくなりました。AI が読む資料に AI が書き込めると、正本がどちらか分からなくなります。

最初に考えたのはブランチで分ける案でした。ただしそれは約束であって、守らせる手段が要ります。GitHub のブランチ保護を使おうとしたところで止まりました。

```
$ gh api repos/<owner>/<private-repo>/branches/main/protection
gh: Upgrade to GitHub Pro or make this repository public to enable this feature. (HTTP 403)

$ gh api repos/<owner>/<private-repo>/rulesets
{"message": "Upgrade to GitHub Pro or make this repository public to enable this feature."}
```

**個人アカウントの非公開リポでは、ブランチ保護も ruleset も使えません。** つまり CI で検知はできても阻止はできない。

ここで別の手が見つかりました。無料の organization を 1 つ持っていたので、そこに置いて bot アカウントを読み取りロールで招く形です。実際に試しました。

```
$ gh repo create <org>/perm-probe --private
$ gh api -X PUT repos/<org>/perm-probe/collaborators/<bot> -f permission=pull
invitation_id=... / permissions=read / invitee=<bot>
```

bot 側で招待を承諾してから、実効権限を確認します。

```
$ gh api repos/<org>/perm-probe --jq ".permissions"
{"admin":false,"maintain":false,"pull":true,"push":false,"triage":false}
```

そして実際に clone して push を試しました。

```
$ git clone -q https://github.com/<org>/perm-probe.git
$ echo "bot wrote this" >> README.md && git commit -q -am "bot push test"
$ git push origin HEAD
remote: Write access to repository not granted.
fatal: ... The requested URL returned error: 403
```

リモートの HEAD は変わっていませんでした。**読めるが書けない、が認証の層で成立します。** ブランチ保護も ruleset も PAT も使わず、無料プランのままで。

ブランチ分離は「約束」ですが、これは「権限」です。守らせる手段が最初から要りません。

![約束と権限は違う。読み取りロールは受け付けの時点で拒否する](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-03_ai-key-boundary_qiita_fig3.png)

図の左右で、止まる場所が違います。ブランチで分ける案は書き込み自体が通ってしまい、後から検知するしかない。読み取りロールは、その手前の受け付けで落ちます。

## 読む場所と書く場所を、リポジトリごと分ける

権限が認証の層で効くと分かったので、設計をそこに寄せました。

最初は 1 本のリポジトリの中で `projects/`（読む資料）と `agent-output/`（AI の成果物）に分ける案でした。しかしそれは「約束」です。守らせるにはブランチ保護が要り、それが使えないと分かったところで詰まりました。

そこで**リポジトリを 2 本に分けます**。

```
組織アカウント
├─ dev-context   AI が読む資料。bot は Read ロールのみ
│   ├─ projects/<repo>/     手元から送った設計書
│   ├─ sandbox-policy/      箱の AI に渡す共通ルール
│   └─ manifest.json        いつ時点の資料かの札
│
└─ dev-output    AI が書く成果物。bot は Write
    └─ <repo>/<task-id>/{grok,claude,codex}/
```

読む側を組織リポに置いて bot を読み取りロールにすれば、書き込みは受け付けの時点で拒否されます。書く側は別リポなので普通に書けます。CI もブランチ保護も要りません。

成果物の階層を 3 段（リポ / 課題 / AI）にしたのは、同じ課題に対する複数の AI の案を朝に横並びで読めるようにするためです。1 段だと案が散ります。

正本は手元の `docs/local` のままにします。GitHub 側は「AI へ渡すための投影」であって正本ではない。ここを曖昧にすると、どちらが本物か分からなくなります。

## 写すだけの同期は、古い設計書を現行として残す

この設計を詰めている途中で、抜けに気づきました。

送る処理が「新しいものを写す」だけだと、手元で捨てた設計書が共有側に残り続けます。古さの札は「いつ時点か」は教えますが「消えたものがある」ことは教えません。

夜間に無人で動く前提だと、これは容量の問題ではなく**正確性の問題**です。AI が廃止済みの設計を現行として読み、朝まで誰も気づきません。

なので同期は鏡写しにします。手元で消えたものは共有側からも消す。消した記録は GitHub の履歴に残るので、後から確認もできます。

あわせて、送る対象は**通してよい種類を名指しする方式**にしました。逆の「これは止める」方式だと、書き忘れた拡張子が無警告で通ります。実物の中身を見た後だと、この差は大きい。

```
許可     *.md *.txt *.html *.json *.yaml *.yml （必要なら *.png）
原則除外 archive/ backups/ *.sql *.dump *.db *.sqlite *.bak
         *.zip *.7z *.tar *.gz *.csv *.pdf バイナリ
```

サイズの上限も入れます。種類を絞っても、その種類の中に大きなものが将来入り込むからです。1 ファイルと全体の両方に上限を置いて、超えたらその回の送信を止める。夜間は誰も警告を見ないので、警告だけでは実質上限が無いのと同じになります。

## 古さの札に、コードの版まで入れる

manifest には時刻だけでなく、各リポのソースの commit hash も入れることにしました。

```json
{
  "generated_at": "2026-09-03T03:30:00+09:00",
  "source": "dev-vm",
  "projects": {
    "many-ai-cli": {
      "exported_at": "2026-09-03T03:29:00+09:00",
      "source_commit": "xxxxxxxxxxxx"
    }
  }
}
```

時刻だけだと「3 時間前の資料」までしか分かりません。箱はソースコードを別にクローンしているので、資料とコードの版がずれます。hash があれば「この資料はこのコードのもの」が確定します。

ここで 1 つ落とし穴があります。箱のタイムゾーンは UTC で `/etc/timezone` も無く、`TZ` 環境変数も空でした。

```bash
$ date; cat /etc/timezone; echo "TZ=$TZ"
Wed Sep  3 00:15:54 UTC 2026
cat: /etc/timezone: No such file or directory
TZ=
```

なので manifest の時刻は必ずオフセット付きの ISO8601 で持ちます。ローカル時刻の文字列を書くと、読む側で 9 時間ずれます。

同じ理由で、箱の locale も `POSIX` のままでした。

```bash
$ locale | head -3
LANG=
LANGUAGE=
LC_CTYPE="POSIX"
```

共有するのは日本語の Markdown なので、この状態だと Python 系のツールで文字化けや `UnicodeDecodeError` が出ます。方式に関係なく先に直す項目です。

## ここで話が変わる。規約の話が出てきた

構成が固まってきたところで、根本的な問いが来ました。

> Grok のサブスク（Bot）に Claude とか ChatGPT のサブスク使わせるのって、昔オープンクローが怒られた、サブスクの規約外の使い方とかにならんかね？
>
> そのリスクがあるなら、できたとしても絶対にやらない。Claude と ChatGPT に BAN されるほうが、何百倍も損失が大きい

これは記憶で答えてはいけない類の問題です。条文を引くところから始めました。

まず、実際の仕組みを確定させます。箱の中の認証がどの方式かを、値を読まずにキー名と種別だけで確かめます。

```python
import json
d = json.load(open("$HOME/.claude/.credentials.json"))["claudeAiOauth"]
print("subscriptionType=", d.get("subscriptionType"))   # max
print("scopes=", d.get("scopes"))
# ['user:file_upload','user:inference','user:mcp_servers','user:profile','user:sessions:claude_code']

c = json.load(open("$HOME/.codex/auth.json"))
print("auth_mode=", c.get("auth_mode"))                 # chatgpt
print("api_key_set=", bool(c.get("OPENAI_API_KEY")))    # False
```

Claude Code は API キーではなく **Claude Max のサブスクリプション OAuth**、Codex CLI は `auth_mode=chatgpt` で ChatGPT のサブスクリプション認証。環境変数の API キーは 3 社とも未設定。

どちらも公式 CLI に公式 OAuth という構成で、リバースプロキシ的な迂回ではありません。この切り分けが後で効きます。

## Anthropic の条文を引く

Anthropic の消費者向け利用規約（2025-10-08 発効）にこうあります。

> Except when you are accessing our Services via an Anthropic API Key or where we otherwise explicitly permit it, to access the Services through automated or non-human means, whether through a bot, script, or otherwise.

出典: https://www.anthropic.com/legal/consumer-terms

これだけ読むと「自動化は API キー限定」に見えます。しかし `where we otherwise explicitly permit it`（別途明示的に許可する場合）という例外が置かれています。

例外側の明示を探しました。ヘルプ記事に、`claude -p`（非対話モード）と Agent SDK、そして**サブスクリプション認証で動くサードパーティ製アプリ**の利用がサブスクの利用枠から消費されると書かれています。

出典: https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan

一方でドキュメントは、スクリプトや CI には `--bare` を推奨しています。そしてこのモードは**意図的にサブスクのログインを読まず**、`ANTHROPIC_API_KEY` を要求します。

> `--bare` is the recommended mode for scripted and SDK calls

出典: https://code.claude.com/docs/en/headless

つまり「サブスク認証で自動化する道はあるが、スクリプト用途の推奨は API キー」という位置づけです。

## OpenAI の条文を引く

OpenAI の利用規約（ROW 版・2026-01-01 発効）には、禁止事項として「データ又はアウトプットを自動又はプログラムにより引き出すこと」、アカウント条項として「アカウントの認証情報を他人と共有したり、アカウントを他人に利用可能にしたりしてはなりません」とあります。

出典: https://openai.com/policies/row-terms-of-use/

そのうえで OpenAI には「Maintain Codex account auth in CI/CD (advanced)」という公式ページがあり、ChatGPT アカウント認証の CI 利用を**条件付きで認めています**。

> The right way to authenticate automation is with an API key.

アカウント認証を使うのは「自分の Codex アカウントとして実行する必要がある場合だけ」で、条件は次のとおりです。実行環境が **trusted private infrastructure** であること。1 つの `auth.json` は 1 台または直列実行のみ。`auth.json` はパスワードとして扱うこと。そして公開リポジトリやオープンソースリポジトリでは使うな。

出典: https://learn.chatgpt.com/docs/auth/ci-cd-auth

## 本当のリスクは、想定と別の場所にあった

条文を引き終えて、判定は「条件付き」になりました。禁止されているという根拠は見つかりません。

ところが、調べている途中で別のものを見つけました。

サンドボックスの `/usr/local/bin` に `persist-cli-auth` という実行ファイルがあります。中身を読むと、対象がこう並んでいました。

```bash
CLI_AUTH_TARGETS=(
	.config/gh        # GitHub CLI (gh auth login -> hosts.yml + config.yml)
	.aws              # AWS CLI (credentials + config)
	.config/gcloud    # gcloud (credentials.db + access_tokens.db + configs)
	.ssh              # SSH keys / known_hosts / config
	.docker           # docker login (config.json auths)
	.vercel           # Vercel CLI
	.fly              # Fly.io CLI (legacy ~/.fly)
	.config/fly       # Fly.io CLI (XDG)
	.netrc            # curl / git / many tools' machine logins (file)
	.npmrc            # npm login token (file)
	.gitconfig        # git identity + credential.helper (file)
	.git-credentials  # git `store` credential helper (file)
)
CLI_AUTH_SAVE_INTERVAL_S="${CLI_AUTH_SAVE_INTERVAL_S:-30}"
```

**12 か所の資格情報を 30 秒ごとにミラーする仕組み**です。コード中には「durable store（箱の外の永続保存先）」へ載せる前提の記述もあります。

現在動いているかを確かめました。ここで 3 回目の間違いをやります。

```bash
pgrep -f persist-cli-auth   # → 「稼働中」と判定した
```

これは自分の ssh コマンド行に一致していました。ssh で渡したコマンド文字列に `persist-cli-auth` という語が含まれているので、`pgrep -f` がそれを拾います。自己一致です。

正しく確かめ直します。

```bash
pgrep -a -f "[p]ersist-cli-auth"   # → 該当プロセスなし
find $HOME/cli-config -mindepth 1 | wc -l   # → 0
sleep 30
find $HOME/cli-config -mindepth 1 | wc -l   # → 0
```

ミラー先は空で、30 秒待っても増えません。動いていませんでした。

**これが今日いちばん重要な発見です。** 規約に違反しているかどうか以前に、認証情報を置いた先が、認証情報を外部へ複製する仕組みを持っていて、しかもその有効化がこちらの制御下にありません。

そして重要なのは、**この問題は「Grok が Claude を呼ぶかどうか」とは無関係に成立している**ことです。箱にサブスク認証を置いた時点で、すでにそこにありました。

最初の問いは「Grok に Claude を使わせてよいか」でしたが、本当の問いは「第三者が管理するコンテナに、自分の高価値なアカウント認証を置いてよいか」でした。

![問いの立て方が間違っていた、と気づく](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-03_ai-key-boundary_qiita_illustration-wrong-question.png)

目の前の小さな箱を熱心に調べているあいだ、自分の机の引き出しが開けっぱなしだった、という絵です。この日の構図がだいたいこれでした。

## OpenCode の件は実在した。ただし線の引かれ方が違った

ここで、最初の質問に出てきた「昔オープンクローが怒られた」の話に戻ります。

私は名前を特定できず、「裏が取れていない」として判断材料から外すよう提案しました。これは行きすぎでした。

訂正が入りました。オープンクローではなく **OpenCode** の話だ、と。

調べ直したら、事実でした。しかも一次資料で確認できました。

Claude Code の Legal and compliance ページにこうあります。

> OAuth authentication is intended exclusively for purchasers of Claude Free, Pro, Max, Team, and Enterprise subscription plans and is designed to support ordinary use of Claude Code and other native Anthropic applications.

> Anthropic does not permit third-party developers to offer Claude.ai login into their own applications, or to route requests through Free, Pro, or Max plan credentials on behalf of their users.

続けて、開発者が Claude.ai の資格情報やセッショントークンを収集・保存・仲介してはならず、サインインは Anthropic 自身のフローで完結しなければならない、とあります。

> Anthropic reserves the right to take measures to enforce these restrictions and may do so without prior notice.

出典: https://code.claude.com/docs/en/legal-and-compliance

技術的な裏付けもあります。箱で実測したトークンの scope は `user:sessions:claude_code` で、Claude Code 専用に絞られていました。第三者ツールから使うと `This credential is only authorized for use with Claude Code and cannot be used for other API requests` というエラーになる、という報告と一致します。

ところが同じページに、明示的な除外があります。

> Nor does it prevent an end user from signing in to the unmodified Claude Code binary with their own Claude subscription, including where a platform hosts Claude Code as described under *Can customers offer Claude Code in their products?* above.

**改変していない公式の Claude Code バイナリに、本人が Anthropic 自身のフローでサインインして使うこと**は、プラットフォームがそれをホストしている場合も含めて禁止されていません。

OpenCode が塞がれたのは、自前のクライアントがサブスク認証で API を叩いていたからです。手元の構成は公式バイナリをそのまま起動しているだけなので、線の反対側にありました。

![塞がれた側と、除外された側](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-03_ai-key-boundary_qiita_fig4.png)

並べてみると、線は「サブスク認証を使うかどうか」ではなく「誰が作ったクライアントか」で引かれています。塞がれているのは第三者製のクライアントで、公式バイナリへのサインインは名指しで除外されていました。

残ったグレーは 1 点だけになりました。同じページの Acceptable use にこうあります。

> Advertised usage limits for Pro and Max plans assume ordinary, individual usage of Claude Code and the Agent SDK.

夜通し無人で叩き続けることが `ordinary, individual usage` の範囲かどうか。これは明確な禁止ではなく前提の記述です。

## 認証を引き上げる。logout はサブコマンドではなかった

判定は出ました。規約というより預け先の問題なので、箱から Claude と Codex の認証を外します。

まず「logout で十分か」を確かめました。公式ドキュメントで「removes and **revokes** the credential」と明記されているのは Console 経由のプロファイルの場合だけで、claude.ai のサブスクリプションログインについては失効に触れていません。

出典: https://code.claude.com/docs/en/authentication

トークンの有効期限を見ました。

```
claude accessToken 期限 : 2026-09-02 21:48 UTC
claude refreshToken 期限: 2026-09-27 23:57 UTC
現在                    : 2026-09-02 19:49 UTC
```

**リフレッシュトークンはあと 25 日間有効**です。もし logout がサーバー側の失効までやらないなら、コピーが外に出ていた場合は 25 日間使えます。

それでも削除だけで足りると判断した理由は、複製の形跡が無いからです。ミラー先が空でプロセスも動いていないので、失効させるべき対象がそもそも存在しません。

**「logout すれば失効するから安全」ではなく「コピーが無いから logout で足りる」。** 根拠が違う点は押さえておく必要があります。

実行しました。ここで 4 回目の間違いです。

```bash
$ codex logout
Successfully logged out          # → auth.json 削除を確認

$ claude logout
Either clears the stored credentials; you'll be asked to authenticate again on the next launch.
(Side note: two MCP servers in this session ... need authorization before their tools work.)
```

Claude Code の応答が明らかに会話でした。**`claude` に `logout` サブコマンドは存在せず**、その語がプロンプトとして解釈されて通常の推論が 1 回走っていました。ログアウトはされていません。

そもそもの取り違えは、その前にあります。`claude --help` と `codex --help` の出力をラベルなしで並べて grep していて、`logout  Remove stored authentication credentials` という行を claude のものだと読んでいました。実際は codex 側の行でした。

`/logout` は対話セッション内のスラッシュコマンドです。非対話で外すならファイルを消します。

```bash
rm -f $HOME/.claude/.credentials.json
```

ドキュメントに「Claude Code manages `.credentials.json` through `/login` and `/logout`」とあるとおり、ログアウトの実体はこのファイルの管理です。

`.claude.json` とそのバックアップ 5 件のキー名も確認しましたが、起動回数・インストール方法・機能フラグ・マシン ID だけで、トークンもメールアドレスも組織名もありませんでした。

これで箱に残る高価値の認証は GitHub の bot アカウントだけになりました。

## 引き上げた結果、軸2 が設計対象から落ちた

ここで「何を失ったか」を書いておきます。これが今回いちばん大きな設計変更です。

冒頭で書いた 5 通りの使い方のうち、**軸2（箱の中の AI が、箱の中の別の AI に仕事を投げる）が成立しなくなりました。** 箱の Claude と Codex は、次に呼ばれた時点で認証エラーになります。

実際、この検討をしている間も、箱では 6 つのプロセスが動いていました。

```
many-ai-cli wrap claude --label=verify-claude ...
many-ai-cli wrap codex  --label=verify-codex ...
many-ai-cli wrap grok   --label=verify-grok
many-ai-cli wrap claude --label=colorcheck-claude
many-ai-cli wrap codex  --label=colorcheck-codex
many-ai-cli wrap grok   --label=colorcheck-grok
```

このうち claude と codex のペインが止まります。「夜のうちに 3 つの AI へ同じ課題を投げて、朝に案を並べて比べる」という使い方は、いまはできません。

**成果物を AI ごとに分ける設計も、同時に不要になりました。** リポ / 課題 / AI の 3 階層にしていたのは、同じ課題への複数の案を横並びで読むためでした。書き手が 1 つになったので、階層が 1 段減ります。検討シートでもこの項目を「対象外」にして、消さずに薄く残してあります。将来この線を越えたくなったときに、何を決める必要があったかが分かるようにするためです。

## ただし、失ったものは思ったより狭い

ここは正確に書いておきたいところです。「複数の AI を使う」こと自体は失っていません。

消えたのは軸2 だけです。5 通りのうち、残る 4 つはそのまま動きます。

- 軸1（夜に箱の AI が単独で作業する）は残る。Grok は xAI のサブスクで動き続けます
- 軸3（自分が箱に入って操作する）も残る
- 軸4（手元の AI が箱を操作する）も残る
- 軸5（手元の AI が箱の AI へ依頼する）も残る

そして軸4 と軸5 では、**Claude と Codex は手元のパソコンで動きます。** 認証は自分のマシンにあります。規約の観点でいちばん素直な形です。

つまり複数の AI を使うこと自体は失われず、**時間帯が夜から昼へ移るだけ**です。夜に 3 つ動かして朝に比べる代わりに、昼に手元で 3 つ動かす。この記事を書くための調査も、実際その形でやっています。

夜にどうしても複数の AI が要るなら、選択肢は 2 つあります。従量課金の API キーに切り替えるか、自分の管理下の環境を用意するか。どちらも今日は採りませんでした。**まず夜が 1 つの AI で足りるかを測ってからでいい**と判断したためです。足りないと分かってから足すほうが、要らないものを作らずに済みます。

## 同じ日に、1 セッションで 4 回同じ形の間違いをした

ここで一度整理します。この日、観測の取り違えを 4 回やっています。

1. スリープ設定の値を読んで「15 分で止まる」と報告した。実際はスリープ状態が 1 つも使えないマシンだった
2. `claude --help` と `codex --help` を並べて出力し、codex の行を claude のものと読んだ
3. `stat /proc/1` の mtime をプロセス起動時刻と読んだ（実際は PID 1 の起動時刻ではない。経過時間が要るなら `ps -o etime= -p 1`）
4. `pgrep -f <語>` が自分の ssh コマンド行に一致して「稼働中」と誤報した

共通しているのは、**出力が返ってきたことと、それが調べたい対象を測っていることは別**という点です。しかも壊れた観測は例外を出さず、それらしい正しそうな値を返します。だから出力を眺めても気づけません。

この 4 件は、家のガイドに 1 項目足しました。既存の「数字を訂正するときは、何を測った数字かまで確かめる」の隣です。値の取り違えが親で、こちらは対象そのものの取り違え、という位置づけにしました。

## そして、境界は最初から引かれていなかった

認証を引き上げて一段落した、と思っていたところに 2 枚のスクリーンショットが届きました。

1 枚目は Grok Bot の Windows アプリの設定画面でした。「現在のコンピューター」に開発 VM の名前が入っていて、「このコンピューターで実行」が**常に許可**になっています。説明文には「Bot がコンピューター上のファイルを開いたり、タスクを実行したりできるようにします。自動レビューですべての操作が事前に確認されます」とありました。

2 枚目はアップデート画面で、こう書かれていました。

> Bot が共有するコンピューターをアップデートします。**ファイルとサインイン情報は保持されますが**、インストール済みのアプリとパッケージは削除されます。

サインイン情報は保持される、が製品の仕様として明記されていました。つまり `persist-cli-auth` が動くのは仕様であって、私が見た停止状態は一時的なものです。**「今動いていないから安全」という読み方はできない**、が確定しました。認証を引き上げた判断は、この点で裏付けられました。

そして 1 枚目のほうが、この日いちばん大きな気づきでした。

一日じゅう「借りている箱に認証を置いていいか」を議論していたのに、**その認証を引き上げた先の Windows マシンは、同じベンダーのアプリが常に許可で実行できる場所**でした。

実機を確認します。

```powershell
Get-Process | Where-Object { $_.ProcessName -match 'grok' }
# Grok Bot が 10 プロセス稼働中（Electron 構成）
```

設定ディレクトリも見ました。値は読まず、キー名だけです。

```
sand-secrets.json          → cursor-machine-id / cursor-accounts / local-exec-file-key
box-secrets-push-state.v1.json → version / ackedCount / scopeHash
gateway-descriptor.json    → entries{<hash>{savedAtMs, encrypted}}
```

`box-secrets-push-state` は**Windows 側から箱へ秘密を送る仕組みの状態管理ファイル**です。`ackedCount` の値は 0 でした。

これで話がつながりました。箱の中の `persist-cli-auth` は停止していてミラー先も空、Windows 側の送信カウンタも 0。**仕組みは作り込まれていて配線も済んでいるが、まだ 1 度も動いていない**。認証を引き上げたのは、動く前という良いタイミングでした。

## 同じマシンに何が置いてあるか

「全権限を渡すのが怖い」の中身を書き出します。

```
この 1 台に、全部ある

 ├ サーバーの接続情報       会社のサーバー / 個人のサーバー / DB / 管理画面
 ├ SSH の鍵                7 本
 ├ AI の認証               Claude Max / ChatGPT / gemini / grok / OpenCode
 ├ 顧客のナレッジベース     会社名・担当者名・社内のホスト名
 ├ 家族の情報              続柄・学校・役職
 ├ 非公開リポジトリ         顧客案件を含む
 └ クラウド同期フォルダ     Google ドライブ / Nextcloud 2 系統
```

そして境界を作りにくくしている事実が 2 つありました。

```powershell
(Get-Acl 'D:\dev').Access | Where-Object { $_.IdentityReference -match 'Users|Authenticated' }
```

```
NT AUTHORITY\Authenticated Users    Modify, Synchronize    Allow
BUILTIN\Users                       ReadAndExecute         Allow
```

開発ツリーが「認証されたユーザー」全員に **Modify** を与えています。つまり新しくローカルユーザーを作っただけでは、そのユーザーからも全部見えて書けます。SSH 鍵もサーバー接続情報も含めて。**ユーザーを分けるだけでは境界になりません。**

もう 1 つは逆に良い発見でした。

```powershell
Get-LocalUser | Select-Object Name, Enabled
```

```
CodexSandboxOffline    True
CodexSandboxOnline     True
ishiz                  True
```

`CodexSandboxOffline` と `CodexSandboxOnline` という有効なローカルユーザーが既にあります。所属グループは `CodexSandboxUsers` と `Users` で、管理者ではありません。Codex CLI のサンドボックス機能が作ったものです。

**同じ発想の実例が、このマシンで既に動いていました。**

ただし現在の Codex の設定はこうでした。

```toml
approval_policy = "never"
sandbox_mode = "danger-full-access"

[windows]
sandbox = "elevated"
```

サンドボックスユーザーは存在するのに、設定は承認なしの全アクセスです。**いちばん緩いツールが、そのマシンの実際の境界になります。** 玄関に鍵を付けても、勝手口が開いていれば施錠したことにはなりません。

![同じマシンに全部ある。そして境界は 1 本も無い](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-03_ai-key-boundary_qiita_fig5.png)

図の左が、この 1 台に置いてあるものです。右が境界になりそうだったもので、実測すると 3 つとも効いていませんでした。ユーザーを分けても ACL で素通りし、アプリは常に許可、別のツールは承認なしの全アクセスです。

## 「使い倒したい」の中身を数えたら、5 通りあった

冒頭に書いた動機が、ここで具体的な形になります。

設計を詰めている途中で、こういう指摘がありました。

> そもそも軸が 3 つあるよね。GrokBot 自体がソースを解析するパターンと、GrokBot がサンドボックスの Claude や Codex を使うパターンと、私が SSH か Web でアクセスして使うシーン。これを整理しないとぐちゃぐちゃになる

さらに 2 つ足されました。手元の CLI で動いている AI が SSH で箱を操作するパターンと、手元の AI が箱の AI へ依頼するパターン。合計 5 通りです。

```
 軸1  箱の AI が自分でソースを読んで作業する（夜・人はいない）
 軸2  箱の AI が、箱の別の AI に仕事を投げる（夜・人はいない）
      ★ このあと認証を引き上げたので、設計対象から外れます
 軸3  自分が SSH か Web 画面で箱に入って操作する（昼・人がいる）
 軸4  手元の CLI で動いている AI が、SSH で箱を操作する（昼）
 軸5  手元の AI が、箱の AI に仕事を依頼する（昼）
```

軸4 は、この検討そのものです。手元の AI が SSH で箱を調べて回っていました。そして**この軸は共有の仕組みを 1 つも使っていません**。

数え上げるときりがないので、設計に効く違いだけで畳みました。共有の仕組みにとって意味があるのは 2 点だけです。**そのとき手元のマシンが動いているか**と、**GitHub へ書くときの名義は何か**。

```
   軸1 ┐
   軸2 ┘ → ケースB「手元が止まっている・人もいない」
            GitHub の共有リポだけが頼り。古さの札と機械の門番が要る。

   軸3 ┐
   軸4 ├→ ケースA「手元が動いている・人が見ている」
   軸5 ┘  資料は SSH でも直接コピーでも渡せる。共有リポは無くても回る。
```

ここから出る結論が効きました。**共有の仕組みが本当に必要なのは、ケースB の時間帯だけ**です。昼の 3 軸は、いま既にある手段で足りています。作るものの範囲は、思っていたより狭くてよかった。

一方で、昼夜を問わず効いてしまう問題が 1 つありました。**箱の中から GitHub へ送ると、人が操作していても手元の AI が操作していても、名義はすべて bot アカウントになります。** GitHub 側からは区別できません。

だから「bot が読むための区画に触れたら弾く」という守り方をそのまま入れると、昼に自分が箱で作業したときも弾かれます。読む場所と書く場所をリポジトリごと分けたのは、これも理由のひとつです。

## 9 つのやり方を並べて比べた

ここから設計の話になります。「もっと AI に任せたいが、開発ローカルの全権限を渡すのは怖い」という要件で、取りうる手を全部並べました。

**案1 何もしない。** 今のまま使い続ける。作業は止まらないが、事故が起きたときの範囲が「全部」になる。

**案2 都度確認に変える。** 「常に許可」をやめて操作ごとに確認する。設定 1 つで今日から変えられるが、確認の回数が多いと内容を読まずに押すようになる。そうなると境界として機能しない。触れる範囲自体も狭まっていない。

**案3 ローカル実行を切って、箱だけで使う。** 開発マシンへの経路が無くなる。いちばん単純で強い。手元のファイルを直接触ってもらう使い方はできなくなる。

**案4 制限ユーザーを作り、ACL で見える範囲を絞る。** AI 用のログインを 1 つ作って、渡してよいフォルダしか見えないようにする。境界が OS の権限で守られるので、約束ではなく仕組みになる。ただし前述のとおり、開発ツリーの ACL 是正がセットで要る。

**案5 Windows Sandbox（使い捨ての仮想環境）。** 分離は強いが、閉じるたびにサインインが消える。継続的にログインして使う道具とは相性が悪い。

**案6 AI 専用の仮想マシンを立てる。** 分離が最強で状態も残る。

**案7 見せる場所だけを作る。** 渡したいものだけを置くフォルダを 1 つ作り、そこ以外は見せない。案4 と組み合わせやすい。

**案8 普段は切っておき、必要なときだけ入れる。** 今日から始められるが、戻し忘れる。人の運用に依存する対策は、忙しいときに必ず破れる。

**案9 開発マシンの中に WSL を立てて、そこで作業させる。** 運用がいちばん軽い。

![9 案を、分離の強さで並べる](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-03_ai-key-boundary_qiita_fig6.png)

分離の強さだけで並べると、上から専用 VM、Windows Sandbox、箱だけで使う、の順になります。この並びを見ているかぎり、答えは案6 です。ここに落とし穴がありました。

## 案6 の評価が、実測で 2 回ひっくり返った

案6（専用 VM）は、最初「いちばん重い」と評価していました。今すでに仮想マシンの中にいるので、その中にもう 1 台作ることになる、と考えたからです。

そこに指摘が入ります。

> ホストからこれ自体がゲスト Hyper-V なんだが

そのとおりでした。ゲストから見えるホスト情報を取ります。

```powershell
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Virtual Machine\Guest\Parameters' |
  Select-Object HostName, PhysicalHostName, VirtualMachineName
```

ホスト名と仮想マシン名が返りました。新しい VM も**同じホストから兄弟として**立てられるので、入れ子にはなりません。さらに「ライセンスが余って遊ばせている」とのことで、私が挙げた「重い」根拠が 2 つとも消えました。

しかも案6 には副次効果があります。**自分の管理下の箱になる**ので、借り物の箱では置けなかった高価値の認証を置く判断ができます。ホストが動いている限り常時稼働なので、「開発マシンを落とすと夜間に資料が見えない」という別の問題の受け皿にもなります。

さらに「ホストの SSD を共有点にする」という案も出ました。案6 の最大の弱点だった「開発ファイルが別マシンになる」を潰せます。到達性を確かめました。

```powershell
Test-NetConnection -ComputerName <ホスト名> -Port 445 -InformationLevel Quiet
# → False（名前が Tailscale の古いエントリを掴んでいた）
```

名前解決が、38 日前から offline のノードを指していました。LAN 側を探します。

```powershell
Get-NetIPConfiguration | Where-Object { $_.NetAdapter.Status -eq 'Up' }
# 開発ゲストは LAN 上のアドレスでブリッジ接続。ゲートウェイはルーター

# ARP に見える近傍を逆引きして SMB を叩く
Resolve-DnsName <候補IP> -Type PTR
Test-NetConnection -ComputerName <候補IP> -Port 445 -InformationLevel Quiet
```

ホストは同じ LAN 上にいて、SMB の 445 も開いていました。名前で指すと Tailscale の古いエントリへ行くので、IP なら通ります。ただし共有一覧は「アクセスが拒否されました」で、E ドライブはまだ公開されていませんでした。

つまり案としては成立しますが、共有した瞬間にそこが境界の穴になりえます。丸ごと共有すると、マシンを分けた意味がその共有点から漏れます。E の中の 1 フォルダだけを公開し、ボット側は読み取り専用にする形が要ります。

## 案9（WSL）は安定するが、この問題には効かない

次に出たのがこの案でした。

> ホストに Hyper-V 2 つじゃなくて、ゲストの中に WSL 立てた方が安定する気がしてんだが

安定性の観点は当たっています。OS が増えないので更新もバックアップもライセンスも要らず、起動は数秒で、メモリも固定で取られません。既に入っていて動いています。

ただし実測すると、今回の懸念には効きませんでした。

```bash
$ wsl -e sh -c "ls -d /mnt/*; ls /mnt/d | head -5; which cmd.exe"
/mnt/c
/mnt/d
$RECYCLE.BIN
System Volume Information
Users
dev
/mnt/c/WINDOWS/system32/cmd.exe
```

```
$ cat /etc/wsl.conf
[boot]
systemd=false
[user]
default=<user>
```

`automount` も `interop` も入ったままで、WSL の中から開発ツリーもユーザーディレクトリも見え、Windows の実行ファイルも呼べます。制限は何も入っていません。境界として使うなら両方を切るのが前提です。

そして決定的なのが 2 点。**Grok Bot は Windows アプリなので WSL の中では動きません。** 立てても「このコンピューターで実行」が見る範囲は変わりません。そして **WSL はこの開発マシンの中にいるので、マシンを落とせば一緒に落ちます。** 夜間の受け皿にもなりません。

安定性の直感は正しくて、ただ解いている問題が違う、というのが正確なところです。

## いちばん効いた指摘

ここまで分離の強さと手間で比べていました。そこにこの指摘が入ります。

> ユーザーを増やす案もホストを増やす案も、私の環境からその Grok Bot のアプリをシームレスに使えないと意味ないよね？

評価軸が足りていませんでした。どれだけ安全でも、**アプリがふだんどおり使えなくなる案は採用されません。**

実測します。

```powershell
(Get-Acl 'C:\Program Files\Grok Bot').Access |
  Where-Object { $_.IdentityReference -match 'Users' }
# BUILTIN\Users  ReadAndExecute, Synchronize  Allow
```

インストール先は一般ユーザーに読み取りと実行を許可していて、設定は `%APPDATA%` 配下のユーザーごと、管理者昇格も要求していません。**別のローカルユーザーからでも起動でき、そのユーザー専用の設定を持てます。**

Windows の `runas` は同じセッションで別のトークンとしてプロセスを動かすので、GUI アプリのウィンドウは同じデスクトップに出ます。案4 なら操作感がほぼ変わりません。

一方、案6 は別マシンなのでリモートデスクトップ越しになります。ウィンドウの中にデスクトップがもう 1 枚入る形です。

この軸を入れたら順位が変わりました。

| 案 | 分離の強さ | ふだんの操作 | シームレスか |
|---|---|---|---|
| 案1 何もしない | なし | 今のまま | ◎ |
| 案2 都度確認 | 弱い | 操作のたびに確認 | △ |
| 案3 箱だけで使う | **強い** | 今のまま。手元だけ触らなくなる | **◎** |
| 案4 制限ユーザー＋ACL | 中〜強 | 別ユーザーで起動。同じ画面に出る | **○** |
| 案5 Windows Sandbox | 強い | 別画面＋毎回初期化 | × |
| 案6 専用 VM | **最強** | リモートデスクトップ越し | △ |
| 案7 見せる場所を作る | 弱〜中 | 今のまま。置きに行く | ◎ |
| 案8 必要なときだけ | 弱い | 使う前後で切り替え | ○ |
| 案9 WSL | 対象外 | **Grok Bot は動かない** | 対象外 |

第一候補は案4、次点が案3 になりました。案6 は分離では最強のままですが、日常操作の面で落ちます。ただし捨てるのではなく、**夜間の自律作業の受け皿として後から足す**、という位置づけに変えました。

![安全さだけで選ぶと、使われなくなる](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-03_ai-key-boundary_qiita_fig7.png)

縦を分離の強さ、横をふだんどおり使えるかにして置き直したのが上の図です。右上に残ったのは案4 と案3 だけで、分離では 1 位だった案6 は左上へ外れます。

## 検討シート（マスク版）

ここまでの 9 案と、残る 4 論点をまとめた検討シートを、固有名を伏せた形で置いてあります。単一の HTML ファイルで、案ごとにメリットとデメリットと採用ボタンがあり、選んだ内容を Markdown で出力できます。

https://ishizakahiroshi.com/articles/2026/2026-09-03_ai-local-permission-boundary/review-masked.html

実物はホスト名や組織名や実パスが入っているので、公開版はそれらを伏せています。構造と論点はそのままです。

## そして当日の朝、同じ問題を解くアナウンスが流れてきた

記事を書いている当日の朝 6 時 5 分に、こんなポストが流れていました。

> Cursor: Self-hosted machines
>
> What changed:
> - Tool execution stays entirely in your own network
> - Team pools are named queues that scale with demand and can hibernate idle machines to avoid keeping capacity warm unnecessarily.
> - Cloud agents can now run on infrastructure you already use: AWS Lambda, Coder, Cloudflare, Daytona, Modal, Namespace, Vercel, and E2B.
> - Self-hosted workers support computer use on Linux and Mac

Cursor Releases（@cursorreleases）
https://x.com/cursorreleases/status/2095256702023242200

1 番目の項目には続きがあって、コードベースもビルド出力も秘密も内部のマシンに残る、と書かれています。4 番目にも続きがあり、セルフホストのワーカーではエージェントがクリックや入力やスクリーンショットやブラウザ操作をできる、とあります。

1 行目がそのままこの記事のテーマでした。一日かけて考えていたことを、ツールを作っている側が製品仕様として書いていました。

公式ブログにはこうあります。

> no inbound ports, firewall changes, or VPN tunnels required

> your codebase, tool execution, and build artifacts never leave your environment

Katia Bazzi（Cursor）
https://cursor.com/blog/self-hosted-cloud-agents

仕組みとしては、worker が外向きの HTTPS でクラウドへ接続し、Cursor 側の agent harness が推論と計画を担当して、tool call だけを worker へ送って自分のマシンで実行する形です。インバウンドのポート開放は要りません。ドキュメントは以下です。

https://cursor.com/docs/cloud-agent/self-hosted-pool

面白いのは、**今回調べていた箱そのものが Cursor のサンドボックスだった**ことです。その提供元が「実行はあなたのネットワークに留められます」という選択肢を出してきた。今日の議論に対する回答が、同じ日に出ていたことになります。

## 認証をサンドボックスの外に置く、という設計はすでにあった

もうひとつ、条文を追っている途中で見つけたものがあります。

Anthropic の Claude Code on the web は、クラウドのサンドボックスでセッションを動かします。その分離の説明に、こうありました。

> **Credential protection**: in Anthropic-hosted environments, git credentials and signing keys stay outside the sandbox, and a proxy authenticates on the session's behalf with scoped credentials.

出典: https://code.claude.com/docs/en/claude-code-on-the-web

**git の資格情報と署名鍵はサンドボックスの外に置き、プロキシがスコープ付きの資格情報で代理認証する。**

今回の検討の途中で「自前の API ゲートウェイを立てて、サンドボックスには本物の鍵を置かない」という案が出ていました。それとまったく同じ形が、ベンダー側の設計として先に実装されていたわけです。

同じページには、セッションが Anthropic のインフラで動くか、組織の self-hosted 環境で動くかを選べることも書かれています。ネットワークアクセスは既定で制限され、無効にもできます。

そして OpenAI も、Codex にクラウド側の実行環境を持っています。しかも認証の扱いは、もう一歩踏み込んでいました。

> Runs in isolated OpenAI-managed containers, preventing access to your host system or unrelated data.

> uses a two-phase runtime model: setup runs before the agent phase and can access the network to install specified dependencies, then the agent phase runs offline by default unless you enable internet access for that environment.

> Secrets configured for cloud environments are available only during setup and are removed before the agent phase starts

出典: https://developers.openai.com/codex/agent-approvals-security

3 つ目が効きます。**クラウド環境に設定した秘密は、準備の段階でだけ使えて、エージェントが動き出す前に取り除かれる。** 依存パッケージのインストールに鍵が要ることはあっても、AI が作業している最中には鍵が残っていない、という設計です。

実行フェーズが既定でオフラインなのも同じ思想でしょう。準備でネットワークを使い、本番では閉じる。

タスクごとに専用の環境が割り当てられることも書かれています。

出典: https://developers.openai.com/codex/cloud

つまり、Cursor も Anthropic も OpenAI も、同じ方向を向いています。**エージェントはどこかで動く。資格情報はその「どこか」の外に置くか、動き出す前に取り除く。実行場所は選べるようにする。**

今日一日「箱に鍵を置いていいか」で悩んでいた問いに対して、3 社とも「そもそも置かない」で答えていました。

## 今後の開発フェーズは、たぶんローカルから離れる

ここからは推測です。

今日やっていたことを一言でまとめると、「自分のパソコンの中に AI を入れたときに、どこに壁を立てるか」でした。制限ユーザーを作る、ACL を直す、VM を立てる、WSL を切る。全部、**自分のマシンの内側を仕切る**話です。

でもたぶん、この方向は本流ではありません。

Devin は、各インスタンスが独立した仮想マシンの中で動きます。ターミナルもブラウザも開発環境もその中にあり、シェルを実行してテストを走らせて、自分の変更を検証してから報告してきます。

https://docs.devin.ai/get-started/devin-intro

https://cognition.ai/blog/what-we-learned-building-cloud-agents

Manus は、タスクごとに隔離されたクラウド VM を割り当てます。さらに Cloud Computer という永続的な VM もあって、こちらは利用者が操作していない間も動き続けます。

https://manus.im/blog/manus-cloud-computer

https://help.manus.im/en/articles/15392111-what-is-the-cloud-computer

Anthropic は Claude Code on the web で、ターミナルとクラウドの間をセッションごと行き来できるようにしています。クラウドへ送るフラグと、手元へ引き戻すフラグがあります。

https://code.claude.com/docs/en/claude-code-on-the-web

OpenAI も Codex にクラウド側を用意しています。タスクごとに専用の環境が割り当てられ、コンテナは OpenAI の管理下でホストシステムから隔離されます。

https://developers.openai.com/codex/cloud

https://developers.openai.com/codex/agent-approvals-security

そして Cursor は、そのクラウド側の実行場所を自分のインフラにできるようにしました。

並べてみると、向きが揃っています。**開発者が「どこで動いているか」を意識しなくてよくなる方向**です。

そうなると、今日一日かけて考えていた「自分のパソコンの中のどこに壁を立てるか」という問い自体が、数年後には古くなっているかもしれません。パソコンの中に AI を入れるから壁が要るのであって、そもそも中に入れなければ壁は要らない。

もちろん、それはそれで別の問題が出ます。今度は「その外側は誰のものか」という話になります。今日ずっと考えていた「借り物の箱に何を置くか」が、規模を変えて戻ってくるだけかもしれません。

だから結論は出ていません。まだ試行錯誤の途中です。

## 学びを、いちばん安い置き場所へ落とす

ここまでで小さな gotcha がいくつも出ました。全部をルール文書に書き足すと、読まれるコストが毎回かかります。

置き場所は「読まれるコストの安い順」に当てはめるのが良いと思っています。梯子はこうです。

1. 機械検査（CI で落ちる検査スクリプト）。誰も読まない。破ると落ちる
2. 触るファイルの先頭コメント。そのファイルを開いた人だけが読む
3. skill。起動語で呼ばれたときだけ読まれる
4. guide。トリガー時に読まれる
5. 台帳。提案や着手の前に引く
6. 全体ルール文書。**全セッションで全文が読まれる**
7. memory。同上

いちばんやりがちな失敗は、**1 が使えるのに 6 へ書く**ことです。文章は破られても誰も気づきませんが、検査は破れません。

![学びの置き場所は、読まれるコストの安い順に当てはめる](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-03_ai-key-boundary_qiita_fig8.png)

梯子にすると、下ほど読む人が少なく、上ほど全セッションで全文が読まれます。今回の 3 つは、機械検査と skill と guide の 1 行に分かれました。

今回の学びを当てはめると、こうなりました。

**「サブスク認証は公式ツールにだけ置く」は機械検査にできます。** 各 CLI の設定ファイルを走査して、第三者ツールがベンダーの OAuth を持っていないかを判定する。実際に書きました。

```powershell
# 判定ルールは 1 行
#   各社のサブスク OAuth は、その会社の公式 CLI の設定ファイルにだけ存在してよい。
#   第三者ツールの設定に入っていたら異常とみなす。

$Targets = @(
  @{ Name='Claude Code'; Vendor='anthropic'; Official=$true;  Shape='claude';    File="$HOME\.claude\.credentials.json" }
  @{ Name='Codex CLI';   Vendor='openai';    Official=$true;  Shape='codex';     File="$HOME\.codex\auth.json" }
  @{ Name='grok CLI';    Vendor='xai';       Official=$true;  Shape='grok';      File="$HOME\.grok\auth.json" }
  @{ Name='gemini CLI';  Vendor='google';    Official=$true;  Shape='flatoauth'; File="$HOME\.gemini\oauth_creds.json" }
  @{ Name='OpenCode';    Vendor=$null;       Official=$false; Shape='providers'; File=$OpenCodeAuthPath }
)
# ※ 第三者ツールの設定ファイルの場所は、ツールごとに違うので変数へ切り出しています
```

判定は 3 分類にしました。公式 CLI の OAuth は OK。第三者ツールの設定にベンダー名の OAuth があれば FLAG。形が読めないものは UNKNOWN で、黙って OK にはしない。FLAG が 1 件でもあれば exit 1 にして、ゲートとして使えるようにしました。

実行するとこうなります。

```
ツール       provider    種別  判定 備考
------       --------    ----  ---- ----
Claude Code  anthropic   oauth OK   plan=max 期限=...
Codex CLI    openai      oauth OK   auth_mode=chatgpt
grok CLI     xai         oauth OK   エンドポイント別 entry 1 件
gemini CLI   google      oauth OK   期限=...
OpenCode     google      api   OK
OpenCode     opencode-go api   OK   ツール自身のサービス
```

実装で 1 つ気をつけた点があります。**値を伏せる方式で書かない**ことです。「pass や token を含むキーは伏せる」という書き方だと、そのリストに載っていないキーが無警告で素通りします。過去にそれで実際に値を出したことがあるので、**出してよいキーを名指しする方式**にしました。この検査が出力するのは、ツール名と provider 名と type と判定と日時だけです。

合成データでも確かめました。架空の `{"openai":{"type":"oauth","access":"xxxxxxxx"}}` を作って渡すと FLAG 1 件で exit 1 になり、同じファイルを公式扱いにすると FLAG が消えて exit 0 になります。**実データを複製してテストに使わない**のは、こういう検査ではとくに大事だと思っています。

一方、「借り物の実行環境に高価値の認証を置かない」は機械検査にできません。対象の環境が毎回違うからです。これは skill にしました。認証方式の判定、環境の性質の実測、**永続化機構の有無の確認**、規約の該当条文、判定と撤去手順、という 5 ステップです。

そして観測の取り違え 4 件は、既存のガイドに 1 項目足すだけにしました。新しい節は作りません。**段を決めた時点で判断を終えると、同じ段の中で節が増えて太る**ので、既存項目の隣に置けないかを先に探します。今回は「数字を訂正するときは、何を測った数字かまで確かめる」という項目が既にあったので、その家族として書けました。

## 委託しようとしたら、信頼確認のモーダルで止まった

作った検査スクリプトと skill の実装を、別の AI CLI へ委託しようとしました。委託用の計画書を作って起動したところ、3 回とも同じ壁で止まりました。

```
[ORCHESTRATION-ERROR] child TUI is waiting on a modal
(Do you trust the contents of this directory)
and never reached its composer; the initial prompt was NOT delivered
```

Codex のディレクトリ信頼確認モーダルです。プロンプトが届く前に止まっています。

作業ディレクトリを信頼済みの場所へ移して再試行しましたが、また同じでした。原因を調べたところ、子プロセスを起動しているデーモンが、**指揮側セッションに登録された作業ディレクトリ**で起動していました。私がシェルの中で `cd` しても、それは自分のプロセスの話で、デーモンには関係ありません。

3 回失敗したところで止めました。同じ操作を 3 回打つ前に観測するのは、この日ずっと守ろうとしていたことです。

結局、実装は自分でやりました。委託の計画書は完了条件がそのまま受入表として使えたので、無駄にはなっていません。検査スクリプトは素の状態で FLAG 0 件・exit 0、合成データで FLAG 1 件・exit 1 を確認し、skill 側はハウスの検査スクリプトで ERROR 0 の exit 0 を確認しました。

自動化を足すと、**壊れる部品が 1 つ増える**。今回は起動そのものが詰まりました。委託の中身ではなく、委託の入口で止まったわけです。

![壁を立てる話をしていたら、入口で止まった](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-03_ai-key-boundary_qiita_illustration-stopped-at-door.png)

壁を立てに行ったのに、そもそも入口の柵を越えられなかった、という絵です。委託の中身ではなく委託の入口で止まる、というのはこういう形をしています。

## 実測してみて、残ったもの

技術的な結論より、手癖として残ったもののほうが大きかった気がします。

**書いた時点の前提は、書いた時点のものでしかない。** 計画書に「常時稼働」「24 時間動く」と書いてあっても、それは書いた人がそう思っていたという記録です。この日は 3 つの前提が実測で崩れました。

**出力が返ってきたことと、それが調べたい対象を測っていることは別。** 4 回外しました。壊れた観測は例外を出さず、それらしい値を返します。

**規模を測る前に、範囲を決めない。** 中身の内訳を見る前に「全部そのまま共有する」と決めていました。実物が分かってから決めるべきでした。

**安全さだけで選ぶと、使われなくなる。** これがいちばん効きました。分離の強さで並べたら案6 が 1 位でしたが、日常の操作が成立しないので採用されません。使われない対策は守ってくれません。

**いちばん緩いツールが、そのマシンの実際の境界になる。** 1 つ締めても隣が開いていれば意味がない。

## おわりに

一日じゅう「借りている箱にどこまで預けるか」を考えていて、最後に「自分のパソコン側には壁が 1 本も無い」と気づいたのが、いちばん間抜けで、いちばん収穫でした。

問いの立て方が最初から少しずれていた。そういうことは、手を動かして測ってみるまで分かりません。

そして冒頭に戻ると、始まりは「怖い」ではなく「**もっと使い倒したい**」でした。使い倒そうとしなければ、鍵をあちこちに置く必要もなかった。**便利にしようとする動きが、そのまま境界を溶かす動きでもある。** 今日の 9 案は全部、この綱引きの上に乗っていました。

結論はまだ出ていません。案4 で試してみて、ふだんどおり使えなければ案3 へ落とす。夜の作業が本当に要るなら案6 を足す。そのくらいの温度で進めます。

小さく。前提を疑って、1 つずつ測るのを、これからも続けていきます。

## あわせて読みたい

- 前回の記事（Qiita 版）: AI と AI の間で手紙を運ぶのをやめたら、本当の犯人が出てきた
  https://qiita.com/ishizakahiroshi/items/6a9edc768efea629bf07
- 前回の記事（note 版・一般向け）
  https://note.com/ishizakahiroshi/n/n987039361c75
- この記事の一般向け版も note に書いています（同日公開予定）
  https://note.com/ishizakahiroshi

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-09-03_ai-local-permission-boundary/

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
