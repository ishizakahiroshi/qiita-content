---
title: >-
  AI に画像を投げて『ここズレてない?』を毎回聞くのに疲れたので、PowerShell + Chrome headless で機械的に検査させた話（Opus
  4.8 の実価格で節約額も出す）
tags:
  - PowerShell
  - Chrome
  - AI
  - Security
  - 自動化
private: false
updated_at: '2026-07-24T22:53:12+09:00'
id: 30f8e9854150a775183d
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![AI に画像を投げる代わりに、機械に叩かせて 92 円 / 記事を節約する話のヒーロー画像](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-24_ai-visual-loop-vs-tool_hero.png)

## 今日、1 日で 4 回同じ事故を出した

半日で 4 回、AI が同じ種類のミスをやりました。全部、自分のところで。

- 秘密が入りうるファイルを `grep -iE 'vision|gemini|ai' /etc/xxx/config/xxx.toml` みたいに雑に叩いて、API キーの生値をコンソール出力してしまった
- 顧客宛てのコピペ用文面で `- ` の行頭記号を使った（コピペ先で文字として残る運用ルール違反）
- Qiita 記事の下書きに社内プロジェクトの固有名を素で埋め込んだ（公開直前に自分で気付いた）
- 1Password の価格を「月 3.99 ドル」と出典 URL 併記なしで書いた（しかも年払い時の実勢価格は別で、当然古い）

どれもルールとしては存在しているミスです。CLAUDE.md にも書いてある、skill 本文にも書いてある。書いてある通りに実行しなかっただけ。「気を付けます」で治るかというと治らない、というのは 4 連発で証明された。

同じ日、記事に貼る図（fig）を HTML から PNG 化しているのですが、こちらも縦オーバーで中身が枠から溢れました。目視で発見 → HTML 修正 → PNG 再生成 → また目視、というループ。3 枚の fig を作るのに 30 分溶かした。

さすがに、これはもう仕組みで塞ぐしかない。

## 「AI に気を付けて」は効かない、を認めるところから

AI に対して「次からは気を付けます」と言わせる方式は破綻しています。理由は 3 つ。

1. **skill 本文や CLAUDE.md は起動時に一度読むだけ**で、作業中は「読んだ内容を実行のタイミングで思い出す」機構が無い
2. 手元では **Claude Code / Codex / OpenCode / Cursor / Grok / Ollama の 6 CLI を並行運用**しているので、Claude Code 専用の hook を仕込んでも他 5 匹には効かない
3. ルールを追加してもそのルールを読まないので、「読まないミス」に対して「読ませるルール追加」は自己言及的な矛盾

つまり、**AI の自己規律に依存しない、外から強制する仕組み**が要る。git pre-commit hook みたいなプロセスの穴じゃなくて（それも有効だけど今日は保留）、まず **数値で叩ける機械的検査**を先に用意する。

やろうと決めたのは 2 つ。

- **記事の md** に対する canonical grep 検査（linter）
- **fig の HTML** に対する Chrome headless での縦オーバー検出

どちらも AI が実行しなくていい。実行するのは私（人間）か、あるいは AI が結果を数値で受け取るだけ。判定ロジックを AI から剥がして機械に閉じ込める。

## 1 つ目: article-lint.ps1（記事の機密漏洩・書式違反を叩く）

grep パターンの canonical 化です。今日発生した 4 種を全部拾えるように、6 項目を機械的に叩きます。

- kb 由来固有名詞（社内プロジェクト名・環境名・人物名・会社名・内部ホスト名）
- API キー生値（`AIzaSy...` / `sk-...` / `sk-proj-...` / `ghp_...` / `xox...` / `AKIA...`）
- 禁止ダッシュ（U+2014 em dash / U+2015 horizontal bar）
- 内部ホスト・プライベート IP（RFC1918 系のプライベートアドレス、および internal / corp / local を含む社内ホスト名）
- 数値主張の URL 近接（価格・iter 回数・bit / byte 等の直後 ±15 行以内に URL があるか）
- 実在人物氏名の疑い（自己言及以外を目視警告）

PowerShell で書いた本体はこんな感じ。

