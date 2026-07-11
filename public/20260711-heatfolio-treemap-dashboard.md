---
title: 口座連携なしで、保有資産をtreemapで俯瞰するローカル完結ダッシュボードの設計
tags:
  - JavaScript
  - YahooFinance
  - 個人開発
  - 資産管理
  - tailscale
private: false
updated_at: '2026-07-11T15:06:18+09:00'
id: b5da260733e416085421
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![ヒーロー](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/heatfolio-hero.png)

上の画面が heatfolio です。tile の面積が評価額、色が前日比。tile の中には「銘柄名 (証券コード)」「評価額」「数量」「騰落率」が並び、右上には Yahoo Finance と TradingView へ 1 クリックで飛べるバッジがついています。ダッシュボードで「あれ、この銘柄なんで下がってる？」と気になった瞬間に、そのまま外部サイトで実チャートを確認できるようにしています（画面の Apple 5 株は demo・後段で解説する USD→JPY 換算の実演例）。

![infographic](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/%E8%87%AA%E4%BD%9C%E8%B3%87%E7%94%A3%E7%AE%A1%E7%90%86%E3%83%84%E3%83%BC%E3%83%AB%E3%81%AE%E6%A6%82%E8%A6%81.png)

## 保有資産の全体感を、口座を明け渡さずに掴みたかった

証券口座と API 連携するタイプの資産管理ツールは、確かに便利です。ただ、その便利さと引き換えに、保有情報や場合によっては ID/PW をどこかのサービスに預けることになる。この「信頼コスト」が個人的にずっと引っかかっていました。

かといって Excel だと、増減は追えても「何にどれだけ寄せているのか」という全体感が入ってきません。数字の羅列は、意思決定に効かない。

heatfolio はその中間を狙って作った、単一 HTML の小さな自作ダッシュボードです。口座連携なし、パスワード不要、データはこの PC 内だけ。それでいて価格は日次で自動更新される。この記事では、選んだ設計判断と実装で面白かった 5 点を書き残します。

リポジトリはこちら（v0.1.0 が上がっています）:
https://github.com/ishizakahiroshi/heatfolio

## 既存の資産管理ツールに感じていた居心地の悪さ

証券会社の API 連携型は、初回に連携するときに「本当にこのサービスの中の人を全員信じていいのか」という問いを毎回スキップさせられます。しかも保有情報は個人の資産構成そのものなので、漏れた時のダメージが他のログイン情報と比較にならない。

決定的に効いたのは、国内で最大手の家計簿・資産管理サービスであるマネーフォワードで、過去に情報流出のインシデントが発生していることでした。特定のサービスを責めたい訳ではなくて、これはむしろ逆の話です。あれだけの規模で、あれだけ真面目にセキュリティに投資している会社でも、脆弱性・委託先・従業員アカウント経由で漏れは起こり得る。だとすると、より小さい会社の類似サービスや、無料の SaaS で「大丈夫ですよ」と言われても、それを鵜呑みにできる根拠はどこにも無い。連携先を信じる/信じないの問題ではなく、「そもそも家計データを人に預ける前提を採用するか」を選び直す話だと思うようになりました。

一方で、口座画面を毎日開いて眺めるのはコストが高い。手元にサマリが欲しい。だから何か作りたい、と思い続けていました。

ただ、いざ作ろうとすると Excel っぽくなっていく。表とグラフが並ぶ、あの見え方です。あれは「今日、どこにお金が寄っているか」を掴むのには向いていません。1 行 1 銘柄の表は、頭の中で総和を組み直す作業を毎回強いてくる。

面積で「量」を、色で「変化」を、同時に見たい。treemap の見え方が、意思決定には効く。ここは譲れないと思いました。

## 選んだ設計判断: 何を捨てるか

面白いのはここからで、「口座 API 連携」と「treemap で俯瞰する」の両方を同時に取ろうとすると、既存の SaaS ツールに寄せるしかなくなります。そこで、片方を明示的に捨てることにしました。

捨てたのは口座連携です。

