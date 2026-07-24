---
title: 'Claude Code の体感バグを数字にする: 3 モデル resume 比較の手順'
tags:
  - Claude
  - ClaudeCode
  - LLM
  - AI
  - debug
private: false
updated_at: '2026-07-24T22:12:05+09:00'
id: 990dd55e53690ca0a4bd
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-24_claude-code-empirical-bug-measurement_hero.png)

## この記事で結局何をやるか（先に結論）

Claude Code のセッションで「なんかおかしい（stall する、tool を呼んでないのに結論だけ書く 等）」と感じたとき、体感を **他人と共有可能な数字** に落とすための手順。

やることは 3 つだけです。

1. セッション jsonl のパス（`~/.claude/projects/<cwd>/<session-uuid>.jsonl`）を控える
2. `claude --resume <session-uuid> --model <別 model>` で **同じ context を別 model にそのまま渡し直す**
3. 追記された jsonl を差分抽出し、以下を機械集計する: thinking-only ターン数 / auto-inject 回数 / thinking-only tokens / 着地までの秒 / 実 tool 実行の有無

この 3 つが揃うと「n=1 の個体差では？」という反論を潰した形で GitHub Issue や社内 Slack に共有できます。以下、手を動かせる粒度で書いていきます。

## 動機（読み飛ばして OK）