```powershell
param(
    [Parameter(Mandatory = $true, Position = 0)][string]$File
)
$lines = Get-Content -Path $File -Encoding UTF8
$violations = @()

# API キー生値
$apiKeyPatterns = @(
    'AIzaSy[A-Za-z0-9_-]{33}',
    'sk-proj-[A-Za-z0-9_-]{20,}',
    'ghp_[A-Za-z0-9]{36}',
    'xox[baprs]-[A-Za-z0-9-]+',
    'AKIA[0-9A-Z]{16}'
)
for ($i = 0; $i -lt $lines.Length; $i++) {
    foreach ($pat in $apiKeyPatterns) {
        if ($lines[$i] -match $pat) {
            # 違反リストに追加
        }
    }
}

# 数値主張の URL 近接チェック
$numericClaimPattern = '\$\d+(\.\d+)?|\d{1,3}(,\d{3})+\s*円|\d+\s*(USD|EUR|GBP|JPY)|\d{4,}\s*(回|iter|iterations)|\d{2,}\s*%|\d+\s*(bit\b|byte\b|KB|MB|GB|TB)'
$urlPattern = 'https?://[^\s<>()]+'
# 数値行があれば ±15 行以内に URL があるかチェック
```

出力は違反ルールごとにグループ化して、行番号・該当行・詳細を吐きます。全 OK なら `exit 0` と `OK` メッセージだけ、NG なら `exit 1` で違反リスト。

```
OK: article-lint 全 6 項目合格
対象: C:\dev\workshop\docs\articles\plan\...\qiita.md
```

数値主張の URL 近接検査は、`\$\d+` みたいな価格表現や `\d+\s*bit` みたいなスペック値を検出したときに、その ±15 行以内に URL があるかを見ます。近接範囲は狭すぎると誤検出だらけになるので、実測しながら詰めた。最初 ±4 行にしたら「同じセクション内で URL は上の方にまとめて置いてるだけの記事」が false positive の嵐になった。±15 行が現状のバランス。

## 2 つ目: fig-overflow-check.ps1（Chrome headless で DOM 検査）

こっちは Chrome の頭ぶった斬りモードを使います。fig HTML を JS 実行込みで読み込ませて、`.frame` / `.card` / `.zone` 等の要素について `scrollHeight - clientHeight` を計算する。差が 2px 以上あれば「中身がボックスからはみ出してる」と判定。

![目視ループと機械検査ループの比較図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-24_ai-visual-loop-vs-tool_fig1.png)

上の図のとおり、以前は「HTML 書く → PNG 生成 → 人間が目視 → AI に『ここズレてる』と伝える → AI が画像を read → 修正案 → PNG 再生成」のループでした。今は「HTML 書く → PNG 生成 → 自動 overflow 検出 → NG なら位置と px 数を出力 → 修正 → 再生成」で、人間と AI 両方が短縮されます。

実装のポイントは JS 注入と DOM ダンプの組み合わせです。

```powershell
# 検査 JS スクリプト。head 末尾に注入する
$checkScript = @'
<script>
(function() {
  function run() {
    var selectors = ['.frame', '.card', '.zone', '.row', '.grid'];
    var overflows = [];
    selectors.forEach(function(sel) {
      document.querySelectorAll(sel).forEach(function(el, idx) {
        var overflow = el.scrollHeight - el.clientHeight;
        if (overflow > 2) {
          overflows.push(sel + '[' + idx + ']: overflow ' + overflow + 'px');
        }
      });
    });
    var marker = document.createElement('div');
    marker.id = '__fig_overflow_marker__';
    marker.style.display = 'none';
    marker.setAttribute('data-count', overflows.length);
    marker.textContent = overflows.join(' | ');
    document.body.appendChild(marker);
  }
  if (document.readyState === 'complete' || document.readyState === 'interactive') {
    run();
  } else {
    document.addEventListener('DOMContentLoaded', run);
  }
})();
</script>
'@
```

これを HTML の `</head>` の直前に差し込んだ一時ファイルを作って、Chrome headless で読ませます。JS 実行後の DOM を `--dump-dom` で取り出して、`__fig_overflow_marker__` の `data-count` と `textContent` を PowerShell 側で正規表現で拾う。

## Chrome headless で 2 時間ハマった話

「実装 30 分、デバッグ 2 時間」の典型。3 つの罠を踏みました。

**罠 1: PowerShell の `&` 演算子で Chrome の stdout が取れない**

最初はこう書いてました。

```powershell
$dom = & $chrome --headless=new --disable-gpu --dump-dom $uri 2>$null
```

これで動くはずだったんですが、`$dom` が空文字列で返ってくる。`--dump-dom` は stdout にダンプするはずなのに、なぜ空か。

原因は PowerShell の `&` 演算子が「別プロセスの stdout をキャプチャする経路が、環境によって取りこぼす」ことがあるらしい。`.NET` の `Process.Start` を使って明示的に `RedirectStandardOutput = true` にすれば取れる。

