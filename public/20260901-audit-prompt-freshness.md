---
title: セキュリティ基準のpinは9日で腐る。監査プロンプト集を58体のAIエージェントで最新化した実録
tags:
  - AI
  - Security
  - プロンプトエンジニアリング
  - AIエージェント
  - 生成AI
private: false
updated_at: '2026-09-01T12:27:27+09:00'
id: 288dbf9aa52b8d9bafbe
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![ヒーロー画像](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_audit-prompt-freshness_hero.png)

自作の監査プロンプト集 [ai-audit-prompts](https://github.com/ishizakahiroshi/ai-audit-prompts) を「2026年の最新トレンドに追随しているか」で総点検し、AIエージェントのworkflowで更新しました。結論だけ先に書きます。

- 9日前に全項目確認したはずのpinned基準が、再確認したら1本古くなっていた（OpenSSF OSPS Baseline の新版が公開4日後の時点で未反映。https://baseline.openssf.org/ ）
- 9つの切り口の並列調査で修正案132件を生成し、**事実性と規約適合の2レンズで敵対検証**してから適用した。内訳は そのまま採用31 / 修正して採用91 / 棄却10
- 実在しない事故名を根拠にした修正案が1件、コマンド挙動の誤説明が1件、2レンズが別々に検出した。**片方のレンズだけなら通っていた**
- エージェントは通算58体、トークンは2回の実行あわせて約885万。途中で利用上限に当たり19体が連続停止したが、resumeで完走

この記事は、その手順と落とし穴の記録です。

## 対象の監査プロンプト集

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=ai-audit-prompts
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/ai-audit-prompts

AIに書かせたコードや作業を安全に取り込むための、貼り付け用の監査プロンプト集です。使い方はリポジトリを clone して、docs 配下の正典プロンプト全文を、使っているAIエージェントへ貼るだけです。特定のAI製品には依存させていません。インストールや設定ファイルもありません。

```text
docs/
  audit_app.md                 アプリ／ソースコード監査の正典
  audit_server.md              管理下サーバー診断の正典（完全read-only）
  audit_doc_vs_impl.md         資料と実装の突合の正典（完全非変更）
  README_invariants*.md        共通契約の正本（app / server）
  README_activation.md         「対象」で正典を選ぶrouting
```

前回の記事: [監査プロンプトをAIツール別に持つのをやめた。10本を対象別3本へ統合した設計](https://qiita.com/ishizakahiroshi/items/a7065348ad7ca46ca521)

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_audit-prompt-freshness_infographic.png)

## きっかけ: 「最新に対応してる?」に即答できなかった

プロンプト集には、監査の拠り所にするセキュリティ基準を名前・版・URLで固定（pin）した一覧があります。9日前に全項目を一次情報で確認済みでした。

それでも即答を避けて再確認したら、OpenSSF OSPS Baseline に新版が出ていました。こちらの確認5日後に公開されたものです。確認から9日、公開から4日で「最新」の看板が嘘になる。この速度を人力で追うのは無理だと判断して、更新そのものをAIのworkflowに任せることにしました。

## 全体の流れ

```text
1. Research   9つの切り口で並列調査（各エージェントは一次情報のURL取得が必須）
2. Critic     調査の抜けを探す完全性批評（不足領域を追加調査）
3. Gap        現行プロンプトと突き合わせて修正案132件を生成
4. Verify     修正案を6件ずつ22組に束ね、各組を2レンズで敵対検証
5. Apply      生き残った案を人間が適用し、機械検査（文書構造check・secrets scan）で締める
```

切り口は、pinned基準の現行版、AIコーディングエージェント設定への攻撃、ソフトウェア供給網、脆弱性情報源、Web/アプリ、サーバー堅牢化、資料と実装の突合、AI監査手法、規制と国内情報源、の9つです。

## 修正案は「そのまま適用できる形」で出させる

散文の提案は受け取りません。全エージェントにJSON Schemaを渡し、置換対象の現文と修正後の完成文を持つ構造で返させます。

```yaml
id: audit_app-17
file: docs/audit_app.md
change_type: modify        # add / modify / remove
current_text: "<置換対象の現文をそのまま>"
proposed_text: "<修正後の完成文>"
rationale: "<なぜ必要か>"
sources: ["https://<一次情報のURL>"]
priority: high             # high / medium / low
```

current_text を実文と完全一致で書かせるのがポイントです。一致しなければ適用時に必ず失敗するので、「それっぽいが当てられない提案」がこの時点で detect できます。

## 2レンズの敵対検証

修正案をそのまま採用すると、もっともらしい誤りが混ざります。そこで役割の違う2種類の検証エージェントに、それぞれ「反証」を試みさせました。

- **事実性レンズ**: 修正案が引く版数・日付・URL・事故名・CVEを、その場で再fetchして裏取りする。確認できなければ落とす
- **規約適合レンズ**: プロンプト集側の決まり（特定製品名を正典に入れない、日付や版を本文へ焼き込まない、read-only契約を弱めない、冗長な重複を作らない）との衝突を探す

判定は keep / modify / drop の3値で、modify には修正済みの完成文を返させます。実際に落ちたもの、直されたものの例です。

- 実在しない事故名が根拠に混ざっていた。検索で痕跡ゼロ。事実性レンズが drop
- あるコマンドのdry-run挙動の説明が一次資料と食い違っていた。事実性レンズが正しい表現に modify
- ベンダー中立であるべき正典に特定製品のモード名が紛れた。規約適合レンズが一般語へ modify
- 「2026-XX-XX時点で最新」を本文に焼き込む案。陳腐化するので規約適合レンズが日付の置き場を直した

132件の判定は keep 31 / modify 91 / drop 10 でした。7割は無修正では通っていません。

![修正案が2つのレンズを通る流れ](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_audit-prompt-freshness_fig1.png)

修正案1件ごとに2レンズが独立に反証し、どちらかが drop すれば採用しない、modify は完成文ごと差し替える、という流れです。

## 運用の学び

- **resumeは「失敗した分だけ」再実行ではない**。使ったworkflow基盤のキャッシュは先頭からの無変更prefixだけで、最初の失敗地点より後は成功済みでも再実行される。再実行された検証エージェントは編集後の作業ツリーを読むので、判定が「既に反映済みなので重複」へ変わる。今回はそれが重複提案8件の正しい棄却として働いたが、逆向きに壊れることもある
- **検証と適用を並走させるなら、検証対象のファイルと編集中のファイルを分ける**。分けないと上の現象が予測不能になる
- **検証フェーズのエージェント数は起動前に見積もる**。22組×2レンズ=44体が利用上限に届き、19体が連続停止した。resumeで完走できたが、budgetガードを入れておくべきだった
- **pinは確認日と更新手順をセットで持たせる**。今回から一覧に確認日を明記し、更新時の同期手順をリポジトリの運用ルールに残した

![深夜の机で小さなロボットの群れが文書に赤入れしている挿絵](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_audit-prompt-freshness_illustration-robots-redlining.png)

検証フェーズの実際の風景は、だいたいこの挿絵のとおりです。1件ずつの赤入れが夜通し続きました。

## 今回プロンプト集に増えた観点（抜粋）

2025〜2026年に顕在化した攻撃・実務の変化から、監査観点をいくつも足しました。抜粋です。

- リポジトリ内のAIコーディングエージェント設定（instruction file・hook・MCP定義・auto-approve設定）を、AI機能の有無に関係なく常時監査する。隠しUnicodeやHTMLコメント経由の命令注入、追跡されたAIセッションログの混入も見る
- 依存パッケージのinstall時ポリシー（lifecycle scriptの遮断、新版のcooldown）、publish経路がOIDCか長期トークンか、AI由来の実在しないパッケージ名（slopsquatting）の登録日照合
- サーバー診断に、ホスト上の資格情報でバックアップを消せるか、公開面に露出したAI推論サービスやMCPサーバー、self-hosted runnerの見覚えのない登録痕跡
- 却下にも証拠を要求する対称な evidence contract（「複数エージェントの合意」を証拠にしない）

全文はリポジトリの docs 配下にあります。

## おわりに

「最新に対応しているか」への正直な答えは、「今日の時点では対応している。ただし9日後は分からない」です。だから確認日を書き、調べる・直す・疑うをAIに任せられる形を整えました。決めるところだけが人間の仕事として残ります。

## ai-audit-prompts はこんなときに刺さります

- AIにコードを書かせているが、レビュー観点が場当たりになっている
- AIに監査を頼んだら勝手にコードを直された経験があり、read-onlyの契約が欲しい
- 監査報告が「概ね問題なし」で終わり、何を確認して何を確認していないのか分からない
- 自分の管理サーバーを、状態を一切変えずに点検させたい

心当たりがあれば、リポジトリを clone して docs のプロンプトを貼るだけで試せます。インストールや設定ファイルは不要です。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=ai-audit-prompts
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/ai-audit-prompts

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## あわせて読みたい

- [AIコード監査の「修正済み」が分からない。findingをC管理にして結果報告を作業ハブにした](https://qiita.com/ishizakahiroshi/items/364c1125fa15fc8b9c77)（監査結果の追跡設計の話で、今回の report 契約の土台です）
- [「監査して」と頼んだら、コードが書き換わっていた](https://ishizakahiroshi.com/articles/2026/2026-07-12_audit-report-only-default/)（既定を「調査のみ」へ変えた事故の話。今回も維持した安全境界の原点です）

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-09-01_audit-prompt-freshness/

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
