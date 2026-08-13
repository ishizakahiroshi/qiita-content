---
title: "OKF v0.2対応でstatusが衝突した。docsweepで文書状態と作業状態を分けた"
tags:
  - Python
  - Markdown
  - AI
  - ClaudeCode
  - CodexCLI
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

![OKF v0.2対応でstatusを二つの軸に分けた](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-09_okf-v0-2-status-split_hero.png)

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-09_okf-v0-2-status-split_infographic.png)

AI コーディングエージェントが残す `plan_*.md` や `bugfix_*.md` を、ほかのツールでも読める形に持ち出したい。そのために docsweep の OKF v0.2 対応を進めました。

実装自体は終わったのですが、途中でかなり本質的な衝突が出ました。OKF にも docsweep にも `status` があり、同じキーなのに意味が違ったのです。

結論から言うと、**文書のライフサイクルは `status`、作業の進み具合は `docsweep_state`** に分けました。さらに、OKF の版ごとの規則は Python に直書きせず、JSON profile から読む設計にしています。

:::note info
この記事で紹介する `okf-check` や profile 選択機能は、docsweep v0.4.0（2026-08-10 リリース）に入っています。執筆時点ではまだリリース前でしたが、現在は PyPI から入手できます。
:::

## docsweepというMarkdown作業記録の整理CLIを作っています

docsweep は、Claude Code や Codex などが作る `plan_*.md`、`bugfix_*.md`、`pending_*.md` を走査し、残作業の発見、状態更新、archive までを助ける Python CLI です。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=docsweep
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/docsweep

基本の使い方は以前の記事にまとめています。

