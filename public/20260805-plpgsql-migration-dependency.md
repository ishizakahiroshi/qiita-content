---
title: "PostgreSQL の migration が「全部成功」したのに本番が壊れる。plpgsql の遅延解決と、カタログを見る検証の限界"
tags:
  - PostgreSQL
  - Supabase
  - plpgsql
  - migration
  - CI
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260805-plpgsql-migration-dependency-hero.png)

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260805-plpgsql-migration-dependency-infographic.png)

先に結論を書きます。plpgsql で書いた関数は、**参照している列やテーブルが存在しなくても `create or replace function` が成功します**。名前が解決されるのは実行時です。

なので migration の適用リストが依存順を見ていないと、こうなります。

```
=== apply 202608040105_...sql ===
    OK
=== apply 202608040106_...sql ===
    OK
=== apply 202608040107_...sql ===
    OK
```

全部 OK。ログもきれい。それでいて本番の機能が実行時に `column "expires_at" does not exist` で落ちる。

先週これを踏みかけました。適用の直前に気づいて止められたのですが、気づけたのが半分は偶然だったので書き残しておきます。

## 何をやっていたか

自分で運営している高校選びの Web サービスがあって、その v0.5.0 のリリース作業でした。セキュリティ監査の是正が主な中身で、DB 側の migration が 4 本。

リリース手順書には適用リストがこう書いてありました。

```powershell
foreach ($v in '...0105','...0106','...0107','...0109') {
  psql -v ON_ERROR_STOP=1 -f "$mig\${v}_*.sql"
}
```

`ON_ERROR_STOP=1` を付けているし、各ファイルは `begin;` / `commit;` で囲ってある。失敗したらそこで止まる。安全に見えました。

## 適用リストが 4 本では足りなかった

リリース前に本番の適用済み台帳を引いたら、想定と違いました。

```sql
select version from supabase_migrations.schema_migrations order by version;
```

`202608040101` から `0104` が**無い**。develop にはコミット済みなのに、本番には入っていませんでした。

これ自体は異常ではありません。migration ファイルがどのブランチにあるかと、本番 DB に適用済みかは別の話です。リリース前に未適用なのは正常な状態です。

問題はその先でした。適用予定の 4 本を読み直すと、こうなっていた。

- `0105` の関数本体が `family_members.expires_at` を参照している。この列を作るのは `0101`
- `0107` の関数本体が `admin_pin_attempts` テーブルを参照している。これを作るのは `0103`

つまり、**リストに載っている 4 本は、リストに載っていない 3 本に依存していた**。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260805-plpgsql-migration-dependency-fig1.png)

図にすると、左が手順書に書いてあった 4 本で、全部 OK が並びます。右が実際の依存で、そこに無い `0101` と `0103` が要る。左のログだけ見ていると、この食い違いは一生見えません。

## 依存が無いのに CREATE が通る

ここが今回いちばん怖かったところです。

PostgreSQL は plpgsql の関数本体を、作成時にはパースするだけで**名前解決はしません**。`check_function_bodies` を on にしても、チェックされるのは構文であって、テーブルや列が実在するかではない。

だから `expires_at` という列がどこにも無い状態でも、それを参照する関数は問題なく作成できます。

```sql
-- family_members に expires_at 列が無くても、これは成功する
create or replace function public.create_family_invite(p_group_id uuid)
returns uuid language plpgsql security definer as $$
begin
  insert into public.family_members (group_id, role, status, expires_at)
  values (p_group_id, 'member', 'invited', now() + interval '7 days')
  returning invite_token into v_token;
  return v_token;
end;
$$;
```

適用は 100% 成功します。オペレーターの画面には OK しか出ない。

壊れるのは、実際に誰かがその機能を使った瞬間です。今回で言えば家族共有の招待作成と、管理者の偏差値訂正。どちらも `42703`（undefined_column）や `42P01`（undefined_table）で落ちます。

しかも自分の構成では、DB は本番・プレビュー・CI が同じ 1 つを見ています。migration を流した瞬間から、まだデプロイしていない**旧バージョンの本番バンドル**が新スキーマの上で動き始める。壊れていたらその時点で実ユーザーに出ます。

適用リストを 7 本に直して事なきを得ました。

## 検証クエリが検出できなかった

もっと嫌だったのはこっちです。

手順書には適用後の検証クエリが書いてあって、家族共有まわりはこうなっていました。

```sql
-- 期待値: 5
select count(*) from pg_proc p
  join pg_namespace n on n.oid = p.pronamespace
 where n.nspname = 'public'
   and p.proname in ('create_family_group','create_family_invite', ...)
   and p.prosrc like '%auth.users%';
```

「5 本の関数に匿名ガードが入ったか」を `pg_proc.prosrc` の文字列一致で見ています。

