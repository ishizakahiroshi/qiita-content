---
title: "Cloudflareの「10,000PV突破おめでとう」が9割bot だったので、開き直って公開APIを作った話"
tags:
  - Cloudflare
  - bot
  - OpenData
  - 個人開発
  - データ品質
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260807-bot-traffic-to-open-data-api-hero.png)

Cloudflare から「あなたのサイトが月間 10,000 PV を突破しました」というメールが届きました。実数は 20,034 PV。個人で運営している小さなサービスなので、素直に嬉しかったです。

ただ、国別の内訳を見て手が止まりました。米国 8,594、日本 142。日本の中学生と保護者向けのサービスなのに。

## manabi-map について

自作で [manabi-map](https://github.com/ishizakahiroshi/manabi-map) という進路管理サービスを作っています。住所を起点に通える範囲の高校を地図で見ながら、親子で比較・記録・検討できるサービスです。全国 47 都道府県・5,095 校を収録し、OSS として公開しています。

サービス本体はこちらです。https://manabi-map.app

リポジトリはこちらです（Star をいただけると励みになります）。https://github.com/ishizakahiroshi/manabi-map

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260807-bot-traffic-to-open-data-api-infographic.png)

## 日本 142 に対して、米国 8,594

Cloudflare の Security Analytics を開いて、24 時間ぶんを見ました。総リクエスト 9.23k。国別の合計とぴったり一致するので、あの「PV」はリクエスト数のカウントでした。

内訳で目を引いたのが Source IPs です。

```
74.7.242.25                          5.35k
240b:12:48c0:6900:1c86:99db:99c4:a6b0   87
2a03:2880:f814:15::                     65
```

1 つの IP が 5,350。全体の 58% です。

### UA は最初から見ない

素性を割るときに User-Agent は見ていません。偽装が 1 行で済むものは判定材料になりません。代わりに使ったのは、詐称にコストがかかる 2 つです。

```
# 1. 割り当てを引く（IP の割り当ては詐称できない）
curl -s https://rdap.arin.net/registry/ip/74.7.242.25 | jq '.name, .country, .remarks'

# 2. 逆引きを引く（PTR は IP の保有者しか設定できない）
nslookup 74.7.242.25
```

結果は `74.7.128.0/17`、登録者 MICROSOFT-MAINT、国 US。そして **PTR レコードがありません**。

ここで Bingbot の線が消えます。大手クローラーは「**往復確認**」で検証できるようになっているからです。逆引きで名前を取り、その名前を正引きし直して元の IP に戻るか見る、という手順です。Microsoft は Bingbot について「逆引きで `*.search.msn.com` が返ること」を公式に案内しています。Google も Yandex も Apple も同じ形式です。

裏返すと、**PTR を持たない IP は、どの大手クローラーでもない**と言い切れます。往復確認の入口が存在しないからです。

Azure の VM は既定で逆引きが設定されません。Microsoft の AS で、PTR がなくて、UA がブラウザとして認識されない。つまり Microsoft ではなく、Azure を借りている誰かです。

## 5,350 という数字に見覚えがあった

サービスの収録校数は 5,095 校です。SEO 用に `/school/<uuid>/` の静的ページを 1 校 1 枚生成しているので、学校ページはちょうど 5,095 枚あります。

5,350 リクエスト。誤差数パーセントで一致します。

サイト全体を頭から 1 周舐めた数でした。サンプルログの中身も `/school/<uuid>/` の羅列で、人間の閲覧パターン（トップ → 都道府県 → 数校）とは似ても似つきません。

ロシアからの 136 は YandexBot でした（`/robots.txt` を 2 回取りに来ていて行儀がいい）。Tencent Cloud の IP、Meta の OGP クローラーも混ざっています。Source OS の「Unknown/Others」が 8.94k という数字が全部を物語っていました。

## じゃあ、本当の訪問者は何人なのか

ここで気になったのが実数です。

Cloudflare Web Analytics を見ました。設定した記憶はあったので開いてみたら、ちゃんと動いていました。bot 除外つきで、直近 24 時間の Visits 12、Page views 26。

12 人。

推定はしていました。日本からの 142 リクエストを、SPA が 1 訪問あたり出すリクエスト数（10〜25）で割ると 5〜15 セッション。当たっていたわけですが、当てたところで 12 人は 12 人です。

ちなみにここで 1 回誤診しています。最初、`curl` で本番 HTML を取って beacon スクリプトを grep したら見つからず「Web Analytics が入っていない」と結論しました。実際は入っていて、Cloudflare の Automatic setup は **edge でブラウザの User-Agent にだけ beacon を注入する**仕様でした。

