---
title: "保存する個人データを減らす migration は、デプロイの順番を間違えると本番が壊れる"
tags:
  - Supabase
  - PostgreSQL
  - Cloudflare
  - 個人開発
  - セキュリティ
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![保存する個人データを減らす migration は、デプロイの順番を間違えると本番が壊れる](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_manabi-map-v090_hero.png)

自分のサービスに監査をかけたら、privacy 関連で 5 件の指摘が出ました。コードを直すところまでは、まあ普通の作業です。詰まったのはそこではなくて、直したものを本番へ入れる順番でした。

先に結論を書きます。**保存済みデータを減らす migration に CHECK 制約が付いていて、今デプロイされているフロントがその制約を満たさない値を書く場合、migration を適用した瞬間から本番の書き込みが失敗します。** 制約とコードのどちらを先に出すかで、壊れる時間の長さが変わります。

## 何のサービスの話か

自作で [manabi-map](https://github.com/ishizakahiroshi/manabi-map) という進路検討の Web サービスを作っています。住所を起点に通える高校を地図で見て、親子で比較・記録・検討できる進路管理サービス（全国 47 都道府県・OSS）です。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=manabi-map
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/manabi-map

インストールは不要で、ブラウザで開くだけです。

https://manabi-map.app

構成は React + TypeScript + Vite、データベースは Supabase、ホスティングは Cloudflare Pages です。今回の話は Supabase の Postgres と Cloudflare Pages Functions の両方に関わります。

### 前回の記事

SSR まわりの話は前に書きました。今回はその上に載っている privacy の話です。

SEO 用のプリレンダー HTML が初期表示を壊していた。createRoot から hydrateRoot へ移すまで
https://qiita.com/ishizakahiroshi/items/0bfa08174025670c0b00

## 指摘の中身は「説明と実装が食い違っている」

出た指摘のうち一番効いたのはこれです。

プライバシーポリシーには「住所は保存しない」と書いてある。でも実装は、ジオコーダが返した住所文字列をそのまま `address` 列に入れて、緯度経度も小数第 7 位まで持っていました。

嘘をつくつもりは無かったんですが、書いた時期と実装した時期がずれていて、そのまま気づかずに来ていました。利用者に未成年が含まれるサービスでこれは良くない。

直す方向は 2 つあります。説明を実装に合わせる（正確な開示にする）か、実装を説明に合わせる（保存を減らす）か。後者を選びました。

## 決めた保存契約

- 永続化する label は「設定地点」に固定する。ジオコーダが返した詳細な文字列は保存しない
- 緯度経度は小数第 3 位へ丸める。だいたい 100m 単位です
- 生の入力と、現在地の正確な座標は永続化しない

小数第 3 位で足りるのかは気になるところでした。周辺の学校を出す、距離を測る、経路を開く。この 3 つがサービスの用途なので、100m ずれても実用上は変わりません。実際に丸めた値で動かしてみて、問題ないことを確認してから決めました。

## 実装は保存の境界に 1 箇所だけ置く

正規化を UI 側に散らすと、必ずどこかの経路が漏れます。保存の直前に通す純粋関数を 1 つ作って、DB の insert / update、localStorage、匿名アカウントからログインへの移送、DB から読んだ後の再保存を全部そこに通しました。

```ts
const PERSISTED_HOME_LABEL = '設定地点'

export function normalizeHomeForPersistence(value: unknown): HomeLocation | null {
  // ...
  const roundCoordinate = (coordinate: number) => {
    const rounded = Number(coordinate.toFixed(3))
    return Object.is(rounded, -0) ? 0 : rounded
  }
  return {
    label: PERSISTED_HOME_LABEL,
    lat: roundCoordinate(value.lat),
    lng: roundCoordinate(value.lng),
  }
}
```

`-0` を潰しているのは、`toFixed` が負の極小値で `-0` を返すからです。DB に入れる前に消しておきます。

## 既存データも減らす。これは戻せない

新しく保存する分だけ直しても、すでに入っているデータは詳細なまま残ります。そこで既存行を同じ規則へ変換する migration を書きました。

```sql
begin;

do $$
declare
  target_count bigint;
  converted_count bigint;
begin
  select count(*) into target_count
    from public.home_locations
   where label is distinct from '設定地点'
      or address is distinct from '設定地点'
      or latitude is distinct from round(latitude, 3)
      or longitude is distinct from round(longitude, 3);

  update public.home_locations
     set label = '設定地点',
         address = '設定地点',
         latitude = round(latitude, 3),
         longitude = round(longitude, 3)
   where label is distinct from '設定地点'
      or address is distinct from '設定地点'
      or latitude is distinct from round(latitude, 3)
      or longitude is distinct from round(longitude, 3);

  get diagnostics converted_count = row_count;
  if converted_count <> target_count then
    raise exception 'count mismatch: expected %, updated %', target_count, converted_count;
  end if;
end $$;

alter table public.home_locations
  add constraint home_locations_label_minimized
    check (label = '設定地点') not valid;

alter table public.home_locations
  validate constraint home_locations_label_minimized;

commit;
```

実際には label / address / latitude / longitude の 4 本の CHECK を足しています。抜粋なので 1 本だけ載せました。

この変換で捨てる住所文字列と座標の精度は、SQL では戻りません。だから作るのと流すのを別の作業に分けて、流す前に必ず backup を取る運用にしました。

DO ブロックの中で件数を数えてから UPDATE して、更新件数が一致しなければ例外を投げています。静かに一部だけ変換されて終わるのが一番こわいので。

## ここで詰まった。制約を先に入れると本番が壊れる

migration の準備ができて、いざ流そうとしたところで気づきました。

`label = '設定地点'` の CHECK を本番に入れる。その時点で本番にデプロイされているフロントは、まだ古いバンドルです。古いコードは `label: '自宅'` と詳細な住所を書きます。

つまり **migration を適用した瞬間から、新しいバンドルが配信されるまでの間、利用者が地点を設定する操作が制約違反で失敗します。**

当たり前と言えば当たり前なんですが、手順書には「backup → migration 適用 → Preview 目視 → 本番マージ」と書いてあって、その順番だと Preview の目視をしている間ずっと本番が壊れていることになります。目視は人がやるので、何分で終わるか読めない。

確認したのはこの 1 行です。

```bash
git show origin/main:web/src/contexts/AppContext.tsx | grep -n "label:"
# 118:        label: '自宅',
# 189:        const h = { label: data.address, ... }
```

本番のコードが何を書くのかを、想像ではなく実物で見る。これをやっていなかったら普通に踏んでいました。

## 順番を入れ替えた

やることは同じで、順番だけ変えました。

1. Preview の目視検収を先に済ませる（Preview は新しいバンドルなので、制約が無くても検証内容は変わらない）
2. backup を取る
3. migration を適用する
4. **間を置かずに** main へマージして本番デプロイを始める

これで壊れている窓が、適用からデプロイ完了までに縮みます。実際は 6 分ほどでした。

![CHECK 制約を先に入れるか、新しいコードを先に出すか。壊れている区間の長さが変わる](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-01_manabi-map-v090_fig1.png)

上が元の手順、下が入れ替えた後です。壊れている区間の長さが変わります。

migration 自体は、末尾の `commit;` を `rollback;` に差し替えたコピーを本番へ流して事前検証しました。DO ブロックの assertion も ALTER TABLE も全部通り、最後に巻き戻るので本番データは変わりません。CHECK 制約が既存行を通ることまで実データで確認できます。

```bash
psql -v ON_ERROR_STOP=1 -f dryrun_202608310101.sql
# NOTICE:  home_locations rows to minimize: N
# ALTER TABLE
# ROLLBACK
```

## ついでに直した 3 つ

同じ監査で出た残りです。短く。

**メンテナンス中の HTTP status**。停止中のページを 200 で返していました。クローラーには「これが正常な内容です」と伝わるので、503 と `Retry-After` に変えました。`/api/` と `/legal/` と `/assets/` と `robots.txt` の素通しはそのまま維持しています。

**家族招待リンクのトークン**。`?token=` を `#token=` に移しました。fragment はサーバーへ送られないので、アクセスログにも CSP レポートの URL にも写りません。すでに配ったリンクの旧形式も有効期間内は読みますが、読んだ直後に `history.replaceState` で URL から消します。

**CSP レポートの受け口**。受け取った内容をそのままログに出していました。許可した項目だけを抽出して、URL からは認証情報と query と fragment を落とす形に変えました。

## 「直した」と「動いている」は別なので測った

ここが一番書きたかったところかもしれません。unit test が通っているのと、デプロイされた環境でそう動くのは別です。

CSP レポートの redaction は、URL の各所に合成の秘密を仕込んだレポートを実際に投げて、ログに何が出るかを見ました。

投げた側:

```json
{"csp-report":{
  "document-uri":"https://example.com/family/join?token=SYNTHETICTOKEN9F3A2B#fragSECRETFRAG",
  "referrer":"https://ref.test/p?rt=REFERRERSECRET",
  "blocked-uri":"https://user:PASSWORDSECRET@evil.test/x.js?q=QUERYSECRET",
  "source-file":"https://example.com/assets/a.js?sf=SOURCEFILESECRET"
}}
```

ログに出た側:

```json
{"documentUri":"https://example.com/family/join","referrer":"https://ref.test/p",
 "blockedUri":"https://evil.test/x.js","sourceFile":"https://example.com/assets/a.js",
 "violatedDirective":"script-src","disposition":"report","statusCode":200}
```

仕込んだ 6 個が全部消えていました。`original-policy` は allowlist に入れていないのでそもそも出ません。

メンテナンスの 503 は、Preview 環境**だけ**に環境変数を立てて測りました。Pages の環境変数は Preview と Production が別なので、本番を巻き込まずに試せます。ただし `wrangler pages secret put` には環境を指定するフラグが無くて、既定で Production に当たり得ます。REST API で `deployment_configs.preview` だけを PATCH する経路にしました。

変更前に project 設定の snapshot を取っておいて、検証後に消してから JSON で突き合わせて完全一致を確認する。ここまでやらないと、戻し忘れに気づけません。

## rate limit は、無料プランでは指定どおりに作れなかった

CSP レポートの受け口は認証がありません。ブラウザが勝手に送ってくるものなので、認証のかけようがない。abuse boundary として rate limit を入れることにしました。

設計上は「同一 IP 60 requests / 1 minute、超過したら 10 分ブロック」のつもりでした。作れませんでした。

Cloudflare の Free プランだと、rate limiting rule の Period が `10 seconds` の 1 択、Block の Duration も `10 seconds` の 1 択です。ドロップダウンを開いても選択肢が 1 つしか無い。

なので実際に入れたのは「10 秒間に 20 回を超えたら 10 秒ブロック」です。本番で 30 連射して、20 回目までが 204、21 回目から 429 になることを確認しました。

正直に書いておくと、**これは事故を止める速度制限であって攻撃対策ではありません。** 1 つの IP から 20 秒ごとに 20 回、平均で毎秒 1 回のペースは通り続けます。1 日にすると約 8.6 万回。Workers / Pages の Free プランは 1 日 100,000 リクエストなので（https://developers.cloudflare.com/workers/platform/pricing/ ）、1 つの IP でほとんど食える計算です。IP を分散されればルール自体を素通りします。

しかも Pages Functions の middleware はほぼ全リクエストで走るので、枠を使い切ると受け口だけでなくサイト全体が止まりえます。そこが本当の危険で、rate limit では埋まっていない。

止められるのは、暴走したスクリプトの無限送信と、軽い手動 abuse まで。ここを曖昧にしたまま「対策済み」と書くと、あとで自分が困ります。

## 学んだこと

- 制約を足す migration は、**その時点で本番に出ているコードが制約を満たすか**を先に見る。`git show origin/main:<file>` で実物を読むのが速い
- 不可逆な変換は、作るのと流すのを別の作業にする。流す前の backup と、`rollback;` に差し替えた dry-run はセットで効く
- redaction は実際のログで見る。「消しているはずのコード」を読んでも、消えている証明にはならない
- プラットフォームの制約で設計どおりに作れなかったら、作れた範囲と、埋まっていない穴を両方書き残す。作れた方だけ書くと「対応済み」に見えてしまう
- ポリシー文書と実装は、どちらかを直した時点でもう片方がずれる。片方だけ直して満足しない

## manabi-map はこんなときに刺さります

- 通える範囲にどんな高校があるのか、地図で見ながら家族で話したい人
- 学校ごとに文化祭や説明会のメモを残して、あとで比較したい人
- 偏差値の数字だけでなく、その根拠がどこから来ているかを確認したい人
- 保存される個人データがどこまでか、はっきり書いてあるサービスを使いたい人（今回の版でここを直しました）

ブラウザで開くだけで使えます。設定ファイルもインストールも要りません。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=manabi-map
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/manabi-map
- 公開データと API の説明: https://manabi-map.app/data/

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## あわせて読みたい

Cloudflareの「10,000PV突破おめでとう」が9割bot だったので、開き直って公開APIを作った話
https://note.com/ishizakahiroshi/n/n3784e1e48f0e

全国5,095校のサイトマップを、今日1日で作り直した話
https://note.com/ishizakahiroshi/n/n3427e64c1d90

## おわりに

監査を回すところまでは、正直わりと気持ちいい作業です。指摘が並ぶと「やることが決まった」感じがする。

しんどいのはその後で、本番に入れる順番とか、無料プランで作れないとか、そういう地味なところで止まります。今回も一番時間を使ったのは SQL でもコードでもなくて、「この順番で流したら誰か困らないか」を考えていた時間でした。

小さいサービスだから止まっても誰も気づかないかもしれない。でも気づかれないことと、壊していいことは別だと思っています。次に制約を足すときも、先に本番のコードを読むところから始めます。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-09-01_manabi-map-v090-address-minimization/

※ ヘッダー画像は AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