- 数量は自分で 1 度だけ入力する（JSON を手で書くか、画面から編集する）
- 価格は Yahoo chart API で日次に自動取得する
- 保有データは PC ローカルの JSON に置く。クラウド DB は使わない
- 外部公開もしない。閲覧は自宅 PC のブラウザか、Tailscale で自分の端末からだけ

「やらないこと」を最初に固めたので、以降の実装判断がぶれずに済みました。ビルドツールも入れませんでした。`index.html` は 1 ファイル、開けばそのまま動きます。

## 中間解の全体像

構成はこの 3 層だけです。

- 静的 UI: `index.html`（treemap の描画、保有と価格履歴を読んで評価額を計算する）
- ローカル配信＋保存 API: `scripts/serve-local.pyw`（Python の http.server + `POST /api/holdings` で編集内容を書き戻す）
- 日次バッチ: `scripts/fetch-prices.mjs`（Yahoo chart API から終値を取得し、`data/prices/history.json` に追記）

データはこの 2 ファイルに集約されています。

```
data/
├── holdings.json          # 保有（銘柄・数量・評価方法）を手入力
└── prices/history.json    # 価格履歴（日次バッチが自動追記）
```

流れは至って単純です。日次バッチが Yahoo から価格を取って `history.json` に追記する。`index.html` が両方の JSON を読んで面積と色を計算する。編集は画面から `POST /api/holdings` を叩けば、保存 API が原子的に書き戻す。それだけです。

以降、この構成のなかで面白いと自分で思っている実装ポイントを 5 つ書きます。

## 実装で面白い 5 点

### 1. 価格取得の一点集約: `fetchClose()` だけ差し替えれば別ソースへ移れる

`scripts/fetch-prices.mjs` の中で、外部 API を叩いているのは `fetchClose()` という関数 1 つだけです。呼び出し側は「シンボル」と「日付範囲」を渡すだけで、返り値は `{ date, close }` の配列。

```javascript
// scripts/fetch-prices.mjs（抜粋・イメージ）
async function fetchClose(symbol, fromDate, toDate) {
  const url = `https://query1.finance.yahoo.com/v8/finance/chart/${symbol}` +
              `?period1=${fromDate}&period2=${toDate}&interval=1d`;
  const res = await fetch(url);
  const json = await res.json();
  // Yahoo のレスポンス構造を { date, close } の配列に整形して返す
  return normalize(json);
}
```

Yahoo API が壊されたり、料金化されたときに、この関数の中身だけを別のソース（Google Finance、Stooq、証券会社のダウンロード CSV など）に差し替えれば、以降のパイプラインは何も変えなくて済む。これは「バラすと分かるが、まとめておくと後で助かる」種類の設計です。

USD 建て銘柄がある日は、この関数で `JPY=X`（Yahoo の USD/JPY）の「始値」も取っておきます（詳しくは 5. で書きます）。

### 2. `holdings.json` の原子置換保存: 書き途中で落ちても壊れない

`POST /api/holdings` の実装は、Python の `http.server` に 1 ハンドラ足すだけです。ただし「書き込み中に落ちて JSON が壊れる」は個人ツールでも普通に起きるので、原子置換で書きます。

```python
# scripts/serve-local.pyw（抜粋・イメージ）
tmp = holdings_path.with_suffix(".json.tmp")
tmp.write_text(json.dumps(new_data, ensure_ascii=False, indent=2), encoding="utf-8")
os.replace(tmp, holdings_path)   # 同一ボリューム内なら atomic
```

一時ファイルに全部書き切ってから `os.replace` で正本を差し替える。POSIX の rename が同一ボリュームでは atomic であることを利用しています。Windows でも `os.replace` は同じ意味で動きます。

これに加えて、書き込む前に JSON を検証しています。検証落ちや書き込み失敗で例外を出した場合、正本 `holdings.json` は 1 バイトも触られていません。「保存 API のバグで資産データが飛んだ」は避けたい事故なので、ここは強めに守っています。

### 3. Windows タスクスケジューラ + VBS ランチャーでノーウィンドウの日次バッチ

日次バッチは Windows のタスクスケジューラで平日 16:05 に叩いています。ここで素直に `node scripts/fetch-prices.mjs` を登録すると、実行のたびに黒いコンソール窓が一瞬ちらつく。地味に気が散ります。

対策は、間に VBS ランチャーを 1 枚挟むだけです。

```vbscript
' scripts/run-fetch.vbs（イメージ）
Set sh = CreateObject("WScript.Shell")
sh.Run "cmd /c node C:\path\to\fetch-prices.mjs", 0, True
' 第2引数 0 = ウィンドウ非表示、第3引数 True = 終了まで待つ
```

タスクスケジューラには `wscript.exe run-fetch.vbs` を登録する。VBS は同期実行で終了コードを親に返せるので、node の exit code がそのままタスクの結果として記録されます。`StartWhenAvailable` を有効にしておくと、PC が落ちて逃した回は次回起動時に自動で追いつきます。

### 4. Tailscale serve でスマホから tailnet 限定 HTTPS

閲覧はローカルサーバー `127.0.0.1:8080` で完結しますが、外出先のスマホからも見たい。そこは Tailscale の serve に頼っています。

```
tailscale serve --bg --https=8443 8080
```

これだけで tailnet 内から `https://<このマシン>.<tailnet>.ts.net:8443/` で見えるようになります。ポート 8443 を使っているのは、同じマシンで別ツールが 443 を使っているためです（tailscale serve は「公開マウント点」で衝突するので、443 の / を奪い合うと片方が消える）。