前回の記事: [AIが残すplan・bugfixを自動で片付けるdocsweepの始め方](https://qiita.com/ishizakahiroshi/items/17bf7a02efcb8c4718ec)

今回追加した操作は、次の形で使えます。

```powershell
python -m docsweep okf-profiles
python -m docsweep export --okf --out snapshot.zip
python -m docsweep okf-check snapshot.zip --json
```

入力は普段の Markdown、処理は OKF Bundle への書き出しと検査、出力はほかのツールでも読める Markdown と JSON です。

## OKF v0.2は、Markdownを大げさな形式に変えない

OKF は Open Knowledge Format の略で、Markdown に YAML frontmatter を付け、知識をツール間で持ち運びやすくするための仕様です。

[OKF v0.2 の公式仕様](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md)はかなり小さく作られています。通常の concept 文書で常に必須なのは、空でない `type` です。未知の `type` や追加フィールドがあっても拒否せず、壊れたリンクや任意フィールドの欠落も文書全体を不適合にしない、という互換性重視の考え方になっています。

ここは docsweep と相性がよいところでした。docsweep も Markdown 本文を正本として扱い、専用データベースへ閉じ込めたくないからです。

一方で、OKF v0.2 には文書のライフサイクルを示す `status` があります。

```yaml
type: plan
status: draft
```

`draft`、`stable`、`deprecated` は、「この知識文書を今後どう扱うか」を示します。

docsweep が以前から使っていた `status` は別物でした。

```yaml
type: plan
status: in-progress
```

こちらは「作業が進行中」という意味です。同じキーへ両方を押し込むと、OKF から見れば `in-progress` はライフサイクル値ではなく、docsweep から見れば `draft` では仕事が終わったか分かりません。

## statusを二つの軸に分けた

新しく生成する文書は、次の形にしました。

```yaml
---
type: plan
status: draft
docsweep_state: planned
---
```

- `status`: OKF の文書ライフサイクル
- `docsweep_state`: docsweep の作業状態
- H1 の `[計画]` や `[完了]`: 人が一覧で見るための表示

![OKFの文書状態とdocsweepの作業状態を分離した図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-09_okf-v0-2-status-split_fig1.png)

たとえば作業が終われば `docsweep_state: done` になります。その成果を再利用できる安定した知識として残すなら、OKF 側は `status: stable` です。同じ「完了っぽさ」でも、片方は作業、片方は文書の寿命を表しています。

この分離により、`manual_*.md` や `reference_*.md` のような静的な知識文書も扱いやすくなりました。これらは OKF の `type` と `status` を持てますが、進行中の仕事ではないので `docsweep_state` や `due` は通常必要ありません。

## 既存のstatusは読めるままにした

すでに多くのリポジトリに、旧形式の文書があります。一斉書き換えを前提にすると、互換対応のために今ある作業記録を壊すことになります。

そこで読み取りは次の優先順位にしました。

1. `docsweep_state` があれば、それを作業状態として使う
2. 無ければ旧 `status: planned` や `status: done` を作業状態として読む
3. それも無ければ H1 やファイル名から判定する

書き出すときだけ、新形式へ正規化します。しかも原本は変更せず、OKF Bundle の中のコピーだけを整えます。

この「古い文書は読める、新しい文書は曖昧さを増やさない」という境界は、今回かなり大事にしたところです。

## 仕様の版が変わるたびにソースを直したくない

もう一つ考えたのが、OKF の版が上がるたびに docsweep の条件分岐を書き換え、毎回リリースしなければならないのか、という問題です。

そこで、版ごとの機械可読な規則を JSON profile に切り出しました。現在の同梱 profile は概ね次の内容です。

```json
{
  "spec_version": "0.2",
  "lifecycle_default": "draft",
  "lifecycle_values": ["draft", "stable", "deprecated"],
  "required_frontmatter": ["type"],
  "unknown_types": "allow",
  "unknown_fields": "preserve",
  "broken_links": "warning"
}
```

![版ごとの規則をコードからJSON profileへ分離したイメージ](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-09_okf-v0-2-status-split_illustration-1.png)

profile の読み込み元は三つあります。

- 通常利用: wheel に同梱した `0.2.json`
- 組織独自ルール: ローカルの JSON ファイル
- 先行検証: 明示指定した GitHub Raw URL

外部 URL は勝手には見に行きません。利用者が `--okf-profile` を指定したときだけ取得し、CI では SHA-256 も固定できます。取得に失敗したとき、別の版へ黙ってフォールバックもしません。

```powershell
python -m docsweep okf-check .\bundle `
  --okf-profile .\okf-profile.json

python -m docsweep okf-check .\bundle `
  --okf-profile https://raw.githubusercontent.com/example/example/<commit>/okf.json `
  --okf-profile-sha256 <sha256>
```

これなら、同じ構造で表現できる規則変更は profile の差し替えで試せます。もちろん、仕様変更が新しい解析処理を要求するならコードの更新とリリースは必要です。**JSON 化はリリースを完全になくす魔法ではなく、データで表せる変更と実装変更を分けるためのもの**です。

## exportしたBundleを自分で検査できるようにした

OKF 対応を名乗るなら、書き出した zip を自分で検査できないと片手落ちです。そこで `okf-check` も追加しました。

Bundle は次のような構造です。

```text
docsweep-okf-2026-08-09.zip
├─ index.md
├─ okf-manifest.json
├─ project-a/
│  └─ docs/local/
│     ├─ plan_feature.md
│     └─ bugfix_issue.md
└─ _archive/
   └─ project-a/archive/plan_old.md
```

`index.md` は OKF の予約 index、`okf-manifest.json` は docsweep 固有の補助情報です。manifest には使用した profile の版と SHA-256、収録ファイル、正規化の有無を記録します。

`okf-check` はディレクトリと zip の両方を読み取り専用で検査します。

```powershell
python -m docsweep okf-check .\snapshot.zip --json
```

処理の流れは単純です。

```text
入力: plan / bugfix / pending の Markdown
  ↓
処理: 原本を変えず、Bundle内コピーをOKF v0.2へ正規化
  ↓
検査: YAML、type、予約ファイル、リンク、profile整合
  ↓
出力: OKF Bundle zip + JSON検査結果
```

未知の `type` や追加フィールドは OKF の許容範囲を尊重します。必須構造が壊れている場合は error、壊れたリンクなどは warning として分けました。「知らないものを見つけたから全部 reject」はしません。

## 実装完了とリリース完了は分けて報告する

今回の変更では、profile 読み込み、状態分離、export、read-only 検査、旧形式互換をテストしました。最終確認は `691 passed` です。

ただし、これは実装の検証結果です。この記事を書いた 2026-08-09 の時点では、PyPI の公開もタグもまだでした。

AI に「対応できました」と言わせると、この二つは簡単に混ざります。記事にするときも、実装済みと配布済みを分けて書くことにしました。実際の配布はその翌日で、v0.4.0 として PyPI に出ています。

## docsweepはこんな人に刺さります

- AI が作った plan や bugfix の Markdown が各リポジトリに増え続けている
- Markdown を正本のまま、残作業と完了記録を整理したい
- 特定の AI ツールだけに閉じず、Claude Code、Codex などで同じ作業記録を使いたい
- 将来の移行や監査に備え、知識を標準寄りの Bundle で持ち出したい

今回の OKF v0.2 対応も、「docsweep を使い続けないと読めない形式」にしないための変更です。

- 紹介ページ: https://ishizakahiroshi.com/work.html?id=docsweep
- GitHub（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/docsweep

## あわせて読みたい

- [AIが残すplan・bugfixを自動で片付けるdocsweepの始め方](https://qiita.com/ishizakahiroshi/items/17bf7a02efcb8c4718ec): docsweep の基本操作と、作業記録を腐らせない運用の全体像です
- [自作 CLI が「安全策」で置いた backup ディレクトリで、非公開の md を公開リポに漏らしていた話](https://qiita.com/ishizakahiroshi/items/0f5b3e3d6f5406f56540): 書き出し機能で「原本を守る」だけでは足りなかった失敗談です

## おわりに

OKF v0.2 対応で一番大きかったのは、コマンドが増えたことではありません。

`status` という一見単純なキーに、文書の寿命と作業の進み具合を詰め込んでいたことに気づけたことでした。

意味の違う状態は分ける。版ごとの規則は profile に寄せる。古い文書は読めるままにする。外部 profile は明示したときだけ読む。

この四つを決めたことで、標準対応が「チェック項目を足した」だけではなく、今後の仕様変更にも追随しやすい形になりました。

リリースが終わったら、実際に別リポジトリの Markdown を Bundle 化して、相互運用側の使い勝手も確かめていきます。

---

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。

※ ヘッダー画像、本文挿絵、インフォグラフィックは AI（画像生成）で作成しています。本文図は HTML から生成しています。