ここ最近、Claude Opus 4.8 の作話や stall について自分の体感を [記事にした](https://note.com/ishizakahiroshi/n/n5015e3856e93)、[別の日にも書いた](https://note.com/ishizakahiroshi/n/n020cff07baf0)。書くたびに強くなる反論が「n=1 の個体差では？」でした。

個体差でないことは、[Anthropic 公式リポの issue 群](https://github.com/anthropics/claude-code/issues) を見れば一次情報で押さえられます（本稿末尾に一覧を置きます）。ただ、自分のセッションで起きたことを **他人と比較可能な数字に落とす方法** を持っておくと、個別 issue の再現テスト提供や、切替判断の合理化に役立つと思いました。

本稿は、そのための手順書です。特定の model が悪いと告発するのが目的ではなく、「自分の目の前のセッションが、他人と共有可能な形で本当に stall / 作話しているか」を測る道具立てを書いています。

## 前提: Claude Code のセッションログはローカルに残っている

Claude Code は各会話を以下に構造化 JSON Lines で残しています。

```
~/.claude/projects/<cwd を「/」→「-」に置換したエンコード>/<session-uuid>.jsonl
```

たとえば `C:\dev\github\private\my-work` で作業していたら `C--dev-github-private-my-work/` 配下です。

この jsonl は各行が 1 イベントで、assistant の 1 ターンは以下の構造で入っています。

```jsonc
{"message":{
  "role":"assistant",
  "model":"claude-opus-4-8",
  "content":[
    {"type":"thinking","thinking":"", "signature":"..."},   // 拡張思考
    {"type":"text","text":"..."},                            // 本文
    {"type":"tool_use","name":"Bash","input":{...}}          // ツール呼び出し
  ],
  "usage":{"output_tokens":1594, ...}
}, "timestamp":"..."}
```

**stall や作話の分析は、この生 jsonl を読むのが正典**です。ラッパー越しの PTY 描画後バイト列（`Baked for 5m` などの thinking ラベル）は情報密度が低く、原因特定には向きません。

## 起きていた症状（この記事の n=1 部分）

Opus 4.8 の 1M context session で、途中から次の 2 症状が併発しました。

1. **reasoning-only stall**: assistant ターンが `thinking` block のみで返り、`text` も `tool_use` も含まない状態が連続する。Claude Code CLI は「本文なし」を検知して `[Your previous response had no visible output. Please continue and produce a user-visible response.]` を user turn として自動注入するが、Opus 4.8 はその回復プロンプトにも thinking-only で応じる。ループが止まらない
2. **未実行の完了扱い**: tool 呼び出しをスキップし、context 内の断片情報から暗算した結論を「関数呼び出し結果」として提示する

このセッションでは stall 総計 32 分、thinking token 累計 111,000、auto-inject 4 回。1 ターンで `output_tokens=64,000` を全部 thinking に燃やした瞬間があり、これは API の max_tokens 上限にちょうど一致するため、**API 側で強制打ち切りされてやっと text 出力に着地した**と読めます。天井に当たらなければ理論上は延々続きます。

![reasoning-only stall の 3 層ループ図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-24_claude-code-empirical-bug-measurement_fig2.png)

図にすると、Claude Code CLI の「本文なしを検知したら回復プロンプトを注入する」という safety net が、Opus 4.8 の thinking-only 応答を誘発し続ける 3 層ループとして描けます。回復用の仕組みが、そのまま悪化装置として動いている構図です。

（このあたりは既に複数の [GitHub Issue](https://github.com/anthropics/claude-code/issues) に上がっている症状で、本稿の主眼はここではありません。）

## 本題: 3 モデル resume 比較で切り分ける

「n=1 の個体差」反論を潰すために、**同一 session の context を継承したまま model だけ変える**手順を書きます。1 モデルの再現ではなく、複数モデルを揃えて出力の差分を見るのがポイントです。

### なぜ resume なのか

- 新規セッションを 3 つ立てて同じプロンプトを投げると、context が違うので比較できない
- resume を使えば、同じ会話履歴・同じ system prompt・同じ tool 応答履歴を持ったまま、次の 1 ターンだけ違う model で処理できる
- Claude Code は `claude --resume <session-uuid> --model <model-id>` で jsonl の続きに追記する形で resume する

### 手順

#### 0. 事前スナップショット

比較を汚さないため、実験前に jsonl をコピーしておきます。

```bash
cp ~/.claude/projects/<encoded-cwd>/<session-uuid>.jsonl \
   ./snapshot_before-experiment_$(date +%Y-%m-%dT%H-%M).jsonl
```

#### 1. 実験プロンプトを 1 つに決める

3 実験通じて **同一プロンプト**を使います。差分の原因を model 以外に持たせないためです。今回私が使ったのはこれ。

```
<業務固有の検証タスクの短い記述>。
考察は最小限にして、まず 1 発クエリまたはコマンドを叩いて結果を出してください。
```

最後の 1 文「考察は最小限に」は thinking 抑制の実験変数です。**この 1 文が有ると無いで挙動が変わる**ので、目的に応じて付ける・外すを決めます。付ければ stall は防げます。外せば「素の状態」を測れます。

#### 2. 元 model のまま実行（実験 A）

現在の session をそのまま使い、上のプロンプトを 1 発投げます。結果を待ちます。

#### 3. resume + model 切替で実行（実験 B, C）

```bash
cd <元 session の cwd>
claude --resume <session-uuid> --model claude-opus-4-7
```

`--model` に流したい ID をそのまま渡します。ピッカーに出てこないモデルでも `claude-opus-4-7[1m]` のように直接指定すれば通ることがあります。

実験 B が終わったら Ctrl+D で抜けて、実験 C で `--model claude-sonnet-5` に置き換えて同じことをやります。

（同じ session-uuid の jsonl に追記されるので、あとで差分抽出できます。）

### 計測項目

jsonl から機械集計できる項目です。

| 項目 | 集計方法 |
|---|---|
| **thinking-only ターン数** | `role=assistant` で `content` が `thinking` のみ（`text` も `tool_use` も無い）ターンの数 |
| **auto-inject 回数** | `role=user` の `content` に `"no visible output"` を含む行の数 |
| **thinking-only tokens** | 上記 thinking-only ターンの `usage.output_tokens` の合計 |
| **着地までの秒** | プロンプト投入 timestamp から、最初の `text` or `tool_use` を含む assistant ターンまでの経過秒 |
| **実タスク完遂** | 期待値と一致する数値が出たか（正解を事前に知っていることが前提） |
| **実際に tool 実行したか** | 目的の tool（今回なら検証対象の関数を叩く Bash）が `tool_use` として現れたか |
| **前ターンの作話を検出したか** | resume 直後の assistant の第一声が、直前の assistant ターンの誤りを指摘しているか |

Python でも Node でもいいです。以下は Python の骨だけ。

```python
import json
from datetime import datetime

events = [json.loads(l) for l in open(jsonl_path, encoding='utf-8') if l.strip()]
# 実験区間の切り出しは、実験前 snapshot の event 数を len(snap) として events[len(snap):] を取る

think_only = 0
auto_inject = 0
think_tokens = 0
first_ts = None
land_ts = None

for e in events:
    ts = e.get('timestamp')
    if ts and not first_ts:
        first_ts = ts
    msg = e.get('message') or {}
    role = msg.get('role')
    if role == 'user':
        c = msg.get('content')
        if isinstance(c, str) and 'no visible output' in c.lower():
            auto_inject += 1
    elif role == 'assistant':
        content = msg.get('content', [])
        has_think = any(x.get('type') == 'thinking' for x in content)
        has_text = any(x.get('type') == 'text' for x in content)
        has_tool = any(x.get('type') == 'tool_use' for x in content)
        if has_think and not (has_text or has_tool):
            think_only += 1
            think_tokens += msg.get('usage', {}).get('output_tokens', 0)
        elif (has_text or has_tool) and not land_ts:
            land_ts = ts

if first_ts and land_ts:
    d1 = datetime.fromisoformat(first_ts.replace('Z', '+00:00'))
    d2 = datetime.fromisoformat(land_ts.replace('Z', '+00:00'))
    print(f"landing: {(d2 - d1).total_seconds():.1f}s")

print(f"thinking-only turns: {think_only}")
print(f"auto-inject: {auto_inject}")
print(f"thinking-only tokens: {think_tokens}")
```

### 実測例（今回のケース）

同一 session（context ~330k）を 3 モデルで走らせた結果です。

| 指標 | A: Opus 4.8（元 model） | B: Opus 4.7（resume） | C: Sonnet 5（resume） |
|---|---|---|---|
| 着地秒 | 208s | 7.8s | 8.6s |
| thinking-only ターン | 3 | 1 | 1 |
| thinking-only tokens | 16,665 | 518 | 440 |
| auto-inject | 1 | 0 | 0 |
| 実タスク完遂 | ✓ | ✓ | ✓ |
| **実際に tool 実行したか** | **✗（暗算偽装）** | ✓ | ✓ |
| **前ターンの作話を検出** | - | **✓ 明示的に指摘** | ✗ 黙って再実行 |
| 応答フッター model 誤記 | - | - | **✓ "Opus 4.8" と誤記** |

![3 モデル比較の棒グラフ（着地秒 / thinking-only tokens）](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-24_claude-code-empirical-bug-measurement_fig1.png)

同じ context に同じプロンプトを流し込んだのに、着地秒で 27 倍、thinking-only tokens で 32 倍の差が出ています。表の縦スクロールでは見落としがちなスケール差を、並べて眺めてもらうための図です。

数字にすると、体感の色々が言語化されます。

- **着地時間で 27 倍差**（4.7 と 4.8）。「速い / 遅い」ではなく 27 倍
- **thinking-only tokens で 32 倍差**。同じ context を読ませているのに、消費している「見えない思考」の量が桁違い
- 実験 A の 4.8 は、実タスクは正解値に着地したが、**実際には目的 tool を呼んでおらず、context 断片からの暗算結果を関数呼び出し結果として提示**していた。しかも暗算が偶然正解と一致したのでユーザー目線では気付きにくい
- 実験 B の 4.7 は resume 直後に **「前回の応答は実際に実行せず結論だけ書いてしまいました」と明示的に自己修正**した。この behaviour は多段レビュー AI として実用的
- 実験 C の Sonnet 5 は真面目に tool を呼び直したが、**応答定型部分の "使用モデル" 欄を context の "Opus 4.8" に引きずられて誤記**した

数字だけ見れば「4.8 は default から外すべき」で終わりますが、それは既に [別記事で告発済み](https://note.com/ishizakahiroshi/n/n020cff07baf0) なので、本稿では**測り方の話**に絞ります。

## この方法論を自分のセッションで使うとき

![朝の机で体感バグを数字に落とし込んでいく静物](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-24_claude-code-empirical-bug-measurement_illustration-morning-desk.png)

朝のコーヒーの湯気の横で、体感を数字に置き換えていく作業を静かに続ける、そういうトーンで書いています。派手な告発ではなく、地味な計測の積み重ねが本稿の主旨です。

上の手順は、Opus 4.8 を告発する道具ではなく、「今のセッションが本当におかしいか」を数字にする道具です。以下のようなときに使えます。

- **新しい model が出た**: 既存タスクの session を resume して新 model に流し込み、旧 model との差分を測る
- **thinking effort 設定を変えた**: 同じ context で `--thinking high` / `medium` / `off` を切り替え比較
- **system prompt を変えた**: resume ではできないが、同じプロンプトを別 session でぶつけて差を測る（context 分は control できないので前提として頭に入れておく）
- **バグレポート提出**: n=1 の体感ではなく、上記の計測項目を添えた再現手順として書けば、Anthropic 側の再現テストに直接使ってもらえる

## 補遺: 関連する既知 issue と一次情報

本稿の症状はすべて Anthropic の公式リポで先行報告されています。方法論を試して自分のケースを裏付けたい人は、以下のどれかに紐付けてコメントするか、独立 issue を立てるのが良いと思います。

- [#63358 Opus 4.8 returns empty thinking blocks](https://github.com/anthropics/claude-code/issues/63358): API default が `display: "summarized"` → `display: "omitted"` に変わった仕様変更。thinking token は課金されるが JSON の中身は空
- [#63604 Opus 4.8 repeatedly emits malformed tool_use blocks, entire response discarded](https://github.com/anthropics/claude-code/issues/63604): reasoning-only stall の直接的原因説明として最も近い
- [#64076 Claude 4.8 Opus hallucinating tool outputs without execution](https://github.com/anthropics/claude-code/issues/64076): 「未実行の完了扱い」に該当
- [#64325 Opus 4.8 hallucinating security incidents and fabricating evidence during long tasks](https://github.com/anthropics/claude-code/issues/64325)
- [#63884 Opus 4.8 starts hallucinating results before parallel tasks finish](https://github.com/anthropics/claude-code/issues/63884)
- [#34218 Claude Code hangs indefinitely during thinking phase](https://github.com/anthropics/claude-code/issues/34218)

Anthropic 公式は Opus 4.8 リリース時に「Opus 4.7 の 4 倍、書いたコードの欠陥を見逃さない honesty & reliability の model」と発表しています。**この公式の説明と、上記 issue 群および本稿の実測との乖離**が現状です。

## まとめ

- Claude Code のセッションは `~/.claude/projects/<cwd>/<uuid>.jsonl` に構造化ログとして残っている
- 同一 session を `claude --resume <uuid> --model <id>` で継承すれば、context を固定したまま model だけ変えて比較できる
- 計測項目（thinking-only ターン / auto-inject / thinking tokens / 着地秒 / 実行 tool の有無 / 作話検出）を jsonl から機械集計すれば、体感が数字になる
- 数字にすれば、GitHub issue の再現データとして通用する形で共有できるし、model 切替判断も個体差反論を潰した形で下せる

小さな決意として、これからは体感バグを見つけたら「まず数字にする」ところまでを 1 セットにして、記事や issue へ返していきたいと思っています。

---

※ ヘッダー画像は AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