Tailscale が入っている自分の端末からしか届かず、認証は Tailscale 側が担う。SSL 証明書も自動で用意されます。個人用途で「外に穴を開けず、でも自分の端末では見える」を実現するのに、これほど楽な選択肢は他に思いつきません。

### 5. USD 建て銘柄はその日の USD/JPY 始値で円換算する `fxAt()`

保有には米国株が混ざるので、合計を円で出すには為替換算が要ります。ただ厳密にやり出すと止まらないので、「その日の USD/JPY 始値で円換算する」で割り切りました。ヒーロー画像の一番大きい Apple のタイルがちょうどこの例で、`AAPL 終値 × 5 株 × その日の USD/JPY 始値` を静かに計算しています。

```javascript
// index.html の valueAt() の中で（イメージ）
function valueAt(holding, date) {
  if (holding.currency === "USD") {
    const usdPrice = priceAt(holding.symbol, date);   // ドル建て価格
    const jpyRate  = fxAt(date);                       // その日の JPY=X 始値
    return holding.quantity * usdPrice * jpyRate;
  }
  return holding.quantity * priceAt(holding.symbol, date);
}
```

fetcher 側は、USD 建て保有が 1 つでもある日だけ `JPY=X` の始値を取りに行きます。個別株の終値はそのまま、為替だけ「その日の朝の値」で束ねる。厳密ではありませんが、意思決定用の見え方としては十分です。

`proxy`（DC の S&P500 積立など、実銘柄ではなく指数近似）は設計として為替を無視しています。ここで為替まで刻むと、指数近似の粗さに比べて overshoot するので割り切っています。

## 学んだこと

- 「便利さ」の代償として何を差し出しているかを、機能一覧に並べて眺めると設計判断は自然に固まる。信頼コストも仕様の一要素です
- 大手が起こしたインシデントは「その会社のミス」ではなく「その仕組み自体のリスク上限」を教えてくれる情報として読める。マネーフォワードの流出は、家計データを他社に預ける仕組みの限界を、業界一真面目な会社の名前で示してくれたと受け取っています
- 外部依存が 1 箇所しかない関数（この場合 `fetchClose()`）を意識して用意しておくと、5 年後の自分に効きます
- 個人ツールは「動き続ける仕組み」に寄せた方が長生きします。タスクスケジューラ + `StartWhenAvailable` の組み合わせは、地味だが強い
- 全体を俯瞰する視覚化は、数字の羅列より遥かに意思決定に効きます。Excel を通り抜けた先に heatmap があります

## 締め

こういう小さな道具は、動き続けている限り少しずつ削って磨けます。「自分の道具を自分で作る」は、案外そんなに大げさな話ではなくて、`index.html` 1 枚と JSON 2 本から始められる、という記録でした。

リポジトリはこちらです。よかったら覗いてみてください。
https://github.com/ishizakahiroshi/heatfolio

---

※ 本文のインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。