回避策は単純で、**同じ URL を UA だけ変えて 2 回取り、バイト数を比べる**ことです。

```
curl -s -o /dev/null -w '%{size_download}\n' https://manabi-map.app/
# 8190

curl -s -o /dev/null -w '%{size_download}\n' \
  -A 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36' \
  https://manabi-map.app/
# 8549
```

359 バイトの差が beacon です。中身を grep する前にこれをやれば「edge で何かが注入されている」とすぐ分かりました。

**bot 判定を伴う機能を `curl` の既定 UA で検証すると偽陰性が出ます。**WAF ルール、bot 対策、A/B 配信、地域別配信も同じです。「入っていない」ではなく「その UA には出していない」だけかもしれません。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260807-bot-traffic-to-open-data-api-fig1.png)

上の図は 24 時間ぶんのリクエストを分解したものです。9.23k のうち、実人間に紐づくのは 200 前後。残りは全部クローラーでした。

## 一瞬、よからぬことを考えた

正直に書くと、ここで「このアクセス数でアフィリエイトの小銭くらい稼げないか」と考えました。

無理でした。しかも 3 段階で無理です。

まず、広告タグもアフィリエイトタグも JS が実行されて初めてインプレッションが発火します。HTML を取って終わるクローラーはタグまで到達しません。

次に、A8 やもしもは基本が成果報酬です。クリックの先で申込が完了して発生します。bot は当然コンバージョンしません。

そして仮に到達しても、広告業界には IVT（Invalid Traffic）フィルタが標準で入っています。データセンター IP、PTR なし、UA がブラウザでない、5,095 ページを均等に舐める。IVT 判定の教科書に載っているような指紋が全部揃っていました。

そもそも、仮に抜け道があってもそれは ad fraud です。中学生と保護者が使うサービスで天秤にかける話ではありませんでした。

## 止められないなら、公式に配ればいい

ここで考え方を変えました。

スクレイプは止められません。robots.txt は紳士協定ですし、UA を偽装されたら見分けもつきません。Cloudflare Pages の静的配信は無料枠なので、叩かれても課金は増えません。実害は「アナリティクスの数字が実態を映さない」ことだけです。

だったら、**公式にデータセットと名乗ってしまえばいい**。

manabi-map のデータは CC BY-SA 4.0 です。公式に配布物として宣言すれば、再利用する側に**帰属表示（リンク）の義務**が発生します。黙って抜かれ続けるより、出典付きで引用される方が被リンクにも認知にも効きます。しかも商用の偏差値サイトが絶対に真似できない差別化になります。

方針が決まりました。`/api/v1/` に安定エンドポイントを置き、`/data/` に説明ページを作り、`Dataset` 型の JSON-LD を出して Google Dataset Search に載せる。

そして 1 つ、譲れない条件を置きました。**公開する各レコードは、出典元の URL を必ず同梱して、そこから元資料へ後追いできること。**

「別途 DATA.md に出典をまとめて書く」では足りません。データを受け取った第三者が、そのレコード単体を見て元の PDF まで辿れること。牛肉の個体識別番号のイメージです。

## 出典は学校単位ではなく、項目単位で落とす

最初は `schools` テーブルに `source_url` 列を 1 本足すつもりでした。すぐやめました。

1 校のレコードには公式サイト URL、全校生徒数、男子比率、緯度経度、学科構成が入っていて、**項目ごとに出典が違います**。公式サイト URL は県教育委員会の一覧、生徒数は県の統計資料、学科は学校のパンフレット。1 列では表せません。

出典を別テーブルに分離しました。

```sql
create table public.school_field_sources (
  school_id        uuid not null references public.schools (id) on delete cascade,
  field_name       text not null
    references public.school_field_source_field_master (code)
    on update cascade on delete restrict,
  official_url     text not null,
  doc_title        text not null,
  published_at     date,
  source_page_or_table text,
  last_verified_at timestamptz,
  last_http_status integer,
  is_official_source boolean not null,
  primary key (school_id, field_name, official_url)
);
```

設計で効いたのは 3 つです。

**主キーに URL まで入れる。**`(school_id, field_name)` を主キーにすると、同じ項目に資料が 2 つあるとき片方が消えます。県の一覧と学校自身の告知の両方を持てるようにしました。

