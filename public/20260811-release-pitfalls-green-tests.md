---
title: "手元の pytest が緑でもリリースしてはいけない。タグを打つまでに踏んだ4つの穴（CI / CSP / 依存上限 / wheel）"
tags:
  - Python
  - CI
  - CSP
  - リリース
  - packaging
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![タイトル画像](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_release-pitfalls_hero.png)

## 先に結論

自作 CLI の新バージョンを出すとき、手元の `pytest -q` は 751 件すべて緑でした。それでもリリースは 4 回止まりました。

| 踏んだ穴 | 手元で出なかった理由 | 対処 |
|---|---|---|
| 依存のメジャー更新で API が消えた | 手元は古い版が固定されていた | バージョン範囲に上限を切る |
| 期待値が OS 依存だった | 手元が Windows、CI が Linux | 期待値を実物から組み立てる |
| CSP が inline script を拒否していた | テストが HTML を実行しない | 外部 js へ出す・テストで禁止する |
| wheel の build だけが落ちた | build はタグを打つまで走らない | ローカルで build して中身を見る |

共通する原因はひとつでした。**出荷する成果物と同じ経路を、一度も通していなかった**ことです。

以下、順に何が起きたかを書きます。急ぎの方は 3 番目（CSP）だけ読んでください。いちばん静かに壊れます。

## 何のツールの話か

自作で [docsweep](https://github.com/ishizakahiroshi/docsweep) という、AI が量産する作業ログ Markdown を片付けるコマンドラインツールを作っています。今回はその v0.4.0 のリリース作業です。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=docsweep
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/docsweep

同じ悩みを持っている方は、下記で入ります。

```bash
pip install docsweep
```

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_release-pitfalls_infographic.png)

## 穴 1: 依存の下限だけ上げて、上限を切っていなかった

脆弱性の対応で、MCP SDK の依存を `mcp>=1.0` から `mcp>=1.28.1` に引き上げました。下限を上げただけです。

push したら CI で 13 件落ちました。

```
ModuleNotFoundError: No module named 'mcp.server.fastmcp'
RuntimeError: MCP には mcp extra が必要です: pip install 'docsweep[mcp]'
```

CI のログを遡ると、解決されていたのは `mcp-2.0.0` でした。2.0 で `mcp.server.fastmcp` が無くなっていて、MCP サーバーが import の時点で落ちます。

手元は `mcp 1.27.2` でした。**引き上げたはずの下限すら下回っている版**が入ったままで、だから 1 件も落ちなかった。editable install のままだと依存の再解決が走らないので、`pyproject.toml` を書き換えても手元の環境は変わりません。

```toml
# 下限は CVE 対応で必要。上限は「fastmcp を import している」から必要
mcp = [
    "mcp>=1.28.1,<2",
]
```

教訓としてはこれだけです。**自分のコードがその依存の特定モジュールを import しているなら、上限を切る**。下限だけ上げるのは、脆弱性を塞ぎながら別の穴を開ける動きになり得ます。

上限を切ったら、外す条件（新メジャーへの対応）を保留メモとして起票しておきます。切りっぱなしにすると、次に困るのは半年後の自分です。

## 穴 2: 期待値が OS に依存していた

同じ CI で、まったく毛色の違うテストが 2 件落ちていました。

```
AssertionError: assert 'closeout-check --path <parent-plan> --json' in '## docsweep ...（生成されたガイダンス文の全体）'
```

AI エージェント向けのガイダンス文を生成する機能があり、その中でコマンド例を組み立てています。組み立ての実装はこうです。

```python
def _shell_command(parts: list[str]) -> str:
    if os.name == "nt":
        return subprocess.list2cmdline(parts)
    return shlex.join(parts)
```

`shlex.join` は POSIX のクォート規則で組み立てるので、`<parent-plan>` は**リダイレクトと解釈されないように**クォートされます。

- Windows: `... closeout-check --path <parent-plan> --json`
- Linux: `... closeout-check --path '<parent-plan>' --json`

