---
title: "Claude Desktop の in-app browser を調べて、自作 OSS に「実装しない」と決めるまで"
tags:
  - ClaudeCode
  - AI
  - OSS
  - 個人開発
  - 設計
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![ヒーロー画像](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260711-not-adopting-in-app-browser_hero.png)

![機能分解による実装判断](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260711-not-adopting-in-app-browser_infographic.png)

7 月のある朝、X のタイムラインに Claude Desktop の新機能が流れてきました。ターミナルの隣にブラウザペインが開いて、AI が自分で画面を確認しながらコードを直していく。in-app browser。正直、いいなと思いました。

自作の OSS に [many-ai-cli](https://github.com/ishizakahiroshi/many-ai-cli) という、複数の AI コーディング CLI（Claude Code / Codex CLI など）を並列で動かして、承認操作と進捗を 1 画面の Web ダッシュボードで見る Go 製ツールがあります。これにも欲しい。そう思って半日調査して、出した結論は「実装しない」でした。

この記事は、うらやましい機能を前にして「作らない」を選ぶまでの調査記録です。機能追加の話より、こっちのほうが設計判断としては難しかった。

## まず本家の機能を分解してみる

公式ドキュメントを読むと、Claude Desktop の browser pane はこういう仕様でした。

- Ctrl+Shift+B でターミナルの隣にブラウザペインが開く
- AI がページの読み取り・クリック・ナビゲーション・スクリーンショットまでやる
- 個人のブラウザとは完全に別のクリーンプロファイルで動く（ログイン情報も履歴もなし）
- 外部サイトへの書き込み操作は毎回 safety classifier の検査が入る。購入や CAPTCHA は AI 代理不可
- Desktop アプリ専用。ターミナル版の Claude Code CLI にはこの機能はない

ここまでは「埋め込みブラウザ」の説明です。でも読み進めると、本体は別のところにありました。

`.claude/launch.json` という設定ファイルに dev サーバーの起動方法（`npm run dev` とポート番号）を書いておくと、AI がファイルを編集した後、自動で dev サーバーをブラウザペインで開いて、スクリーンショットを撮り、コンソールエラーを読み、自分で直す。この自己検証ループが本体で、ブラウザペインはその画面に過ぎない。

「できました！」と言われて開いたら真っ白、エラーをコピペして貼り返す。あの往復が消える機能です。そりゃ欲しい。

## 嬉しさを 2 つの層に分けたら、話が変わった

欲しい欲しいと言っていても設計にならないので、この機能の「嬉しさ」を分解しました。すると 2 層に割れます。

1 つ目は操作層。AI が自分の目でブラウザを確認して、自分で直す層。自己検証ループはここに属します。

2 つ目は表示層。人間が「AI の作った画面」をウィンドウを切り替えずに見られる層。ブラウザペインの見た目はこっち。

![嬉しさの 2 層分解と、それぞれの担い手](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260711-not-adopting-in-app-browser_fig1.png)

図にすると、操作層の担い手は AI 側のツール、表示層の担い手はダッシュボードで、そもそも担当が別物だと分かります。この分解をした時点で、自作ツールに足すべきものは思ったよりずっと小さいのではという疑いが出てきました。

## 操作層は、ラッパーが足すものが何もなかった

many-ai-cli は PTY ラッパーです。各 AI CLI の入出力を包んで観察はできますが、CLI のツールループの中には割り込めません。「ブラウザをクリックする」という能力を外から注入することは構造的にできない。

一度は本格的な構成も検討しました。Hub が隔離した Chromium をリモートデバッグポート付きで起動して、各 CLI には Playwright MCP を `--cdp-endpoint` で同じブラウザに接続させ、ダッシュボードには CDP のスクリーンキャストをミラー表示する案です。調べた限り、技術的には全部つながります。Playwright MCP は外部ブラウザ接続も隔離プロファイルも公式サポートしていました。

でも書き出してみて気づきます。この構成でブラウザを操作しているのは Playwright MCP であって、ラッパーは 1 ミリも操作に関与していない。ユーザーが各 CLI に Playwright MCP を足せば、自己検証ループは今日の時点でもう手に入るんです。ラッパーが足せるのは「AI が操作している様子を眺める画面」だけ。それは観賞価値であって、必要性ではない。

操作層は却下。というより、最初からラッパーの土俵にありませんでした。

## 表示層は「クリックで開けばいい」に勝てない

残った表示層で、ダッシュボードに Preview タブを作って iframe で `localhost:3000` を映す案を考えました。

これも書き出すと弱い。ローカルで使っているなら、ブラウザで見たければ自分でタブを開けばいいだけです。iframe は新しいタブより画面が狭く、DevTools が使いにくく、X-Frame-Options を返すフレームワークだと映りもしない。埋め込むこと自体に価値がない。

そこで案を減量しました。PTY 出力から dev サーバーの URL を検出して、セッションカードに「:3000 で起動中」のバッジとリンクだけ出す案。並列で 4 セッション走らせているとポートが 3000 / 5173 / 8080 と散らばって「どのセッションのやつだっけ」とログを遡ることがあるので、これなら小さく効きそうに見えました。

## 最後に残った案も、トリガーが実在しなかった

バッジ案の実装ポイントまで洗い出した後で、根本の問いにぶつかります。その dev サーバーの URL、いつ PTY に流れてくるんだっけ。

現実のワークフローを 3 パターンに分けると、全部空振りでした。

- 一番多いパターンでは、dev サーバーはユーザー自身が別ターミナルで常駐させています。AI には「この画面直して」と頼むだけで、HMR が勝手に反映する。この場合、URL はラッパーの監視する PTY に一切流れません。検出するものが存在しない
- AI が自分でサーバーを立てる場合も、ツール実行の出力は画面上で折りたたまれていて、URL がスクレイプ面に確実に出る保証がない
- 素の HTML に至ってはサーバー自体が不要で、file:// で開けば終わり

![3 つの動線すべてで検出器が空振りする図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260711-not-adopting-in-app-browser_fig2.png)

並べてみると、検出器を仕掛けようにも、獲物が通る道がそもそもない状態です。作っても発火しない機能になる。

ここで本家の設計に戻ると、Claude Desktop がわざわざ launch.json という仕組みを持っている理由が見えてきます。あれは「検出」を最初から諦めて、アプリ自身が dev サーバーの起動係になる設計です。自分で起動するから、いつどのポートで動くかを確実に知っている。

同じことを many-ai-cli でやるなら、セッションごとの起動設定とサーバー起動ボタンとプロセス管理を抱えることになります。それは承認と監視のハブが、タスクランナーや IDE に化けるということで、基本コンセプトからの逸脱です。本家と同じ問題に、別の製品形態のまま答えようとするとこうなる。

## 現状の機能で、もう足りていた

調査の最後に、いま既にできることを棚卸ししました。

many-ai-cli のダッシュボードでは、ターミナル出力中のファイルパスがクリックできて、ポップアップから「既定のアプリで開く」を選ぶと OS の関連付けで開きます。AI が「report.html を作りました」と出力したら、パスをクリックして 2 クリックでブラウザ表示まで届く。素の HTML はこれで完結していました。

ローカルサーバー派は自分で立てて自分で開く。今まで通りで誰も困っていない。AI に画面を自己検証させたい人は、各 CLI に Playwright MCP を足せばラッパー越しでも普通に動く。ラッパーの仕事は、それを邪魔しないことだけでした。

## 学んだこと

- うらやましい機能は、まず嬉しさを層に分解する。層ごとに「これは誰の責務か」を問うと、自分が作るべき部分は案外小さい。今回はゼロだった
- 本家の設計の妙な出っ張り（launch.json）には理由がある。そこを読むと、自分の製品形態との相性が判定できる
- 「実装しない」で終わる調査は失敗ではない。機能を増やさない根拠が言語化されて残るので、次に同じ誘惑が来たとき秒で判断できる
- 現状機能の棚卸しは調査の最後にやる価値がある。欲しかったものの大半が、既にあった

新機能のニュースを見るたび、たぶんまた「うちにも欲しい」と思います。そのときに層分解の物差しを 1 本持っておく。今回の収穫はコードではなくて、それでした。

## many-ai-cli を試すなら

宣伝を兼ねて、ツール本体の入り口だけ置いておきます。

**many-ai-cli** は、複数の AI コーディング CLI を並列で動かしたときに、承認待ちを取りこぼさないためのローカル Hub です。Claude Code / Codex CLI / GitHub Copilot CLI / Cursor Agent CLI / Grok Build CLI を PTY で包み、ブラウザ 1 画面で承認と進捗を見ます。Hub 自体は `127.0.0.1` 固定で、外には出しません。

- GitHub: https://github.com/ishizakahiroshi/many-ai-cli
- 製品の全体像（機能紹介）: [Claude も Codex も Copilot も Cursor も、ぜんぶ 1 画面で承認する（many-ai-cli v0.3.2）](https://note.com/ishizakahiroshi/n/nfaf2ab0bdb9c)

### インストール

おすすめは npm レジストリ経由です。Windows でブラウザから exe を拾う導線を避けられるので、入れやすいです。

```powershell
pnpm add -g many-ai-cli
```

`pnpm` が無い場合は、同じレジストリからどれでも構いません。

```powershell
bun install -g many-ai-cli
# または
npm install -g many-ai-cli
```

パッケージマネージャ別:

```powershell
# Windows (winget)
winget install ishizakahiroshi.many-ai-cli
```

```bash
# macOS (Homebrew)
brew install --cask ishizakahiroshi/tap/many-ai-cli && many-ai-cli setup
```

```bash
# Linux は GitHub Releases の deb / rpm / zip
# https://github.com/ishizakahiroshi/many-ai-cli/releases/latest
```

入れたあとは `many-ai-cli setup` でショートカットを作り、Hub を起動して、いつもの `claude` / `codex` / `copilot` / `cursor-agent` を打ち始めるだけです。詳細とトラブルシュートは README にまとめてあります。

複数 CLI の見張りに疲れているなら、たぶん刺さると思います。issue も歓迎です。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