**`field_name` を master への FK にする。**`schools.official_url` のような `table.column` 形式の code だけを受け付けます。master 側に CHECK を置いて、`code = table_name || '.' || column_name` を強制しています。

```sql
constraint school_field_source_master_code_matches_columns
  check (code = table_name || '.' || column_name)
```

フリーテキストにすると、`official_url` と `schools.official_url` と `officialUrl` が並んで、集計するたびに手で寄せる羽目になります。**表記ゆれは運用で直すものではなく、入らないようにするものだと思っています。**

**404 になっても行を消さない。**`last_http_status` と `last_verified_at` を持たせました。消すと、次の巡回でまた同じ URL を引きに行きます。リンク切れは「情報がない」ではなく「到達できなかった」という**状態**です。状態を消すと、同じ失敗を何度でも繰り返します。

そして出力側です。出典のある項目だけを公開レコードに載せます。

```js
// 出典が確認できた項目だけを残す
for (const [column, sourceCode] of SOURCED_SCHOOL_FIELDS) {
  if (row[column] != null && sourcedFields.has(sourceCode)) record[column] = row[column]
}
```

つまり出典が足りない学校をまるごと落とすのではなく、**足りない項目だけが落ちます**。入試統計はさらに細かく、メトリクス単位です。倍率と受験者数と合格者数で出典が違うので、`fact_kind_code` が一致する出典を持つメトリクスだけを出します。出典が 1 つも無ければ、その統計は丸ごと出しません。

## 出典を必須にしたら、3 分の 1 が消えた

出典ゲートを実装しました。`official_url`（学校の公式サイト）が入っていない学校は公開対象から外す、というシンプルなルールです。

群馬県で測ったら 83 校中 83 校、100% でした。これなら全校通るなと思って、設計メモにも「実質全校が通る」と書きました。収録データは公開しているので、群馬の一覧はここから見られます。https://manabi-map.app/pref/gunma/

実装して動かしたら、こうなりました。

```
official_url あり: 3,415 / 5,095 校（67.0%）
```

11 府県が丸ごとゼロでした。大阪 242、愛知 224、兵庫 220、福岡 170、京都 101。

1 県の実測を全国に一般化していました。

## 原因は 3 週間前の自分だった

欠落がランダムではなく西日本に固まっているのが引っかかりました。

3 週間前、西日本 27 府県のデータ投入を AI エージェントで一括処理して失敗し、段階展開に戻したことを記事に書いています。https://qiita.com/ishizakahiroshi/items/8b5d57a9df4c678299b1

段階展開に戻した判断自体は正しかったです。ただ、その段階展開の中で別の穴が空いていました。

投入時の `candidate.sql` を実際に開いて比べたら、3 段階で劣化していました。

鳥取（最初のブロック）は列に `official_url` があり、値も入っている。愛知（中盤）は列はあるが値が `null`。福岡（終盤）に至っては、**列リストから `official_url` そのものが消えている**。

ブロック別の充足率がこれです。

| ブロック | official_url |
|---|---|
| block-0 鳥取 | 100.0% |
| block-1 中国 | 100.0% |
| block-2 四国 | 62.6% |
| block-3 東海 | 31.9% |
| block-4/5 近畿 | 13.2% |
| block-6 北九州 | 0.0% |
| block-7 南九州 | 13.3% |

きれいに落ちていきます。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260807-bot-traffic-to-open-data-api-fig2.png)

根本原因は、**ブロックごとに insert の列リストを手書きしていた**ことでした。各ブロックが独立に SQL を生成する設計で、共通のスキーマ契約がありません。だから後半のブロックで項目が抜けても誰も検知しませんでした。

そして厄介なのは、この後です。他の項目も調べたら、同じことが起きていました。

| ブロック | official_url | name_kana | school_departments |
|---|---|---|---|
| block-0 鳥取 | 100.0% | 100.0% | 100.0% |
| block-3 東海 | 31.9% | 100.0% | 99.6% |
| block-4/5 近畿 | 13.2% | 22.8% | 43.1% |
| block-6 北九州 | 0.0% | 0.0% | 74.9% |

**項目ごとに崩れ始める時期が違います。**ふりがなは近畿から、公式サイト URL は四国から。列リストが手書きだったという仮説を、これ以上ないくらい裏づけていました。

## カタログ 1 枚から取る

補完に入ります。ここで大事だったのは、**1 校ずつ検索しない**ことでした。

