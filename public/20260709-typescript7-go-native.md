---
title: TypeScript 7（Go 移植版）がリリース。何が速くなって、誰が嬉しいのかを整理する
tags:
  - JavaScript
  - Go
  - 開発環境
  - TypeScript
  - tsgo
private: false
updated_at: '2026-07-09T10:00:51+09:00'
id: cc1a155894db8a90087d
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

結論から書きます。TypeScript 7 になっても、TypeScript で作られたアプリを使うエンドユーザーは 1 ミリも嬉しくなりません。速くなるのは開発者の待ち時間です。ビルド、型チェック、エディタの反応。そこが桁で縮みます。

2026 年 7 月 8 日、TypeScript 7.0 が正式リリース（GA）されました。コンパイラを JavaScript から Go に丸ごと移植した、いわゆる「tsgo」がついに本流になった形です。X のタイムラインで「TS が Go 化」と流れてきて、最初に思ったのが「これ、ビルドが速くなるだけだよね？」でした。調べてみると半分正解で半分外れだったので、この記事で整理します。導入タイミングの判断まで含めて書きます。

![TypeScript 7 Go ネイティブ化 ヒーロー画像](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-09_typescript7-go-native_hero.png)

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-09_typescript7-go-native_infographic.png)

## 何が出たのか。tsgo と Project Corsa の話

TypeScript のコンパイラ（tsc）と言語サービスは、これまで TypeScript 自身で書かれていました。それを Microsoft が Go に移植するプロジェクトを 2025 年 3 月に発表しました。コードネームは Corsa。発表時の公称は「約 10 倍高速」です。