これは実装が正しい挙動です。悪いのはテストのほうで、Windows での見た目をそのまま期待値に書いていました。

```python
def _closeout_cmd_fragment() -> str:
    """その OS の引用規則で組み立てた実物を期待値にする。"""
    from docsweep.inject import docsweep_command
    cmd = docsweep_command("closeout-check", "--path", "<parent-plan>", "--json")
    return cmd.split("-m docsweep ", 1)[1]
```

**プラットフォーム差を吸収する関数があるなら、その出力をベタ書きで写さない**。同じ関数で組み立てたものと比べれば、どの OS でも意味は同じままです。

## 穴 3: 自分のアプリの CSP が、自分の画面を殺していた

ここが本題です。

このツールには読み取り専用の Web UI があります。関係図を描くページだけが、表示のたびに外部 CDN からライブラリを取っていたので、今回のリリースで同梱に切り替えました。

同梱は成功しました。テストも通っています。テンプレートに `https://` で始まる外部参照が残っていないことを機械的に検査していて、これは緑。

それで、リリース前の目視で画面を開いたら**真っ白**でした。

![見えないガラスに阻まれる紙飛行機](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_release-pitfalls_illustration-1.png)

ヘッダーには「32 nodes / 57 edges」と出ています。データは届いている。なのにキャンバスに何も描かれない。

観測したことを順に並べます。

- ネットワークのリクエストはページ本体とライブラリの 2 本だけ。どちらも 200。**外部へのアクセスは 0 件**（同梱自体は成功している）
- `typeof cytoscape === "function"`（ライブラリは読めている）
- 描画先の `div` は 1282 × 804 で存在している
- **`canvas` が 0 枚**（＝初期化コードが走っていない）
- コンソールにエラーが出ていない

初期化コードはページ内の inline `<script>` に書いてありました。そしてレスポンスヘッダはこうでした。

```
Content-Security-Policy: default-src 'none'; script-src 'self'; style-src 'self' 'unsafe-inline'; ...
```

`script-src 'self'` に `'unsafe-inline'` がありません。**inline script が実行を拒否されていた**のです。

疑いを確定させるため、そのページ上で inline script を動的に足して、CSP 違反イベントを拾いました。

```javascript
window.__v = [];
document.addEventListener('securitypolicyviolation', e =>
  window.__v.push(e.violatedDirective + '|' + e.blockedURI));
const s = document.createElement('script');
s.textContent = 'window.__inlineRan = true;';
document.body.appendChild(s);
// → __inlineRan は undefined、__v は ["script-src-elem | inline"]
```

同じ理由で、他の 2 ページも死んでいました。片方は `onclick=` で呼んでいた関数が定義されず、コピーのボタンが無反応。もう片方は `submit` ハンドラが 1 つも付かず、フォーム全体が沈黙。**画面には何のエラーも出ません。**

### なぜこれが最悪の壊れ方なのか

CSP による inline のブロックは、コンソールを開けば警告が出ます。逆に言えば、**コンソールを開かなければ何も分からない**。

- 画面は普通にレンダリングされる（HTML と CSS は通る）
- サーバーのログにも何も出ない（リクエストは 200）
- テストも通る（HTML を実行しないので）
- コードを読んでも異常に見えない（JavaScript としては正しい）

「同梱できているか」を確認していたのに、「見えているか」を一度も確認していなかった。完了条件の書き方の問題でもあります。

- 悪い: 「ライブラリを外部ネットワークなしで読み込めている」
- 良い: 「ページを開いてノードが描画されている（外部リクエスト 0 件）」

前者は中間状態です。そこで検証が止まると、その先が抜けます。

### 直し方

厳格な CSP の下でサーバー側の値を JavaScript へ渡すには、**データ島**が使えます。`<script type="application/json">` は実行されないので、CSP のブロック対象になりません。

```html
<div id="cy"></div>

<script type="application/json" id="graph-data">{{ graph|tojson }}</script>
<script src="/static/graph.js"></script>
```

