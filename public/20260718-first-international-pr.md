---
title: 初めて海外から Pull Request が届いた日、レビューで盛大にやらかした話
tags:
  - OSS
  - pullrequest
  - I18n
  - GitHub
  - コミュニケーション
private: false
updated_at: '2026-07-18T18:23:55+09:00'
id: 47e330f498b062b7288a
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![初めての海外 PR のヒーロー画像](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-18_first-international-pr_hero.png)

朝、GitHub の通知に見慣れない名前がいた。

Nguyễn Văn Huyên さん。ベトナムの方だ。自作の OSS 「many-ai-cli」に、Pull Request を送ってくれていた。

内容を開く前から、少し胸が熱くなった。海外からの、はじめての PR だ。

## 自作のツール（宣伝を先にすみません）

自作で [many-ai-cli](https://github.com/ishizakahiroshi/many-ai-cli) という AI コーディング CLI 用の Web ダッシュボードを作っています。複数の AI コーディング CLI を並列で走らせて、承認をブラウザ 1 タブに集約する道具です。スマホからでも承認できます。

同じ悩みを持っている方は、下記で入ります。

```bash
npm i -g many-ai-cli
```

リポジトリはこちらです（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/many-ai-cli

## 想像以上に丁寧な PR だった

diff を開いてまず息を呑みました。`+4855 / -20`。かなり大規模です。

内訳を見ていくと、単なる「翻訳追加してみました」の範疇を大きく超えていました。

- ベトナム語のマニュアル一式（`README.vi.md` / `CLAUDE.vi.md` / `docs/manual_*.vi.md` 全部）
- Hub UI 用のロケール `vi.json`（1256 キーの完全翻訳）
- コード側の分岐追加（`i18n.ts` の `navigator.language` から `vi` / `vi-VN` 自動判定、`voice.ts` の音声認識ロケール `vi-VN`、`git-view.ts` の commit message 言語、`settings.ts` の docs リンク、`user-prefs.ts` の wake word）
- README ja/en への 1 行のクロスリンク追加（既存構造を尊重）

しかも翻訳の質が高い。声調記号（ダイアクリティカル）が完璧で、機械翻訳では出ない自然な言い回しがあちこちに見えます。手作業だ、これは。

コードのほうも、たとえば `settings.ts` で `_summaryIsJa` の意味論を `!== 'en'` から `=== 'ja'` に直してあります。Vietnamese を追加するときに必要な、地味だけど大事な修正です。気づいて直してくれている。

```ts
// Before: 「英語じゃなければ日本語」だと vi のときに日本語判定されてしまう
function _summaryIsJa(): boolean { return (window as any).__lang !== 'en'; }

// After: 「日本語のときだけ日本語」なら vi は英語ラベル側にきれいに流れる
function _summaryIsJa(): boolean { return (window as any).__lang === 'ja'; }
```

これはすごい。ちゃんと使い込んでから PR を出してくれた人だ。

## それで、私はやらかしました

そこから、レビューを始めました。

`en.json` と `vi.json` のキー数を比べます。

- en.json: 1289 キー
- vi.json: 1299 キー

差分がある。33 キーが `vi.json` に足りない、43 キーが余分。しかも足りない 33 キーは全部 `bug_report_*` の系列。少し前に `bug_report/preview` API を追加したときに増えたやつです。

なるほど。PR のベースが少し古いんだな。rebase してもらって、`bug_report_*` を翻訳してもらう必要がある。よし、丁寧に伝えよう。

私は、彼への感謝を込めたベトナム語のコメントを書きました。冒頭は「初めての海外 PR、感激しました 🎉」。翻訳品質・コード配慮・既存構造の尊重、順番に褒めていきます。

そして最後に、こう書き足しました。

> 「ちょっとだけ同期がずれていて、`bug_report_*` の 33 キーが足りない、古い 43 キーが余分。rebase と 33 キーの翻訳追加は、こちらで対応します。あなたは何もしなくていい。ゆっくり merge を待っていてください 😄」

投稿ボタンを押しました。清々しい気持ちだった。

この 5 分後、私は自分の勘違いに気付きます。

## 「main」と「develop」

`en.json` の 1289 キー。これは develop ブランチのものでした。

彼の PR がターゲットにしているのは main。main の `en.json` は 1256 キーで、彼の `vi.json` と完全に一致しています。

つまり、彼の PR は最初から正しく完成していました。target も、キー数も、構造も、全部合っていた。

私は develop（まだ main に merge されていない内部の作業ブランチ）と比較して、「あなたの PR は古い、こちらで直してあげる」的なコメントを投げていたわけです。

冷や汗が出ました。

彼の視点から読み返してみます。「初めての海外からの PR、大きな時間をかけて丁寧に作った翻訳」に対して、maintainer から返ってきたのは「あなたのは古い、こちらで補完します、あとは merge を待ってて」というコメントです。

自分が逆の立場だったら、しばらく画面を閉じたくなる。「もう二度と PR 出さない」と思うかもしれない。

## 訂正コメントを書き直した

訂正コメントは、また同じくベトナム語で、正直に書きました。

- 事実確認: PR は正しく main を target している、`vi.json` は `en.json` と 100% 一致（1256 キー）
- 誤りの内訳: 内部の develop ブランチと比較を間違えた
- 謝罪: 「あなたの PR が未完成で、こちらの介入が必要である」と匂わせた言い方をしたこと
- 決定: PR は 100% あなたが出した形のまま merge する

そしてそのまま merge しました。彼のコミット `0e15d45 docs(i18n): add Vietnamese manuals and Hub UI locale` は、彼の名義で main の git log に残っています。

## 器がそもそもおかしかった

ここまでが今回の失敗談ですが、もう 1 つ、これは彼とは関係なく大きな気付きがありました。

彼の PR、`index.html` の変更が 1 行あります。

```html
<option value="ja">日本語</option>
<option value="en">English</option>
+<option value="vi">Tiếng Việt</option>
```

そして `settings.ts` `i18n.ts` `voice.ts` `git-view.ts` `user-prefs.ts` の 5 ファイルにも、`ja` / `en` の 2 分岐を `ja` / `en` / `vi` の 3 分岐にする修正が入っています。

つまり、「言語を 1 つ増やす」ために、HTML と TS 5 ファイルを手で編集する必要がある設計だった。

これは、設計側の問題です。本来は `web/src/i18n/` に `<code>.json` を置いたら勝手に検出されて `<select>` に追加され、コード側も分岐を書き足さなくていい、そういう auto-discovery であるべきです。

たとえばこういう形。

```
web/src/i18n/
├─ en.json  { "__meta": { "display_name": "English",   "bcp47": "en-US" }, ... }
├─ ja.json  { "__meta": { "display_name": "日本語",     "bcp47": "ja-JP" }, ... }
└─ vi.json  { "__meta": { "display_name": "Tiếng Việt", "bcp47": "vi-VN" }, ... }
```

server 側で glob して `[{"code":"vi","name":"Tiếng Việt","bcp47":"vi-VN"}, ...]` を返し、`<select>` を動的生成、`voice.ts` などは `__meta.bcp47` を読むだけ。次に韓国語やタイ語の PR が来ても、JSON を置くだけで済みます。

彼は「既存のパターンを尊重して」、あるパターンに素直に従って PR を出してくれただけです。器の側が古い設計を強要してしまっている。

これは、次のリリースまでにこちらでリファクタする宿題として持ち帰りました。

## 学んだこと

書き手として今回残しておきたいのは 3 つ。

**レビューでキー数を比べるときは、まず PR の target ブランチと比較する。** 自分の頭の中の「最新」は、たいてい develop や作業ブランチにあって、target ではない。target を明示的に指定して diff / grep する。

**「手伝います」は、貢献者の完成品を未完成扱いにする言い方かもしれない。** 「必要なら手伝います」ではなく、「あなたの PR は完成しています」を先に伝える。追加作業が本当に必要でも、それは merge した後の別の話にする。

**貢献者に「不足がある」ように見える設計は、たいてい maintainer の宿題。** 言語を 1 つ増やすのに 6 ファイル手で編集させる設計は、貢献者に負担を押し付けている。次の設計に直すのは、こちらの責任。

## many-ai-cli はこんなときに刺さります

- Claude Code / Codex / Copilot / Cursor / Grok を同時に走らせて、状態を 1 画面で監視したい人
- 各 CLI の承認待ちをまとめて捌きたい、ターミナルの往復を消したい人
- 離席中もスマホから承認したい人
- 今回のマージで、Web UI がベトナム語表示にも対応しました（次のリリースで載ります）

いずれかに心当たりがあれば、`npm i -g many-ai-cli` で 1 分で試せます。設定ゼロで動きます。

- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/many-ai-cli
- npm: https://www.npmjs.com/package/many-ai-cli

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## おわりに

Nguyễn Văn Huyên さん、ありがとうございます。

私のやらかしは正直恥ずかしいけれど、記事にする価値はあると思っています。同じことで悩んでいる maintainer が、この記事で「あ、私だけじゃないんだ」と思ってくれたら、それだけで十分です。

そして、次のリリースでは、ベトナム語対応がちゃんと入ります。器も直します。それが、彼の PR に対する私からの返礼です。

小さく。丁寧に、次の一歩を積んでいきます。

---

※ ヘッダー画像は AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