日本の高校は、県教育委員会が「県立高校一覧」を、私学協会が「加盟校一覧」を持っています。1 枚のページに数十校ぶんの公式サイト URL が並んでいます。1,680 校を 1 校ずつ検索したら、工数も相手サイトへの負荷も跳ね上がります。

3 波かかりました。

第 1 波で 1,359 校。ここで gap が 321 校残りました。全体の 19.1% で、設定していた停止条件（20%）の内側だったのでそのまま走り切っています。

第 2 波の前に内訳を見たら、**高知が 34 校中 0 件、香川が 11 校中 0 件**でした。全体率だけを見ていたので、県単位の異常が素通りしていました。

原因を調べたら 3 種類に分かれました。

香川は gap 11 校が全件私立なのに、見に行ったのが県教育委員会（公立）のページでした。宮崎と熊本はその逆です。手順書には「公立と私立で参照先が異なるので両方を押さえる」と書いてありました。書いてあったのに、片方しか当たっていませんでした。

高知は、カタログページに HTTP 200 が返っているのに 0 件でした。ページは存在します。第 2 波で抽出方法を変えたら、**同じ URL から 34 校すべて**取れました。鹿児島も同じで、21 → 43 に増えています。「200 なのに 0 件」はページ不在ではなく抽出失敗を疑うべきでした。

第 3 波では、残った 164 校の内訳を先に数えました。市立が 58 校（35%）。名古屋 14、神戸 9、京都 9。市立高校は各市の教育委員会が所管するので、**県のカタログにも県私学協会の名簿にも載りません**。参照先の分類そのものが足りていませんでした。

実際に使ったカタログはこのあたりです。

- 名古屋市の市立高校一覧 https://www.city.nagoya.jp/kodomo/schools/1015850/1015870.html
- 国立高専機構の全国一覧 https://www.kosen-k.go.jp/nationwide/all_kosen_linkmap
- 文部科学省の通信制高校検索 https://www.mext.go.jp/tsushinseikoukou/search/
- 高知県の公立高等学校ホームページ一覧 https://www.pref.kochi.lg.jp/doc/kochi-highschoolguide/

市教育委員会、国立高専機構、文部科学省の通信制高校検索を足して、最終的にこうなりました。

```
5,085 / 5,095 校（99.8%）
41 県が 100%、公開 API から消える県はゼロ
```

これは補完を終えた直後の実測値です。学校レコードは追加も是正も入るので、現在の値はデータセットの説明ページに出しています。https://manabi-map.app/data/

### gap は県別ではなく、所管別に数える

3 波かけて分かったのは、**取りこぼしの正体は地理ではなく所管だった**ということです。

県別に数えると「高知が 0 件」までしか見えません。同じ gap を所管別に数え直すと「市立が 58 校で最大の塊」が出てきます。県立高校しか載っていないカタログを何回舐めても、市立高校は 1 校も増えません。**参照先の分類が足りていないときは、回数を増やしても収束しません。**

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260807-bot-traffic-to-open-data-api-fig3.png)

上の対応表が、最終的に使った参照先です。設置者が変われば名簿の持ち主も変わります。私立の名簿を持つ県私学協会は、ドメインが `.or.jp` だったり `.gr.jp` だったり `.com` だったりして統一されていません。ここは後で効いてきます。

残り 10 校のうち半分は、URL が取れないのではなく **DB のレコードが実態と合っていない**ケースでした。男子部と女子部が別々に運営されている学校を 1 行で「共学」と持っていたり、統合前の校名と統合後の校名が両方「募集中」で並んでいたり。これはカタログを増やしても解決しません。別課題として切り出しました。

## 文章で書いた防御を、機械で落ちる形にする

設計メモには「1 県でも全欠落したら生成を停止する」と書いてありました。実装されていませんでした。

これが一番危なかったところです。データ補完前にビルドしていたら、大阪・愛知・兵庫・福岡が丸ごと欠けた API が公開され、**しかも検証は通っていました**。ファイルは全部生成されているし、中身の形式も正しいので、落ちる理由がありません。

そこで、ビルド後の検証スクリプトに throw を足しました。

```js
const expectedPrefApiSlugs = new Set(
  prefectures
    .filter((prefecture) =>
      schools.some((s) => s.is_active !== false && s.prefecture === prefecture.name))
    .map((prefecture) => prefecture.slug),
)
const missingPrefApiSlugs = [...expectedPrefApiSlugs].filter(
  (slug) => !actualPrefApiSlugs.has(slug),
)
if (missingPrefApiSlugs.length > 0) {
  throw new Error(`public prefecture API is missing source prefectures: ${missingPrefApiSlugs.join(', ')}`)
}
```

