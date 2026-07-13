---
title: 自作 OSS に HTML 対応を足そうとして、100 行書いてから設計を revert した話
tags:
  - Python
  - リファクタリング
  - 設計
  - 開発プロセス
  - OSS
private: false
updated_at: '2026-07-14T02:23:13+09:00'
id: f0b3bfa30f11c70842a4
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![書いた 100 行を、戻した - docsweep への HTML 対応で気づいた設計転換](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260714-docsweep-html-meta-refactor/hero.png)

自作の OSS を拡張していた日のことです。パーサに新しい機能を 100 行ほど足したところで、ふと違和感がありました。追加している場所が違うのかもしれない、と。

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260714-docsweep-html-meta-refactor/infographic-v2.png)

## docsweep に HTML 資料も認識させたかった

自作で [docsweep](https://github.com/ishizakahiroshi/docsweep) という OSS CLI を作っています。ざっくり言うと、AI コーディングツール（Claude Code / Codex CLI / GitHub Copilot CLI 等）が生成する `plan_*.md` や `bugfix_*.md` の陳腐化を管理する道具です。frontmatter とファイル名から状態を読み取り、完了したものを archive フォルダに寄せてくれます。Web UI（`docsweep serve`）で複数プロジェクトを横断して眺めることもできます。

同じ悩みを持っている方は、pip で入ります。

```bash
pip install docsweep
docsweep brief   # 今日 1 個だけ取り組むならこれ、を提案してくれる
docsweep serve   # ブラウザで横断ボードを開く
```

リポジトリはこちらです（Star していただけると励みになります）: https://github.com/ishizakahiroshi/docsweep

その docsweep で今のところ認識できるのは md ファイルだけでした。でも実務では、モックの HTML や検討シートの HTML も同じ `docs/local/` に溜まっていきます。これらも docsweep から見えるようにしたい、というのが今回の題材でした。

## 最初は「パーサを HTML 対応に拡張する」で考えた

frontmatter を持てない HTML に、frontmatter 相当のメタ情報を書く方法として、HTML コメントを使うことにしました。

```html
<!DOCTYPE html>
<!--docsweep-meta
type: mockup
status: draft
tags: [ui, admin]
related: [plan_admin.md]
docsweep_policy: archive_with_release
-->
<html lang="ja">
```

docsweep 側でこの HTML コメントを見つけて YAML として読めれば、md の frontmatter と同じ構造の辞書が手に入ります。ここまでは素直な設計です。

問題はここからでした。HTML には mockup や review-sheet や incident など複数の種類があります。docsweep は md のとき、ファイル名パターン（`plan_*.md` など）から type を判定していました。HTML でも同じ枠に乗せるか、それとも meta の `type:` を使うか。私は後者を選び、meta から拾った type で `TypeDef` を動的に合成するコードを書きました。

さらに HTML 側では `draft` `kept` `decided` という語彙で状態を表現したい場面があります。docsweep の内蔵語彙は `planned` `watching` `done` などで、これらとは違います。そこで状態語彙のエイリアス機構を拡張し、`draft` を `planned` に、`kept` を `watching` に、`decided` を `done` に読み替える定義を追加しました。

`detect.py` に HTML コメント抽出の正規表現、`scan.py` に `.html` を対象拡張子に追加、`TypeDef` を合成するヘルパ、状態モデルの extra_aliases 追加、`docsweep_policy` フィールド認知。差分は 100 行を超えていました。テストも書き足しました。全 529 → 533 で通っています。

## 「ちょっとまてよ」

ここで一度、進捗を報告しました。skill 側もこの流儀に合わせて修正する段取りに入ろうとしたところで、ユーザーから 1 言もらいました。

> HTML を作る skill 側を docsweep 対応させれば、ややこしいこといらないのでは？
>
> ディレクトリを作って HTML を保存するのではなくて、必ず docs/local か、ユーザーが指定してるディレクトリに入れるとか、もっと合理化できないかな。

読んだ瞬間、手が止まりました。

## 増やす場所を間違えていた

冷静に整理すると、複雑さは 2 か所から生まれていました。1 つは「新しい type をどうやって docsweep に教えるか」、もう 1 つは「HTML の状態語彙を docsweep の内蔵語彙にどう合流させるか」です。

私は両方とも「docsweep 側で受け入れられるように広げる」方向に解こうとしていました。パーサに機能を足し、状態モデルに別名を足し、TypeDef を動的合成する。docsweep 本体が「なんでも読める」ようになる方向です。

でも生成側の skill が、docsweep の既存の流儀に合わせて出力してくれれば、どちらの複雑さもいらないのでした。ファイル名を `mockup_<topic>_YYYY-MM-DD.html` にすれば、docsweep 側は既存の filename pattern マッチだけで type を判定できます。状態も `planned` `watching` `done` の標準語彙で書けば、alias 不要です。

パーサに機能を足す前に、生成側を疑う。書いてしまえば当たり前ですが、実装を始めるとどうしても「今触っているコードの中で解く」バイアスが働きます。

![設計の Before/After](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260714-docsweep-html-meta-refactor/fig1.png)

上の図は方針転換の前後を並べたものです。左側が最初の案で、docsweep 本体に 3 系統（HTML パーサ・synthetic TypeDef・状態 alias）を足す形。右側が転換後で、skill 側でファイル名と状態語彙を docsweep の流儀に揃え、docsweep 本体には HTML コメントの読み取りだけ残す形です。

## revert して skill 側で解く

方針が決まったら、あとは戻すだけです。

- `states.py` の extra_aliases（`draft` / `kept` / `decided`）を revert
- `scan.py` の synthetic TypeDef 合成を撤去
- `config.py` の `DEFAULT_TYPES` に `mockup_*.html` `review_*.html` `design_*.html` `incident_*.html` を追加（既存の `plan_*.md` と同じ流儀で並ぶ）
- `detect.py` の HTML コメント抽出は温存（`_parse_frontmatter_dict` が md/html 両対応で 1 か所に集約）
- `engine.py` の `docsweep_policy: never_archive` 尊重は温存（sweep/promote/apply が policy を見て自動移送から外す）

skill 側では、design-html と review-sheet の出力を `<type>_<topic>_YYYY-MM-DD.html` にフラット命名で統一しました。保存先は「起点になった `plan_*.md` があればそのフォルダに colocate、なければ `docs/local/`」という優先順位です。案件によっては `docs/service-a/` や `docs/service-b/` のようにドメイン別サブフォルダで運用しているプロジェクトもあるので、親 md と同じ場所に置く運用にすれば自動でドメイン分離が守られます。

![親 md と colocate](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260714-docsweep-html-meta-refactor/fig2.png)

上の図は「親 md と colocate」を絵にしたものです。ドメイン別のサブフォルダ運用でも、親の plan/bugfix と同じ場所に子の HTML が並ぶので、リリース時に一括で archive されます。

## 学び

新しい要件を受け入れるとき、「受け入れる側」を広げるか「持ち込む側」を作法に合わせるかは毎回選べます。前者は柔軟で聞こえがいいのですが、コアの複雑さが増える方向です。後者は生成側で規約を守れば、コアはそのままで済みます。

今回の場合、docsweep はテストが 500 件を超える育ったコアで、ここに if 分岐や状態合流を増やすと、あとから読む人が「なぜこう分かれているのか」を追う負担が積み上がります。一方で skill は文字ベースの薄いテンプレで、書き手（AI）が毎回参照するだけなので、規約を厳しくしても運用コストは大きくありません。

コアの複雑さと辺境の規約は、どっちを厚くしても機能は成立します。厚くする場所を選ぶだけで、あとから読む人の負担がまるで違う。手を動かしていると忘れがちなので、コミット前に一度立ち止まって「これ、逆じゃない？」と自問する習慣にしていきたいところです。

もう 1 つ副次的な学び。テストが通っている状態からの revert は、コード上は「消すだけ」でも、判断としては「増やしたものと同じ責務を別の場所で受ける」宣言です。テストが通っているからと惰性で残さず、宣言と一緒に動かすと、あとで意図を追いやすくなります。

## docsweep はこんなときに刺さります

- AI コーディングツールを使い始めてから `plan_*.md` や `bugfix_*.md` が溜まって放置状態になっている
- 複数プロジェクトを行き来していて「今日はどれから触るんだっけ」を毎朝迷う
- 完了した md を毎回手で archive フォルダに移すのが面倒
- Web UI で横断カンバンを開いて棚卸ししたい

いずれかに心当たりがあれば、`pip install docsweep` で 1 分で試せます。設定ゼロで動きます。

- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/docsweep
- PyPI: https://pypi.org/project/docsweep/

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## おわりに

小さく。書いた 100 行を、必要なら戻す勇気を持つ。これを、これからも習慣にしていきます。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
