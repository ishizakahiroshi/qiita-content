---
title: 自作 CLI が『安全策』で置いた backup ディレクトリで、非公開の md を公開リポに漏らしていた話
tags:
  - Python
  - PyPI
  - gitignore
  - OSS
  - poka-yoke
private: false
updated_at: '2026-07-18T20:50:10+09:00'
id: 0f5b3e3d6f5406f56540
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![docsweep v0.3.1 の反省と学びのヒーロー画像。backup が漏らしていた話](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-18_docsweep-backup-leak_hero.png)

![安全策による漏洩と多層防御のインフォグラフィック](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-18_docsweep-backup-leak_infographic.png)

自作の CLI ツールを PyPI に公開して 2 週間。ふと自分の公開リポを見に行ったら、その CLI が私の非公開 md を勝手に 150 個ちかく origin/main に push していました。しかも push していたのは、ツール自身が「安全のため」に作った backup ディレクトリでした。

「安全策」のつもりで置いた機構が、逆に情報漏洩の入口になっていた話です。反省と、どう根っこを潰したかと、そこから学んだことを書いておきます。

## `docsweep` はこんなツール（宣伝ちょっとだけ）

自作で [docsweep](https://github.com/ishizakahiroshi/docsweep) という CLI を作っています。AI が量産する plan / bugfix の Markdown を、腐らせず自動で片付けるクロスプラットフォーム CLI。です。

同じ悩みを持っている方は、下記で入ります。

```bash
pip install docsweep
```

リポジトリはこちらです（Star をいただけると励みになります）: <https://github.com/ishizakahiroshi/docsweep>

そして、今回の話はまさにこの `docsweep` そのものが起こした事故です。自分のツールで自分がやらかしました。

## 何が起きたか

私は `docs/local/*.md` を「家宣言の非公開」として運用しています。作業ログ、脆弱性の一次調査メモ、顧客名を書いた plan、そういうものが入っています。`.gitignore` に `docs/local/` を書いて、公開リポには絶対に載せない前提です。

ところが、複数の公開リポで `git ls-files docs/local` を叩いたら、こう出ました。

- `many-ai-cli`: 123 file が tracked
- `offline-md-editor-viewer`: 8 file が tracked
- `PlainSheet`: 18 file が tracked
- `ShotTTL`: 1 file が tracked

いや、そんなはずない。`.gitignore` で `docs/local/` を除外しているのに、なぜ？

`git log --oneline docs/local` で追ったら、犯人は綺麗に一種類でした。全部、`.docsweep/backup/<元 md 名>.<タイムスタンプ>.md` というパスから、直接 `docs/local/*.md` の内容が入った md が生えていました。

## 「安全策」だったはずの backup 機構

`docsweep` は、対象の md を書き換える前に、`.docsweep/backup/<name>.<epoch>.md` に **書き換え前の全文コピー** を残していました。30 日で自動掃除する簡易なもので、私が「安全のため」に入れておいた機構です。

流れはこうです。

1. `docsweep sweep` が `docs/local/plan_foo.md` の H1 を `[完了]` に更新
2. 更新前に `.docsweep/backup/plan_foo.1721312345.md` として全文コピーを保存
3. ユーザーが「なんかまとめて `git add .` するか」で `.docsweep/backup/*.md` まで stage
4. `git commit` して `git push`。公開リポの origin/main に **元の docs/local と同じ内容の md** が到達

つまり、`docs/local/` を `.gitignore` に書いていても、`.docsweep/backup/` を書き忘れていた瞬間、そこ経由で「元の md の中身」が公開リポに送られる構造でした。

![docsweep の backup 機構が公開リポに md を漏らしていた 3 ステップの図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-18_docsweep-backup-leak_fig1.png)

図のとおりです。私が守っていたのはあくまで `docs/local/` パスであって、`docsweep` が **別の場所に作った複製** はガードの外にありました。

## 最初は「`.gitignore` の 1 行を足せばいい」と思った

素直に考えれば、こう直せば済みます。

- `docsweep` 側で `.docsweep/backup/` に `.gitignore` を自動生成
- 公式の README で `.docsweep/` を `.gitignore` に足すよう案内
- 家の `project-init` テンプレ（新規リポ用の共通 `.gitignore`）に `.docsweep/` を追記

これで **これから作る新規リポ** は守れます。

でも、この方向を書き始めた瞬間に手が止まりました。守れないケースが 2 種類あります。

- **既に clone 済みのユーザー**: 既存プロジェクトは共通 `.gitignore` テンプレが自動では入らない。git は tracked file を後から `.gitignore` に足しても勝手には外さない。既に漏れた分の掃除も別途必要
- **私自身の運用**: 上記の対応をしていない状態で、既に 4 リポの origin/main まで到達してしまっている

「未来の repo だけを守る対策」を書いていることに気づきました。過去にすでに漏れた分と、対策を知らない既存ユーザーは救えません。

## 「そもそも backup 要る？」と、実装を数えた

`.gitignore` を足す前に、一回だけ立ち止まりました。

そもそも、`docsweep` が backup していた書き込みは、実際にはどんな種類だったか。手元のツールで、直近 1 か月分の書き込みイベントを分類してみました。ざっくり内訳です。

- 99% は `update_line` 経由の 1 行差分。H1 ステータスラベルの `[計画]` → `[完了]` 化、frontmatter の `due:` の日付更新など
- 残り 1% は `move` / `delete`。archive/ への移送で、この場合は Git 側で mv として履歴が残る

そして、記事執筆時点までに `docsweep` の backup ディレクトリから「実際に何かを復元した」回数は 0 回でした。1 行差分の書き戻しなら、`git checkout HEAD -- <file>` で足ります。それすら滅多にやらない。

つまり、backup が守っていたのは「万が一の未来の事故」だけで、その未来はほぼ起きない、というのが実測でした。

一方、backup が生んだ実害は 150 file の漏洩、こっちは今すでに起きている。

比較すると、答えははっきりしました。**backup 機構は撤去する**。

## 対策: backup 機構ごと撤去して、経路を消した

`v0.3.1` として次の変更を出しました。

- `atomic.backup()` 関数を削除
- `backup_dir_for()` / `_ensure_gitignored()` / `_cleanup_backups()` を削除
- `take_backup` kwarg を廃止
- `BACKUP_DIR_NAME` / `BACKUP_RETENTION_SECONDS` / `GITIGNORE_MARK` / `GITIGNORE_RULE` の定数を削除
- ドキュメントから `.docsweep/backup/` の言及を全部消す

削除したのは全て **内部ヘルパ** で、CLI サブコマンド と MCP tool の外向き API は無変更です。だから SemVer では **patch** で出しました（`v0.3.1`）。「破壊的変更に見えるが、公開 API は無変更」というときの SemVer 判定は、公開 API を単一の物差しにするのが個人 OSS では扱いやすいです。

同時に、家の `project-init` テンプレの共通 `.gitignore` に `.docsweep/` を 1 行追加しました。これで新規リポでは同じ罠が再生産されません（既存 clone は救えないので README で告知）。

既存ユーザーの後始末はシンプルです。

- 各プロジェクトの `.docsweep/backup/` ディレクトリを手動で削除
- 万が一 tracked なら `git rm -r --cached .docsweep/backup/` してから `.gitignore` に `.docsweep/` を追記して commit

`state.json`（docsweep の状態ファイル）は温存で問題ありません。`v0.4` でも使う正規データです。

## 学び

同じ形で他人にも刺さりそうな学びを 4 つに絞ります。

- **「安全のため」に足す機構が、しばしば新しい事故経路になる**。守るために置いたはずのバックアップ、キャッシュ、一時ディレクトリは、そこ経由で本体の情報が別の場所へ複製されている。移動先が公開圏内に入っていないかは、追加時に一回考える価値があります
- **デフォルト `.gitignore` に依存する設計は「未来のリポ」しか守らない**。既存 clone は共通テンプレを後から自動では受けない。今日足したルールで、明日 clone した人は救えるが、昨日 clone した人と、あなた自身は救えない
- **`.gitignore` は防御の「1 層目」でしかない**。ゲートを 4 層で考えるようになりました。層 1: AI ルール（書く瞬間の自問）、層 2: pre-commit hook（commit の水際）、層 3: CI 上の secrets-scan（push の水際）、層 4: リリースのゲートチェック（公開前の最終砦）。今回抜けていたのは層 2 と層 3 でした
- **SemVer の major/minor/patch は、外向き API の変化を単一の物差しにする**。「見た目破壊的」でも、CLI コマンドと引数と MCP tool が変わっていなければ patch にできます。逆に、内部を全然触っていなくても、`--foo` フラグの意味を変えたら minor です

## docsweep はこんなときに刺さります

- `plan_*.md` / `bugfix_*.md` を AI に書かせているけど、いつまでも `[完了]` のまま docs 直下に残っている人
- 複数プロジェクトで作業ログの陳腐化が気になっている人
- `docs/local/` を非公開ポリシーで運用しつつ、AI 生成 md の量に困っている人
- 今回の `v0.3.1` から、書き込み前 backup を作らなくなり、プロジェクト空間内に **何も余計なコピー** を残しません（`state.json` だけです）

いずれかに心当たりがあれば、`pip install docsweep` で 1 分で試せます。設定ゼロで動きます。

- リポジトリ（Issue / PR 歓迎）: <https://github.com/ishizakahiroshi/docsweep>
- PyPI: <https://pypi.org/project/docsweep/>

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## おわりに

自分のツールが自分の秘密ファイルを漏らす、というのは正直かなりショックでした。それでも、`.gitignore` を足すだけで済ませずに、一度「そもそも backup 要るか」まで戻れたのは、我ながらよくやったと思っています。

「守るために足したもの」が「守りたかったものを漏らす経路」になっていないか。少し習慣にしていきます。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-07-18_docsweep-backup-leak/

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