これ、**依存が欠けていても 5 を返します**。関数のソースに `auth.users` という文字列があるかを見ているだけなので、その関数が実行時に動くかどうかは一切見ていない。

カタログを覗く検証は、名前解決されない plpgsql に対しては無力でした。当たり前と言えば当たり前なのですが、検証クエリを書いた時点では気づいていませんでした。

実際に呼ぶ検証に差し替えました。

```sql
begin;
select public.create_family_invite('00000000-0000-0000-0000-000000000000'::uuid);
rollback;
```

期待するのは `authentication required` です。認証情報が無いので当然そこで止まる。**止まったということは、その手前にある `expires_at` の参照が解決できた**という意味になります。`0101` が抜けていれば、その前に `column "expires_at" does not exist` で落ちる。

`rollback` で囲っているので本番データは変わりません。適用直後の 1 秒で「本当に動くか」が分かります。

実際に流した後の出力がこれでした。

```
BEGIN
ERROR:  authentication required
CONTEXT:  PL/pgSQL function create_family_invite(uuid) line 7 at RAISE
```

エラーが出て正解、という検証です。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260805-plpgsql-migration-dependency-fig2.png)

並べてみると差がはっきりします。左のカタログを見る検証は、依存が欠けていても期待値の 5 を返して通ってしまう。右の実呼び出しは `authentication required` で止まり、その手前まで名前解決できたことを示します。同じ DB に対して、片方は気づけず片方は気づける。

## おまけ: 自己検証アサーションが構造的に必ず失敗していた

依存順を直して適用したら、今度は migration 自身が止まりました。

```
psql:202608040105_....sql:405: ERROR:  C2 assert failed: family RPC anonymous guards are incomplete
```

各 migration の末尾に `do $$ ... raise exception ... $$;` で自己検証を入れる方針にしていて、そのアサーションが落ちた。トランザクション内なので全部ロールバックされ、被害はゼロです。

原因を見たら、修正内容ではなくアサーションのほうがバグっていました。

```sql
and pg_get_function_identity_arguments(p.oid) in ('text', 'uuid');
```

`pg_get_function_identity_arguments()` の戻り値を型名だけだと思っていたのですが、実際は**引数名を含みます**。

```
 pg_get_function_identity_arguments
------------------------------------
 p_name text
 p_token uuid
```

`in ('text','uuid')` は 1 件も一致しません。だから `function_count` が 0 になり、`<> 5` で必ず例外を投げる。**ガードが正しく入っていようがいまいが、このアサーションは通らない**書き方でした。

`p.pronargs = 1` に変えて解決しました。引数 1 個の版だけを数える、という元の意図はこれで足ります。

静的レビューでは見つかりませんでした。SQL としては正しく、意図も読める。実際に本番へ流して初めて出た類のバグです。テスト用の DB で一度通しておけば防げた、というのが素直な反省です。

## 学んだこと

- plpgsql の関数本体は CREATE 時に名前解決されない。**依存する列やテーブルが無くても作成は成功する**。壊れるのは実行時
- したがって migration の適用リストは、依存順を人間が保証するしかない。`ON_ERROR_STOP=1` は依存欠落を検出できない
- **カタログを覗く検証（`pg_proc.prosrc` の like 等）では依存欠落を検出できない**。トランザクション内で実際に呼んで `rollback` する検証を併用する。期待するエラーで止まれば、そこまでの名前解決は通っている
- migration に自己検証アサーションを入れるなら、そのアサーション自体を一度は実 DB で通しておく。構造的に必ず失敗する書き方をしていても、静的レビューでは読み飛ばす
- 「本番はどこまで適用済みか」を答えられるのは `supabase_migrations.schema_migrations` への 1 クエリだけ。ブランチを見ても分からない

最後のが根っこだと思っています。コードは Git のブランチで本番到達を制御できるのに、DB スキーマはブランチと無関係に人間が psql で適用する。この 2 本の経路を突き合わせる仕組みが無いと、いつかズレる。

適用済み台帳をリポジトリに落として CI で突き合わせる、というのを次にやろうと思っています。まだ入れていないので、これでうまくいくかは分かりません。

## Manabi Map について

この記事の題材は、自分で作っている [manabi-map](https://github.com/ishizakahiroshi/manabi-map) という高校選びの Web サービスです。住所を入れると通える高校が地図に出て、気になる学校を親子で保存・比較・メモできます。全国 47 都道府県・5,095 校を収録して OSS で公開しています。

サービス: https://manabi-map.app

リポジトリはこちらです（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/manabi-map

## おわりに

適用が全部 OK で終わったときほど、一度実際に呼んでみる。地味ですが、これを手順に入れるかどうかで結果が変わりました。

小さく。適用直後に 1 回叩く、を習慣にしていきます。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
