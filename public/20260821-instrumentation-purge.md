---
title: "調査用のデバッグコードをリリースに混ぜない設計（台帳 + build tag + 成果物検査）"
tags:
  - Go
  - TypeScript
  - 設計
  - staticcheck
  - OSS
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-21_instrumentation-purge_hero.png)

![記事の要約（インフォグラフィック）](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-21_instrumentation-purge_infographic.png)

上の 1 枚に記事全体が入っています。台帳が「登録されているか」を、build tag が「同梱するか」を、成果物の grep が「本当に入っていないか」を、それぞれ別々に受け持つ形です。

結論から書きます。調査用に仕込んだデバッグコードは、リリース前に手で消すのをやめました。代わりに次の 3 つでやっています。

- 仕込んだら台帳（`instrumentation.json`）に登録する。登録の無い観測コードは CI が止める
- 同梱は build tag のオプトインにする。何もしなければリリース成果物には入らない
- 最後に成果物を grep して、本当に入っていないかを機械で確かめる

きっかけは、承認バーのバグを追うために `approval.ts` へ 37 行の記録用コードを仕込んだことでした。原因が確定したら消す。また似た症状が出たら、また書く。この往復が面倒で、しかも危ないと気づいたところから始まります。

## many-ai-cli について

自作で [many-ai-cli](https://github.com/ishizakahiroshi/many-ai-cli) という AI ツール / Web ダッシュボードを作っています。複数の AI コーディング CLI を並列で走らせ、承認をブラウザ 1 タブに集約。スマホからでも。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/many-ai-cli

同じ悩みを持っている方は、下記で入ります。

```powershell
pnpm add -g many-ai-cli
```

入れたら 1 回だけ `many-ai-cli setup` を実行します。PATH が通っていなくても `pnpm exec many-ai-cli setup` で作れます。

起動後の見え方は OS で違います。

Windows はデスクトップに「MANY-AI-CLI」のショートカットが 1 個できます。ダブルクリックするとタスクトレイに常駐し、トレイメニューの「Hub を開く」でブラウザが開きます。止めるときは同じメニューの「Hub を停止」です。

macOS と Linux は「Many AI Hub Start」「Many AI Hub Stop」の 2 個ができます（`.command` / `.desktop`）。Start を押すと黒いコンソールウィンドウとブラウザが一緒に開きますが、そのウィンドウが Hub の実体プロセスなので、閉じずに最小化してください。Linux の GNOME では `.desktop` を初回だけ右クリックして「起動を許可」を選ぶ必要があります。

ターミナルから直接起動するなら `many-ai-cli serve --open` が全 OS 共通です。開いたら左下の「+ 新しいセッション」から使う CLI を選ぶだけです。

なお実機で動作検証できているのは Windows のローカル Hub と統合ランチャーまでで、ネイティブ Linux とネイティブ macOS は十分に検証できていません。README にもそう書いています。

前回の記事: [Claude も Codex も、承認を 1 画面で。many-ai-cli v0.7.0 を出しました](https://qiita.com/ishizakahiroshi/items/fc31553bf9427109d4dc)。今回の話は、その v0.7.0 で承認検出を追いかけるために仕込んだ観測コードの後始末です。

## 台帳はもうあった。止められていたのは「消し忘れ」だけだった

観測コードの管理そのものは、前からやっていました。`instrumentation.json` という台帳があって、検査スクリプトが CI で 3 つのことを見ています。

- 台帳に無い観測コードが生えていないか
- `status=active` のものが期限（`due`）を過ぎていないか
- `status=removed` にしたのに実ファイルが残っていないか

これで「消し忘れてリリースする」はもう起きません。実際、台帳に載せる運用にしてからその事故はゼロです。

問題は別のところにありました。台帳が `files`（その観測専用のファイル）と `sharedFiles`（間借りしている共有ファイル）を分けているのですが、この分け方がそのまま作業量の分布になっていたのです。

```
専用ファイル 5 本 413 行   ファイルごと消せる
共有ファイル側            消して、また書き直す
  web/src/app/approval.ts       37 行
  web/src/app/terminal.ts       記録点 1 箇所
  internal/hub/input_gate.go    呼び出し 3 箇所
  internal/wrapper/wrapper.go   呼び出し 2 箇所
  internal/hub/server.go        ルート登録 1 行
```

413 行のほうは `rm` で終わります。面倒なのは 37 行のほうでした。

## 「デバッグ用のフォルダを作る」では解決しなかった

最初に考えたのは、素朴に「debug フォルダを作ってそこに全部入れる」でした。フォルダごと消せば終わり、という発想です。

これは半分しか効きません。`approval.ts` の 37 行は、検出ロジックの中に食い込んでいるからです。

```ts
const debugOn = isApprovalDebugEnabled();
const liveOptsForDebug = debugOn ? options.slice() : null;
const gateForDebug = debugOn ? [...].filter(Boolean).join('+') : '';
let bufActionForDebug = 'keep';

if (bufOpts.length > options.length || ...) {
  options.length = 0;
  options.push(...bufOpts);
  bufActionForDebug = 'replace';   // 観測のためだけの代入
}
```

`bufActionForDebug` は「何が起きたか」を記録するためだけの変数です。これを別フォルダに移すことはできません。移せるのは、記録を受け取って画面に出す側だけです。

ここで方針を変えました。置き場所ではなく、呼び出し点の形を変える。

## 記録点は 1 行にして、意味づけは受け取る側でやる

共有ファイルに残すのは、これだけにしました。

```ts
const bufProbe = probeScope('approval.buf', () => ({
  sessionId: id,
  site: 'chunk',
  gate: gateReason(),
  liveOptions: options.slice(),
}));

// ...本来のロジックだけが残る...

bufProbe?.emit(() => ({
  tailLines: bufferTail.length,
  bufOptions: bufOpts,
  afterOptions: options.slice(),
}));
```

`probeScope` は、受け取る側（sink）が登録されていなければ即 `null` を返して、渡したラムダを 1 度も評価しません。だから配列コピーもゲート理由の文字列組み立ても走りません。

そして `replace` だったのか `fill1` だったのか `keep` だったのかの判定は、受け取る側が「前」と「後」の差分から計算します。呼び出し点に観測用の状態機械を置かない、という一点だけ守れば、37 行は 12 行になりました。

この「意味づけは受け取る側でやる」は後で効いてきます。表示都合の話（変化が無かった回はパネルに出さない、など）も全部そちら側に寄せられるからです。

## 過去 3 件の事故は、消し忘れではなくゲートの失敗だった

ここまでは整理の話です。設計を変えることになったのは、台帳のコメント欄を読み返したときでした。そこには過去 3 件の事故が書いてあります。

- 承認回答の本文をそのまま記録するエンドポイントを出荷した
- 打鍵バイトを無マスク・0644・無制限で永続化するトレースを出荷した
- 既定 ON のままリリース直前まで残っていた計測を出荷した

3 件とも「消し忘れ」ではありません。全部「ゲートの失敗」です。既定を OFF にし忘れた、ゲート自体を書き忘れた、ゲートの外でバイト列を保存していた。

ランタイムのゲートは実行時の判断なので、間違えられます。台帳と期限は「消したか」を守りますが、「入ったまま出たときに何も起きないか」は守っていませんでした。

## 前提が 1 つ間違っていた

そこで「ビルド時に落とす」を検討したのですが、最初は却下しました。理由はこうです。追いかけている症状は日常運用中に出るもので、デバッグビルドに切り替えて再現を待つのは現実的ではない。だからランタイムゲートしかない。

もっともらしいのですが、確かめずに書いていました。実際に動いているプロセスを見たら、こうでした。

```
Path : D:\dev\github\public\many-ai-cli\dist\many-ai-cli.exe
```

npm 経由で入れたリリース版ではなく、自分の `make build` の成果物でした。作者が日常で使っているのは最初から開発ビルドです。だとすると、開発ビルドを日常使いにするコストはゼロです。

前提が崩れたので、方針を反転しました。**build tag が「同梱するか」を決め、ランタイムゲートが「収集するか」を決める。** ゲートは消しません。その上にもう 1 枚重ねます。

![同梱・収集・検査の 3 層](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-08-21_instrumentation-purge_fig1.png)

上の図のとおり、ソースツリーには観測コードが残っていてよくて、分岐するのはビルドのところです。`-tags maidebug` と `MAI_DEBUG=1` を渡したときだけ観測が入り、リリースは goreleaser 経由で何も渡さないので入りません。付け忘れたらクリーンな側に倒れる、という向きが大事です。

Go 側は build tag で実装と空実装の 2 枚に分けました（[Build constraints](https://pkg.go.dev/cmd/go#hdr-Build_constraints)）。

```go
//go:build maidebug

func (s *Server) probe(channel string, args ...any) {
	if f := probeSinks[channel]; f != nil {
		f(s, args...)
	}
}
```

```go
//go:build !maidebug

func (s *Server) probe(channel string, args ...any) {}
```

記録点は両方のビルドでコンパイルされます。既定側は空関数なので、呼び出しごとコンパイラが畳みます。ただし「何も実行されない」は保証できても「引数の評価コストがゼロになる」までは保証できないので、渡す値は整数や短い文字列に限る、というルールを添えました。

Web 側は esbuild を `bundle: false` で使っているので tree shaking が効きません。出力から外すには entry から外すしかないので、既定では `web/src/debug/` を entry から除外して、空の `index.js` を書き出しています。

書いてみて気持ちよかったのは、この方式にすると「リリース前に撤去する」という工程そのものが消えることでした。ソースに残っていても成果物に入らないなら、消すかどうかは「もう要らないか」という片付けの判断に格下げされます。

## 成果物を grep する、で踏みかけた罠

最後の 1 枚が成果物検査です。ビルドしたバイナリを観測用の文字列で grep して 0 件を確認する、というだけの話に見えました。

素朴に channel 名（`approval.buf` など）で grep しようとして、途中で気づきました。**それは誤検出します。** 記録点はリリースビルドにも残るので、`probe("input.gate", ...)` の文字列リテラルが最適化で落ちるかどうかはコンパイラ任せです。落ちなければ、観測が入っていないのに赤くなります。

なので検査の的を「受け取る側にしか存在しない文字列」に絞りました。台帳に `artifactNeedles` を足して、各エントリが「成果物に絶対現れてはいけない文字列」を宣言します。

```json
"artifactNeedles": ["/api/debug/mobile-view", "debug mobile view sample"]
```

観測用エンドポイントのパス、受け取る側が出すログメッセージ名、パネルの UI 文字列。この 3 種類から選びます。呼び出し点にも出てくる名前は入れません。

## 消したものを、あとで戻せるようにする

もう 1 つ欲しかったのが再導入です。同じ症状が再発したとき、前回どこに何を書いたかを思い出すのが毎回つらい。

台帳に「撤去したコミットの SHA」を持たせるのが素直ですが、それはやめました。手で維持しないと壊れる値をリポジトリに置きたくないからです。代わりに、履歴から機械的に探します。

`instrumentation.json` の履歴を新しい順にたどって、対象の id が「そのコミットで `removed`、親コミットで `removed` でない」最初の 1 件を探す。それが撤去コミットです。実データで 1 回回して確認しました。

そのうえで、逆 diff を path 限定で当てます。ここが肝で、単純な `git revert` ではだめでした。実際の撤去コミットを見たら、観測コードの削除と一緒に CHANGELOG の 58 行が入っていたからです。全部戻すと CHANGELOG まで巻き戻ります。

```
git diff <撤去コミット> <その親> -- <files と sharedFiles> | git apply -3
```

台帳が撤去直前に持っていたファイル一覧だけに絞れば、混ざった変更を巻き添えにしません。

ついでに 1 つ穴も見つかりました。その撤去コミットは `*_test.go` も一緒に消していたのに、台帳の `files` には `.go` 側しか載っていませんでした。検査スクリプトの除外リストに `_test.go$` があるのは「検査対象にしない」という意味で、「台帳に書かない」ではないのに、書く側がそこを混同していたわけです。付随するテストも `files` に書く、というルールを足しました。

## 実装してから気づいたこと 2 つ

設計としては上で終わりなのですが、実際に入れてみて 2 つ踏みました。両方とも自分の手癖の話です。

1 つ目。将来使うかもしれないと思って、遅延評価版の `probeLazy` を先に置きました。呼び出し元が 1 つも無いので、staticcheck の U1000 が拾って CI が赤くなりました（[U1000: Functions and methods that are never used](https://staticcheck.dev/docs/checks/#U1000)）。使う当ての無い API を先に置かない、という当たり前の話です。必要になったときに足せばいい。

2 つ目はもっと間抜けです。検査スクリプトに足した正規表現が、なぜか正しいファイルまで「タグが無い」と判定しました。

```js
const MAIDEBUG_TAG_RE = /^\/\/go:build\s+maidebug\b/m;
```

これで合っているはずなのに落ちる。同じ正規表現を単体で当てると通る。30 分ほど溶かして、バイトを見てようやく分かりました。

```
g  \b   /   m   ;
```

末尾の `\b` が単語境界ではなく、**バックスペース文字そのもの（0x08）** として書き込まれていました。編集の途中でエスケープが 1 段外れていたのです。`.source` を出力しても画面上は `\b` に見えるので、目視では絶対に見つかりません。

これは効いた。正規表現やエスケープを含む編集の後は、目で見て納得しないでバイトで確認する。同じ手順で書いた全ファイルを走査したら、混入はこの 1 箇所だけでした。

## あわせて読みたい

- [Agent Plugins を自作ツールに取り込むか検討して、見送りました。配布モデルが逆向きだった](https://qiita.com/ishizakahiroshi/items/000383ea267490d25886) 。今回と同じく「採らなかった理由」を残す話です。見送りも設計判断として台帳に置いています。
- [対応 AI CLI を増やす前に、自分のツールの利用者数を測った方がいい](https://qiita.com/ishizakahiroshi/items/f491815a42b157f15932) 。思い込みの前提を実測で崩す話。今回の「日常使いは自前ビルドだった」と同じ形の気づきです。
- [OpenRouter を many-ai-cli に載せるなら、独立プロバイダにはしない](https://qiita.com/ishizakahiroshi/items/4feda29aa885644c982b) 。機能を足すかどうかではなく、どの層に置くかを決めた回です。

## many-ai-cli はこんなときに刺さります

- Claude Code や Codex、Copilot、Cursor、Grok、opencode を同時に走らせていて、どれが止まっているか分からなくなる人
- 承認待ちのたびにターミナルを行き来していて、その往復を減らしたい人
- 離席中や外出先から、スマホで承認だけ返したい人
- デバッグ用の計測をリリースに混ぜたくない人（今回の仕組みは v0.8 系で入れています）

いずれかに心当たりがあれば、`pnpm add -g many-ai-cli` と `many-ai-cli setup` の 2 コマンドで試せます。設定ファイルを書く必要はありません。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=many-ai-cli
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/many-ai-cli
- npm: https://www.npmjs.com/package/many-ai-cli

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## おわりに

観測コードを消す作業が減ったこと自体より、「消し忘れを人の注意力で守らない形にできた」ほうが嬉しかったです。ゲートは間違えられるけれど、成果物の grep は間違えません。

まだ全部は終わっていません。成果物検査を CI に配線したところまでで、それが実際に走るのは次のリリースのときです。走ったかどうかは、そのとき確かめます。

小さく。仕込んだものは台帳に載せる、を続けていきます。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