ポイントは `expectedPrefApiSlugs` の作り方です。**47 をハードコードしていません。**「アプリ側データに実在する県」から期待値を組み立てて、生成物と差集合を取ります。定数で書くと、県が増えたときに定数を直し忘れて、また同じ穴が空きます。期待値は宣言せず、入力から導出させます。

同じ発想でゲートを重ねました。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260807-bot-traffic-to-open-data-api-fig4.png)

生成側は「載せてよいものだけ載せる」、検証側は「載ってはいけないものが載っていたら落とす」で、向きが逆です。片方だけだと、実装を書き換えたときに静かに緩みます。

検証側の 4 つはこうです。

**1. 都道府県の差集合**。上のコード。

**2. `deviation` リーク検出**。公開 JSON を再帰的に走査して、キー名に `deviation` を含むものが 1 つでもあれば JSON パス付きで throw します。偏差値は API に出さない方針なので、キー名レベルで機械的に禁じました。

```js
if (/deviation/i.test(key)) return `${path}.${key}`
```

**3. 出典ホストの拒否リスト**。Wikipedia と商用の偏差値サイト 3 ドメインを正規表現で弾きます。「うっかり出典として登録してしまった」を公開前に落とすためです。

**4. 出典ホストの許可リスト**。`ed/ac/lg/go.jp`、`pref.<県>.jp`、`city*.jp` に当たらないホストは、**明示登録がない限り throw** します。

そして、これらが本当に落ちるかを負のテストで固定しました。現時点で 28 本あります。

- 県ファイルを 1 つ削ると、欠けた県の slug 付きで落ちる
- `deviation` を含むフィールドを 1 つ紛れ込ませると落ちる
- `official_url` の無いレコードを 1 件混ぜると落ちる
- **公開してよい学校を落としすぎても落ちる**（絞りすぎの逆方向も守る）
- 出典の無い入試メトリクスを 1 つ出すと落ちる
- 未審査の `gr.jp` を出典ホストにすると落ちる／審査済みの私学協会ホストは通る

最後の 1 つが気に入っています。許可リストは「弾くこと」だけテストすると、リストを空にしても全部通るのと区別がつきません。**通るべきものが通るテストと、落ちるべきものが落ちるテストは両方要ります。**

もうひとつ、名乗りの文言そのものにもテストを付けました。

```js
export function formatDatasetCoverage(prefectureCount, schoolCount, nationwidePrefectureCount) {
  const prefix = prefectureCount === nationwidePrefectureCount ? '全国 ' : ''
  return `${prefix}${prefectureCount} 都道府県・${schoolCount} 校`
}
```

全県そろっているときだけ「全国」と書けます。1 県でも欠ければこの接頭辞は消えます。**データセットのカバレッジを人が手で書くと、必ず実態より大きく書きます。**書けなくしました。

文章は実行時に飛ばされます。機械で落ちる形にして初めて防御になります。

## 許可リストを厳しくしたら、本物の一次資料が弾かれた

上の 4 番目のゲートで、実際にビルドが落ちました。

私立高校の名簿を持っているのは各県の私学協会です。自治体ではないので `ed/ac/lg/go.jp` を持っていません。実際のドメインはこうです。

```
www.osaka-shigaku.gr.jp
www.hyogo-shigaku.or.jp
kumamoto-pref-hs.jp
k-shigaku.com
```

`.gr.jp` も `.or.jp` も `.com` もあります。ここで「じゃあ `or.jp` と `gr.jp` を丸ごと許可しよう」とやると、許可リストの意味がほぼ消えます。任意団体でも取れるドメインなので、審査の網が広がりすぎます。

採った回避策は、**収集時に実際に中身を確認したホストだけを列挙する**ことでした。コードにもそう書き残してあります。

```js
// 私学協会・教育情報ポータル等は一次資料の発行主体でも、自治体向けの
// ed/ac/lg/go.jp suffix を持たない。任意の .or.jp / .gr.jp を広く許可せず、
// 収集時に確認した公式カタログの host だけを明示する。
const TRUSTED_OFFICIAL_SOURCE_HOSTS = new Set([ /* 15 ホスト */ ])
```

このリストは最初から完全ではありませんでした。足りないホストが残っていて、ビルドが落ちました。**それでよかったと思っています。**落ちた場所がビルドだったので、直すのに 1 コミットで済みました。許可リストを緩くしていたら、落ちる代わりに素性の怪しい出典が公開 API に混ざって、気づくのは誰かに指摘されたときです。