```javascript
// /static/graph.js
const dataEl = document.getElementById("graph-data");
const GRAPH = JSON.parse(dataEl.textContent);
```

inline のイベント属性（`onclick=` など）も同じく拒否されるので、`data-*` 属性と委譲に置き換えます。

```html
<button type="button" data-action="copy-context" data-path="{{ path }}">コピー</button>
```

```javascript
document.addEventListener("click", (e) => {
  const btn = e.target.closest('[data-action="copy-context"]');
  if (!btn) return;
  copyContext(btn.dataset.path || "");
});
```

![同じ値でも渡し方で通るか死ぬかが変わる](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_release-pitfalls_fig1.png)

並べてみると、渡している値は同じでも「inline に置くか、外部 js とデータ島に分けるか」だけで実行されるかどうかが変わります。左の書き方はブラウザが黙って捨てるので、画面にもログにも痕跡が残りません。

そして、二度と同じ形で壊れないようにテストで縛ります。既存のテストは「外部 script を参照していないか」しか見ておらず、この事故を素通ししていました。

```python
SCRIPT_TAG = re.compile(r"<script\b([^>]*)>(.*?)</script>", re.IGNORECASE | re.DOTALL)
EVENT_ATTR = re.compile(r"\son(?:click|change|submit|input|load|error|key\w+|focus|blur)\s*=", re.IGNORECASE)

def test_templates_have_no_inline_script() -> None:
    offenders: list[str] = []
    for tpl in sorted(TEMPLATES.rglob("*.html")):
        text = tpl.read_text(encoding="utf-8", errors="replace")
        for m in SCRIPT_TAG.finditer(text):
            attrs, body = m.group(1), m.group(2)
            if "src=" in attrs.lower():            # 外部読み込みは OK
                continue
            if "application/json" in attrs.lower():  # データ島は実行されない
                continue
            if not body.strip():
                continue
            offenders.append(f"{tpl.name}:{text[: m.start()].count(chr(10)) + 1} inline script")
        for m in EVENT_ATTR.finditer(text):
            offenders.append(f"{tpl.name}:{text[: m.start()].count(chr(10)) + 1} inline イベント属性")
    assert not offenders, f"CSP (script-src 'self') が inline を拒否します: {offenders}"
```

修正後、同じ手順で計測し直しました。canvas が 3 枚生成され、ノードが描画され、リクエストは自分のオリジンの 3 本だけ、CSP 違反は 0 件。ここまで見て初めて「直った」と言えます。

## 穴 4: wheel の build は、タグを打つまで走らない

CI を緑にして、`main` へマージして、タグを push しました。publish のワークフローが動いて、**build のジョブで落ちました**。

```
ValueError: A second file is being added to the wheel archive at the same path:
`docsweep/okf_profiles/0.2.json`

The most likely cause of this is an entry in the
`tool.hatch.build.targets.wheel.force-include` table.
```

原因は単純です。

```toml
[tool.hatch.build.targets.wheel]
packages = ["docsweep"]

# ↓ packages が既に含んでいるパスを、もう一度足していた
[tool.hatch.build.targets.wheel.force-include]
"docsweep/okf_profiles" = "docsweep/okf_profiles"
```

パッケージ配下のデータファイルに `force-include` は不要です。以前のバージョンでは通っていたので、hatchling 側が重複をエラー扱いにするようになったのだと思います（この点は未確認です）。

幸い build 段階で落ちたので、PyPI にも GitHub Release にも成果物は 1 つも出ていませんでした。タグを削除して同じ番号で打ち直せます。**逆に言えば、publish が成功した後だと同じ番号は二度と使えません。**

ついでにもう 1 つ。hatchling は VCS の追跡状況を見てファイルを選びます。**untracked のまま置いた同梱アセットは wheel に入りません**。今回はコミット済みだったので助かりましたが、これも build して中を開くまで分からない類のものです。

```powershell
Add-Type -AssemblyName System.IO.Compression.FileSystem
$zip = [System.IO.Compression.ZipFile]::OpenRead("dist\docsweep-0.4.0-py3-none-any.whl")
$zip.Entries.FullName | Where-Object { $_ -match 'static|okf_profiles' } | Sort-Object
$zip.Dispose()
```