- 2025 年 3 月: 移植発表（[A 10x Faster TypeScript](https://devblogs.microsoft.com/typescript/typescript-native-port/)）
- 2026 年 4 月: TypeScript 7.0 Beta（[Announcing TypeScript 7.0 Beta](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-beta/)）
- 2026 年 6 月 18 日: RC（[Announcing TypeScript 7.0 RC](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-rc/)）
- 2026 年 7 月 8 日: GA（[Announcing TypeScript 7.0](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)）

インストールは拍子抜けするほど普通です。

```bash
npm install -D typescript
npx tsc --version
```

これだけ。今 npm から入る最新の `typescript` が、もう Go ネイティブ版です。`tsgo` という別コマンドを覚える必要はありません（nightly の `@typescript/native-preview` にだけその名前が残っています）。開発リポジトリは [microsoft/typescript-go](https://github.com/microsoft/typescript-go) で公開されています。

面白いのは移植の方針で、ゼロから書き直したのではなく、既存の JavaScript 実装（コードネーム Strada）をファイル単位で忠実に Go へ書き写しています。型チェックの挙動を変えないためです。実際、約 2 万件のコンパイラテストのうち、エラー検出が食い違うのは 74 件だけと報告されています（[Progress on TypeScript 7 - December 2025](https://devblogs.microsoft.com/typescript/progress-on-typescript-7-december-2025/)）。Bloomberg、Figma、Google、Linear、Notion、Slack、Vercel などが 1 年以上実プロジェクトで検証してきた、とも。

## 10 倍の中身。ネイティブ化と並列化が半分ずつ

「Go にしたから速い」だけではありません。公式の説明では、高速化の内訳はざっくり半分がネイティブコード化、残り半分が共有メモリでの並列処理です。JavaScript 実装はシングルスレッドの制約が重く、マルチコアを活かせていなかった。Go にしたことでワーカー間でメモリを共有しながら並列に型チェックできるようになった、という話です。

ついでにメモリ使用量もおおむね半分になります。巨大な monorepo で tsserver が重い、落ちる、という悩みへの回答でもあります。

## で、ユーザーは嬉しくなるのか？

なりません。ここは冒頭の繰り返しになりますが、大事なので図にしました。

![速くなる場所と変わらない場所の対比図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-09_typescript7-go-native_fig1.png)

上の図のとおり、速くなるのは開発マシンと CI の中だけです。TypeScript はコンパイル時に型を全部消して JavaScript を出力する言語なので、出てくる JS は 6 系でも 7 系でも同じもの。ブラウザや Node.js で動くコードは 1 バイトも変わらず、アプリの実行速度もページの表示速度も変わりません。

じゃあ意味がないのかというと、逆です。開発者の体感は大きく変わります。

- 型チェック（tsc）が約 10 倍。数十秒待っていたやつが数秒で返る
- エディタの反応。プロジェクトを開いたときの読み込み、補完、ホバー、エラー表示が速くなる。日常で一番効くのはたぶんここです
- CI の型チェック時間が縮む。つまり CI の課金時間と、PR のマージ待ちが縮む

「ビルドが速くなるだけ」という最初の理解は、この 2 番目を見落としていました。tsc はビルドツールであると同時に、エディタの裏で常時動いている言語サービスでもある。そっちが速くなる方が、毎日の積み重ねとしては大きい。

## とはいえ、全員が今日乗り換えられるわけでもない

GA と言っても、周辺がすべて追いついたわけではありません。公式が挙げている制限で、判断に効くのはこの 3 つです。

1 つ目。プログラム的に使う API がまだ安定公開されていません。新しい API は 7.1 で提供予定です。これが刺さる代表が typescript-eslint で、TypeScript の内部 API に依存する lint を使っている場合、当面は互換パッケージの TypeScript 6 を併用する二本立てになります。

```bash
npm install -D typescript@npm:@typescript/typescript6
```

2 つ目。Vue、Svelte、Astro、MDX、Angular などの組み込み言語対応が未対応です。これらの言語ツール（Volar など）が TS 6 の API に依存しているためです。Vue プロジェクトの人は焦らず待ちでいいと思います。

3 つ目。JS の出力（emit）を tsc にやらせている場合、ダウンレベルは ES2021 target までで、レガシーデコレータ（experimentalDecorators）も未対応です。ただし、いまどきのプロジェクトはトランスパイルを Vite や esbuild や swc がやっていて、tsc は `noEmit` の型チェック専用機になっていることが多い。その構成ならこの制限には当たりません。

なお、これまでの JavaScript 実装（6 系）は今後、セキュリティ修正と重大バグ対応のみになります。新機能は 7 系にしか来ません。長く留まる場所ではない、というのがはっきりしています。

## 私はこうする。既存は 3 か月後、新規は最初から 7

判断を図にするとこうなります。

![導入タイミングの判断分岐図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-09_typescript7-go-native_fig2.png)

図のとおり、プロジェクトの状況で分けました。新規プロジェクトはこれから作るのにわざわざ 6 系を選ぶ理由がないので、最初から 7 で始めます。既存プロジェクトは 3 か月ほど置いてから入れます。エコシステム（typescript-eslint の対応、7.1 の API 公開、Vue 系ツールの追従）が落ち着くのを待つ意図です。

正直に書くと、noEmit 構成の既存プロジェクトなら今日上げてもほぼ困らないはずです。型チェッカーとしては 1 年以上実戦で叩かれていて、GA 時点で「枯れていない」と言うほど生煮えでもない。それでも 3 か月置くことにしたのは、開発の待ち時間に今そこまで不満がないからです。急いで得るものが「自分の数秒」だけなら、lint の二本立て運用みたいな暫定対応を抱えるより、周辺が揃ってから一気に上げた方が総コストは安い。そういう計算です。

エンドユーザーに価値が出ない変更は、慌てて入れる理由もない。逆に言えば、開発者の待ち時間が既にストレスになっている大規模プロジェクトなら、今日入れる価値が十分あります。そこは各自の痛みの大きさで決めればいいと思います。

## まとめ

- TypeScript 7.0 は 2026 年 7 月 8 日に GA。コンパイラが Go ネイティブになり、型チェックは約 10 倍、メモリは半減
- 速くなるのは開発者のビルド・型チェック・エディタ。エンドユーザーへの影響はゼロ（出力される JS は同じ）
- 制限は「安定 API 未公開（7.1 待ち）」「Vue 系未対応」「emit は ES2021 まで」の 3 つ。noEmit 構成なら実質ノーダメージ
- 自分の方針は、新規は最初から 7、既存は 3 か月後にまとめて移行

3 か月後の自分がこの記事を読み返して答え合わせをする予定です。カレンダーに入れました。

### 一次情報

- [Announcing TypeScript 7.0（GA・2026-07-08）](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)
- [Announcing TypeScript 7.0 RC](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-rc/)
- [Announcing TypeScript 7.0 Beta](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-beta/)
- [Progress on TypeScript 7 - December 2025（互換性の詳細）](https://devblogs.microsoft.com/typescript/progress-on-typescript-7-december-2025/)
- [A 10x Faster TypeScript（2025-03 の移植発表）](https://devblogs.microsoft.com/typescript/typescript-native-port/)
- [microsoft/typescript-go（GitHub）](https://github.com/microsoft/typescript-go)

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
