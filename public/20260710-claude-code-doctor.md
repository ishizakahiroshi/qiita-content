---
title: "30 個のスキルを積んだ Claude Code に /doctor をかけて、棚卸ししてみた"
tags:
  - ClaudeCode
  - AI
  - CLI
  - Skill
  - 生産性
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![30個のスキル棚を /doctor で棚卸しするヒーロー画像](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-10_claude-code-doctor_hero.png)

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-10_claude-code-doctor_infographic.png)

/doctor という診断コマンドが `/help` 経由でひっそり増えていたので試しました。結論から言うと、私の環境ではほぼ「掃除するところがない」判定でした。ただ、走らせて眺めていたら「スキルの棚が今どう使われているか」がデータで見えて、そちらのほうが収穫でした。今日はその素振り記録と、30 個ほど積み上げた自作スキルをどう運用しているかの tips を書きます。

## TL;DR

- Claude Code 2.1.206 の /doctor は「未使用スキル・重い CLAUDE.md・冗長な設定・遅いフック」まで一気に棚卸ししてくれる健康診断コマンド
- 走らせた結果、実質「stray な計画 md が 1 枚あったので workshop 配下に移した」だけで完走。逆に「使用回数ゼロなのに残しているスキル」の存在価値をデータで再確認できた
- 30 スキルを回している側の tips: 起動語を「日本語で自然に口に出せる短句」で複数用意しておくと、AI が勝手にディスパッチしてくれる時間が伸びる

## /doctor は何をチェックしてくれるか

/doctor は単なる `claude doctor` の環境診断より一段広い範囲を見ています。実際は次の 10 チェックが走ります（0〜9 で数える流儀）。

- インストールと設定ファイルの健全性（native / npm / brew の重複、settings.json のパース、agents 定義の重複）
- 使われていないスキル・MCP サーバ・プラグイン
- ローカル CLAUDE.md と checked-in CLAUDE.md の重複と矛盾
- checked-in CLAUDE.md から「コードを読めば分かること」を削るべきか
- 常時ロードの CLAUDE.md を遅延ロード（スキル化・サブディレクトリ化）できるか
- 遅いフック
- コンテキストを食っている拡張の内訳
- 本体のバージョンが最新か
- 既定パーミッションモードを auto にしていいか
- よく拒否されている read-only コマンドを事前承認していいか

「読み取り専用でまず全部走らせて、まとめて提案 → 最大 2 回だけ確認 → 適用」という進行が固定されていて、勝手にファイルをいじらないのが良いです。

## 実際に走らせた結果（マスクあり）

私の環境（Windows / PowerShell 主・Bash 併用）で走らせた結果はこんな感じでした。

- **インストール**: `~/.local/bin/claude` (native) 一本。`installMethod: native` と一致、npm-local の残骸なし
- **バージョン**: 2.1.206（最新と一致）。`autoUpdates: false` は自分の意思なのでそのまま
- **既定モード**: すでに `permissions.defaultMode: "auto"` 設定済み。提案なし
- **拒否パターン**: 直近 50 セッションで拒否記録は約 10 件、すべて単発。事前承認に値する繰り返しなし
- **未使用**: 有効化中のプラグイン 1 個が 0 使用だったが LSP 系（受動的に働くやつ）なので keep 判定
- **自作スキル 30 個**: 6 個が使用回数ゼロだったが、いずれも「特定プラットフォーム配布用の狭用途」で、トリガー語での on-demand 発火が前提。**削らない** 判定

唯一「あ、これは掃除だな」だったのが `~/.claude/skills/plan_memory-broom-skill.md` という stray な計画 md が、スキル置き場のルート直下に置きっぱなしになっていたこと。スキルローダーは無視するだけなので実害はないんですが、置き場所として不適切。workshop の `plans/` 配下に移動して完走しました。

## 自作スキル 30 個の内訳（一部マスク）

これがこの記事の本題です。「30 スキルもあると管理どうしてるの」とたまに聞かれるので、種類別に晒します。名前はぼかしています。

### よく使う高頻度スキル（月に何十回）

- **記事下書きスキル**（write-*）。note / Qiita / Zenn 用の下書き生成。ヒーロー画像パイプラインとフッターの定型化まで込み。使用回数 57
- **HTML 資料生成スキル**（design-*）。インシデント報告・UI モック・LP・スライドを 1 ファイル self-contained な HTML で作る。使用回数 42
- **スキル早見表ディスパッチャ**（S）。`/S <自然言語>` で 30 個の中から 1 個をピンポイント起動する薄いラッパー。使用回数 101（実質的にスキル群への玄関）
- **協議シート生成**（review-sheet 系）。意思決定用の HTML5 チェックシート
- **セッション振り返り**（session-recap）。「今日の学び／次にやること」を md に書き出す
- **監査プロンプト集**（audit-prompts）。DB や WebApp を別 AI CLI（codex 等）で監査する時の貼り付け用プロンプトを 1 発で用意