```powershell
$psi = New-Object System.Diagnostics.ProcessStartInfo
$psi.FileName = $chrome
$psi.Arguments = "--headless=new --disable-gpu --dump-dom `"$uri`""
$psi.RedirectStandardOutput = $true
$psi.RedirectStandardError = $true
$psi.UseShellExecute = $false
$proc = [System.Diagnostics.Process]::Start($psi)
$dom = $proc.StandardOutput.ReadToEnd()
$proc.WaitForExit()
```

これで取れるようになった。`&` を使わなくてはいけないケースもある（自作 skill から呼ばれる runtime 制約とか）ので、両者の使い分けは覚えておく価値ある。

**罠 2: 通常起動している Chrome が headless セッションを乗っ取る**

上の Process.Start に切り替えても、まだ stdout が空のときがある。Chrome の起動ログを見ると「既存のブラウザ セッションで開いています」と言われている。

Chrome は「既に開いているプロセスに URL を渡す」挙動が既定なので、headless で起動したつもりでも通常起動の Chrome に URL が渡って、headless プロセスは何もせずに終わる。この間の 15 分ぐらいムダにした。

対策は `--user-data-dir` で完全に独立したプロファイル指定。

```powershell
$userdata = Join-Path $env:TEMP "chrome-figcheck-profile"
# --user-data-dir=$userdata を Arguments に追加
```

これで既存の Chrome セッションに干渉されなくなる。開発機で普段使いの Chrome を開いたまま headless 検査を走らせる、というワークフローが成立するようになった。

**罠 3: JS 実行タイミングと `--dump-dom` の同期**

`--dump-dom` はページ読み込み完了時点の DOM を吐くけど、非同期に走る JS（`setTimeout` 等）の結果を待たない。最初、`setTimeout(run, 300)` で書いてたら永遠に marker が生成されなかった。

対策は `--virtual-time-budget=2000` を付ける（Chrome の内部時間を 2 秒進める）＋ JS 側は `setTimeout` を使わず `DOMContentLoaded` で即実行。両方揃うと、`--dump-dom` の出力に JS が生成した marker が入る。

```
& $chrome --headless=new --virtual-time-budget=2000 --dump-dom $uri
```

## トークン節約の実測見積もり（Opus 4.8 の実価格）

ここまでで tool は動く。次は「これ、AI トークン的にどれくらい得なのか」を実額で出したい。

**前提の一次資料**

- Claude API 料金表（<https://platform.claude.com/docs/en/about-claude/pricing>）
- Claude Vision の画像トークン計算式（<https://platform.claude.com/docs/en/build-with-claude/vision>）

Opus 4.8 の現行料金（前者から）。

- Input: **$5 / MTok**
- Output: **$25 / MTok**

画像トークンは後者に公式が載っています。

> Each patch is a 28×28-pixel block of the image, referred to as a visual token. An image, therefore, costs `⌈width / 28⌉ × ⌈height / 28⌉` visual tokens.

Opus 4.8 は High-resolution tier（max 2576 px 長辺、最大 4784 tokens）。

![目視ループと linter ループのトークンとコスト比較表](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-24_ai-visual-loop-vs-tool_fig2.png)

上の図のとおり、私の使う fig PNG は 1280×720 を 2 倍スケールで書き出した 2560×1440。この画像の視覚トークンは `⌈2560/28⌉ × ⌈1440/28⌉ = 92 × 52 = 4784` トークン（High-res tier の上限にほぼ一致）。

これを金額換算します（Opus 4.8 の Input $5 / MTok・Output $25 / MTok は公式表 <https://platform.claude.com/docs/en/about-claude/pricing>、画像トークン計算式は <https://platform.claude.com/docs/en/build-with-claude/vision> のとおり）。

- 1 枚読み込み: 4784 tokens × $5 / 1,000,000 = **$0.0239 ≒ 3.6 円**（1 USD = 150 JPY 換算）

「AI に画像を投げて修正させる」1 往復の内訳を組んでみます。

- 入力: 画像 1 枚（4784 tok）＋ 会話プロンプト（500 tok）＋ 過去のやりとり（約 5000 tok）= 約 10000 input tok
- 出力: 修正案の文章（1000 tok）
- 入力代: 10000 × $5 / M = **$0.05**
- 出力代: 1000 × $25 / M = **$0.025**
- 1 往復合計: **$0.075 ≒ 11 円**

fig 3 枚それぞれで平均 3 往復の目視レビューが必要だったとして、`3 × 3 × 11 = 99 円 / 記事`。

これが linter 経由になるとどうなるか（引き続き同じ pricing 表 <https://platform.claude.com/docs/en/about-claude/pricing> の Opus 4.8 単価で試算）。

- 入力: linter の出力（例: `.card[0]: overflow 130px`）＋ 会話（500 tok）= 約 600 input tok
- 出力: 修正案（500 tok）
- 1 往復: 600 × $5 / M + 500 × $25 / M = **$0.0155 ≒ 2.3 円**

しかも、linter の場合「130 px はみ出し」と数値で分かるので、修正の 1 発命中率が高い。実測で平均 1 往復で収まった。fig 3 枚合計で `3 × 1 × 2.3 = 6.9 円`。

**節約額: 99 円 - 6.9 円 = 約 92 円 / 記事**（試算根拠は上記 pricing / vision の 2 URL）

年 50 記事書くとして `92 × 50 = 4600 円 / 年`。おまけに人間の目視時間も 30 分削れるから、時給換算するとさらに大きい。

article-lint の方も同じ理屈。

- 目視レビュー: 石坂さん + AI が記事全文（~8,000 字 = 約 3,000 tok）を読み合いで数往復
- linter: `L86: kb-identifier "myapp" 検出` みたいな 100 tok の入力で 1 発修正

1 記事あたり 15-20 円レベルの節約になります。

**tokenizer 差分の注意**

同じ pricing ページ（<https://platform.claude.com/docs/en/about-claude/pricing>）の脚注に、Opus 4.7 以降・Fable 5・Mythos 5・Sonnet 5 は新 tokenizer で **約 30% 多くトークン化される**と書いてあります。

> Claude Opus 4.7 and later Opus models, Claude Fable 5, Claude Mythos 5, Claude Mythos Preview, and Claude Sonnet 5 use a newer tokenizer that contributes to their improved performance on a wide range of tasks. This tokenizer produces approximately 30% more tokens for the same text.
> （出典: <https://platform.claude.com/docs/en/about-claude/pricing>）

上の試算は「画像トークンは公式表の値そのまま」「テキストトークンは概算」なので、テキスト部分は実際にはもう少し多いかもしれない。オーダーは変わらない。

## 副次効果: 他 CLI にも効くのが嬉しい

これ、Claude Code 専用の hook で作らなかったのが大きい。**PowerShell 単体で動く CLI 独立ツール**として作ったので、Codex でも Cursor でも Grok でも、`& 'C:\dev\tools\article-lint.ps1' '<file>'` を叩けば同じ検査が走る。

事故が起きたら linter に規則を 1 行追加するだけで、6 匹全員に自動で新しいチェックが効くようになる。AI 側の設定を触らない。これは大きい。

もう 1 個、副次効果というよりオマケ寄りの対策として、**過去の失敗集**をグローバル `~/.claude/guides/` に置きました。ルール条文ではなくて、事故の現物を「状況 → やったこと → 症状 → 原因 1 行 → 教訓 1 行」で並べる。

抽象化しないで生々しく残す方針で書いた。「秘密が入りうるファイルへの grep は `-c` / `-o` / `^KEY_NAME` に絞る」みたいに教訓化するより、「grep -iE 'vision|gemini|ai' で ai の 2 文字が chatgpt_api_key にマッチして生値流出した」と現物で残した方が、次に読んだ AI（or 私）が引っかかる気がする。

これは linter で塞げない類のミス（設計判断ミスや調査アプローチのミス）向けの、感情記憶に頼る保険。ただし本命はあくまで機械的検査（linter）で、事故集は「読むだけで実行はしない」タイプなので効果は限定的、と割り切ってる。

## 結論: AI を信じずツールに聞け

大した思想でもなくて、単に「ルールを AI に読ませて実行させる」を諦めた話です。実行してくれないんだから、実行しなくても機械が代わりに叩く形に持っていく方が早い。

面白かった学びを 4 つ。

- **AI に「気を付けて」は原理的に効かない**。手抜きじゃなくて構造的な限界。実行のタイミングでルールを思い出せない
- **canonical grep パターンをファイルに固定するだけで、6 CLI 全部で同じ検査が回る**。CLI 独立に作るのが強い
- **fig の縦オーバーみたいな視覚判定も、DOM の `scrollHeight` / `clientHeight` 差分で機械化できる**。「なんとなくズレてる」を AI に判定させる必要は無い
- **Opus 4.8 の実価格で計算すると、目視ループから機械検査ループへの移行は 1 記事あたり 90 円くらいの節約**。年 50 記事なら 4,500 円、10 年で 4.5 万円。人間の時間コストは別

「AI に真面目にチェックさせる」より「機械が確実にチェックする」の方が、金銭的にも時間的にも精度的にも有利。当たり前を当たり前にやるだけ。

明日以降、同型の事故が起きたら linter に規則を足していきます。事故を教科書にする方式。多分また明日か明後日、新しい種類の事故が発生するので、それも記事にできそう。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-07-24_ai-visual-loop-vs-tool/

---

書いた人: ishizakahiroshi

Web エンジニア 18 年目。バックエンド・インフラ・AI 連携が主戦場です。

- ポートフォリオ: <https://ishizakahiroshi.com/>
- GitHub: <https://github.com/ishizakahiroshi>
- X: <https://x.com/ishizakahiroshi>