タグを打つ前にこれを 1 回やっていれば、失敗は起きていませんでした。

## 4 つに共通していたこと

並べてみると、全部同じ形をしています。

1. 依存の上限 → **CI の解決結果**を見ないと分からない
2. OS 差 → **Linux で動かさないと**分からない
3. CSP → **ブラウザで描画させないと**分からない
4. wheel → **build して中を開かないと**分からない

![どの工程で何が初めて分かるか](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-11_release-pitfalls_fig2.png)

上の図は、走る場所ごとに「そこで初めて分かること」と「今回そこで出た穴」を並べたものです。手元の pytest の行だけ、出た穴の欄が空になります。

手元の `pytest -q` はこの 4 つをどれも通りません。緑だったのは事実ですが、緑が保証していたのは「Windows の、この依存構成の、Python プロセス内での振る舞い」だけでした。

もっと痛かったのは、このうち 2 つは**すでに自分で言語化してあった**ことです。リリース用の手順をまとめた自分のドキュメントには、「CI が緑でなければタグを打たない」も「ローカルで build が通ること」も書いてありました。今回はそれを起動せず、その場で手書きしたチェックリストで進めた。手書きのリストは、書いた本人がその瞬間に思いつく範囲しか持てません。

蓄積したチェックリストを持っているのに使わないのは、持っていないのと同じでした。ここがいちばんの反省点です。

## 次からこうする

- リリース作業の 1 行目を「直近の CI が緑か」の確認にする（`gh run list --limit 1 --json conclusion`）
- タグを打つ前に、ローカルで `python -m build` を通し、**wheel を展開して中身を見る**
- UI の受入項目は「〜が画面に見える」で書く。「〜を読み込めている」で書かない
- 依存の下限を上げたら、上限を切るかどうかを同時に決める
- 蓄積済みの手順があるなら、それを起動する。その場で書き直さない

## docsweep はこんなときに刺さります

- AI に任せた作業の記録が溜まる一方で、どれが終わったのか分からなくなっている人
- 古くなった計画書が「まだ生きているつもり」で残り続けて、判断のノイズになっている人
- 複数のリポジトリを並行で回していて、今日どれに手を付けるかを毎朝迷っている人
- 親子に分けた計画を「実装完了」と「本当に終わった」を混ぜずに締めたい人（v0.4.0 で `closeout-check` が入りました）

いずれかに心当たりがあれば、`pip install docsweep` で 1 分で試せます。設定ゼロで動きます。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=docsweep
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/docsweep
- PyPI: https://pypi.org/project/docsweep/

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## あわせて読みたい

- [AIが残すplan・bugfixを自動で片付けるdocsweepの始め方](https://qiita.com/ishizakahiroshi/items/17bf7a02efcb8c4718ec)：今回のツールの入口。どういう問題を解くために作ったかはこちらに書いています
- [自作 CLI が『安全策』で置いた backup ディレクトリで、非公開の md を公開リポに漏らしていた話](https://qiita.com/ishizakahiroshi/items/0f5b3e3d6f5406f56540)：同じく「良かれと思った実装が静かに壊す」型の失敗。v0.4.0 の秘密情報ガードはこの続きです
- [AIが毎日量産するplan_*.mdを腐らせない。docsweepをPyPIに初リリースした](https://zenn.dev/ishizakahiroshi/articles/20260703-docsweep-first-release)：最初のリリースの記録

## おわりに

「テストが緑だから出せる」と思っていた時期が、正直、今日までありました。

緑はうれしいのですが、それが何を保証しているのかは意外と狭い。今回いちばん効いたのは、ブラウザで実際に画面を開いて「何も描かれていない」を自分の目で見たことでした。あれを飛ばしていたら、機能停止したページをそのまま配っていたはずです。

小さく。出荷する経路で 1 回動かす、を習慣にしていきます。

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