### 中頻度（週〜隔週）

- **リリースパイプライン適用**（release）。タグ駆動で npm / crates / GitHub Release まで通す。使用回数 11
- **プロジェクト初期化**（project-init）。CLAUDE.md / AGENTS.md / .gitignore / secrets-scan を新規リポに 4 層防御込みで配線
- **LICENSE 生成**（license-init）。個人 OSS の LICENSE を家の正典フォーマットで生成
- **リポ整合監査**（repo-consistency）。README / manifest / アプリ内表記のズレを検出
- **GitHub Actions セットアップ**（github-actions 系）
- **アイコン生成**（make-icon）。SVG 手書き→favicon 一式を焼く
- **DB マイグレーション適用**（supabase-migrate）
- **地図系プロジェクトのデプロイ**（domain-specific-deploy）

### 低頻度・0 使用だが持っておく（狭用途配布スキル）

- Chrome ウェブストア申請
- Firefox AMO 申請
- Rust crates.io 予約・公開
- Python PyPI 公開
- 分類列の master 化 refactor

これらは「使う頻度は年 1〜2 回だが、その一回で 3 時間節約できる」タイプ。使用回数がゼロでも消さないのは、**トリガー語（「Chrome 拡張のリリース」「crates 予約」など）で AI が勝手に到達してくれることに価値がある** からです。人間が「どのスキルだっけ」と思い出す必要がない状態が本体の効用。

![30スキルの稼働ピラミッド](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-10_claude-code-doctor_fig1.png)

上の図のように、稼働はピラミッドになっていました。頂点の 5〜6 個が高頻度で 8 割を持ち、底辺の 0 使用スキルは「年 1〜2 回の保険」です。回数だけで切ると、いちばん効く保険から先に落ちます。

## 効率的な運用 tips

30 個回してみて実感している運用のコツ。

### 1. 起動語は「日本語で自然に口に出せる短句」を 3 個以上用意する

英字コマンド名で呼ばせるのは負け筋。「Chrome 拡張 公開」「ウェブストア 申請」「webstore publish」のように、**その状況で人間が自然に口走る言い方** を description に埋めておくと、AI が SKILLS.md から自力でディスパッチしてくれます。私の場合、`/S` に「これ HTML で説明して」と投げれば `design-html` が起動する。呼び出し方を覚えなくていいのが最強。

### 2. スキル本体は薄く、参照 md を厚くする

SKILL.md 本体に手順を全部書くと、AI が読み込む時のコンテキスト食いが増えます。SKILL.md には「起動判定 → 判断ロジック → 参照先」だけ書いて、実手順は `references/*.md` に分割。必要な時だけ Read してもらう構造にしておくと、他のスキルを圧迫しません。

### 3. 「4 層防御」で秘匿情報を漏らさない

公開スキル（記事生成・画像生成・README 生成）は毎回 secret-leak-guardrails を Read してから走らせる作法。①スキル本体で AI に自問させる ②pre-commit hook で機械チェック ③GitHub Actions で最終チェック ④リリーススキルの前提チェックで再確認。書く瞬間の自問が一次防御で、後段の grep は保険という考え方。

### 4. スキル置き場は 1 箇所に集約、各 CLI からはジャンクションで見せる

Claude Code / Codex / Cursor / opencode で同じスキル棚を共有するために、正本を 1 箇所（`C:\dev\workshop\skills\`）に置いて、各 CLI の期待するパスへディレクトリジャンクションを張っています。スキルを 1 か所直せば全 CLI に反映される。管理コストが 4 分の 1 になります。

### 5. /doctor は「掃除」より「棚卸し」ツールとして使う

診断結果で削除するものが出なくても、`skillUsage` を眺めると自分のワークフローがデータで見えます。「意外とこのスキル使ってないな」「このスキル、月 3 回叩いてるじゃん」という気付きのほうが、掃除より価値があります。四半期に 1 回くらいのペースで走らせるつもりです。

![掃除期待と棚卸し収穫の対比](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-10_claude-code-doctor_fig2.png)

上の図は、走らせる前の期待と実際の収穫の対比です。削除リストはほぼ空でも、0 使用スキルを keep する根拠と、高頻度スキルの偏りが数字で見えました。掃除道具というより棚卸し表、という感覚です。

## 小さな決意

30 個は多いようで、実際は 5〜6 個の高頻度スキルが 8 割の稼働を持っていて、残りは「その一回のための保険」でした。/doctor がそれをデータで裏付けてくれたのが良かった。掃除は少なかったけど、棚卸しは十分できた。棚を大きくしすぎず、でも保険は捨てず、次の 3 か月もこのペースで積んでいこうと思います。

紹介した /doctor コマンドは `/help` から一覧に出ます。走らせて眺めるだけでも 10 分。おすすめです。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