厳しい許可リストは運用で必ず 1 回落ちます。**落ちる場所を、公開後ではなくビルドにしておく**のが設計の仕事だと思っています。

## 学んだこと

**1 県の実測を全国に一般化しない。**群馬 83 校の一覧（https://manabi-map.app/pref/gunma/ ）を見て「実質全校が通る」と書いたのが全部の始まりでした。データ投入がバッチ単位で行われている以上、バッチごとに品質が違って当たり前です。全 47 県を集計してから判断すべきでした。

**停止条件は全体率と個別率の両方で見る。**全体 19.1% は閾値内でした。でも県単位では 0/34 が生じていました。平均は異常を隠します。

**文章で書いた防御は守られない。**これが一番刺さりました。手順書には「公立と私立の両方を押さえる」と書いてありました。守られませんでした。設計メモの「1 県でも全欠落したら停止する」も実装されていませんでした。どちらも、読む人が読む前提で書かれていました。上の節に書いたとおり、機械で落ちる形にして初めて防御になります。

**期待値をハードコードしない。**「47 県あるはず」と定数で書くと、県が増えたときに定数を直し忘れます。期待値は入力から導出して、生成物と差集合を取る。宣言ではなく計算で持つと、更新漏れが構造的に起きません。

**「HTTP 200 で 0 件」は抽出失敗を疑う。**同じ URL から 0 件と 34 件が出ました。ページが無いのではなく、こちらの取り方が悪いだけでした。カタログページは今どき JS で描画されていることが多いので、**HTML をパースする前に背後の JSON を探す**方が速いです。愛知は `aichi-school-navi.aichi-c.ed.jp/json/schoolList.json` がそのまま使えました。DevTools の Network を XHR で絞って 1 回リロードすれば見つかります。HTML パーサを書くより確実で、相手サイトへのリクエストも 1 回で済みます。

## manabi-map はこんなときに刺さります

- 子どもの進学先を、偏差値の並び順ではなく通学のしやすさから考えたい人
- 学校のパンフレットや説明会の予定を、親子で 1 か所にまとめておきたい人
- 出典が確認できる情報だけを見たい人（偏差値は公的資料をもとにした編集推計として、根拠を確認できた範囲だけ載せています）
- 高校のオープンデータを再利用したい人（今回作った `/api/v1/` から、出典 URL つきで取れます）

サービスはこちらです。https://manabi-map.app

リポジトリはこちらです（Issue / PR 歓迎）。https://github.com/ishizakahiroshi/manabi-map

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## おわりに

bot に全件舐められたことが、結果的にデータ品質の棚卸しになりました。出典を必須にするルールを 1 本入れただけで、3 年分くらいの負債が可視化された感じです。

正直、朝の時点では「変なアクセスが来てるな」くらいの話でした。夜には公開 API ができていて、データの充足率が 67% から 99.8% になっていました。何が起きたのかよく分かっていません。

公開 API は本番に出ています。全件が `https://manabi-map.app/api/v1/schools.json`、県別が `https://manabi-map.app/api/v1/schools/<slug>.json`、説明ページが https://manabi-map.app/data/ です。CC BY-SA 4.0 なので、帰属表示さえしてもらえれば自由に使えます。

1 レコードにはこういう形で出典が同梱されています。

```json
{
  "name": "…",
  "prefecture": "群馬県",
  "official_url": "https://…",
  "provenance": { "field_sources": [ { "field_name": "schools.official_url", "official_url": "https://…", "doc_title": "…", "last_verified_at": "…" } ] },
  "lifecycle": "…"
}
```

出す順番には自分で仕掛けを入れてあります。migration と SQL を順番に適用してからでないとビルドが落ちます。自分で自分の足を止める仕掛けを作るのは、あまり気持ちのいいものではないですが、たぶん必要です。

止められないものは、名乗ってしまう。今回はそれでうまくいきました。次も同じ手が使えるとは限りませんが。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-08-07_bot-traffic-to-open-data-api/

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi

田舎で在宅のシステムエンジニアをしています。実務 18 年、バックエンドとインフラと AI 連携が専門です。現場の業務課題を、最小限の実装で確実に動くものにするのが信条です。業務委託・受注を受け付けています（フルリモート対応）。こんな相談、歓迎です。

- サイト: https://ishizakahiroshi.com/
- GitHub: https://github.com/ishizakahiroshi
- X: https://x.com/ishizakahiroshi
