---
title: "WebアプリのAndroid化で起動直後に落ちるとき。Manifestと端末のAPKを確認する"
tags:
  - Android
  - TWA
  - CustomTabs
  - adb
  - 個人開発
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![WebからAndroidへ](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-27_android-web-pilot-crash/01_2026-09-27_android-pilot_hero.png)

WebアプリをAndroidアプリから開く試作を作りました。Android Browser HelperでTWAを使う構成です。ところが、端末に入ったアイコンを押すと、画面が出る前に落ちる。Chromeから同じサイトは普通に開けます。

原因はManifestのActivity宣言漏れでした。修正後も落ちた理由は、端末に古いAPKが残っていたこと。この二つを分けて確認するまでと、動いてから分かったCustom Tabsの仕組みを残します。

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-27_android-web-pilot-crash/02_2026-09-27_android-pilot_infographic.png)

## まず何を確認するとよいか

同じ症状なら、端末で動いている版とクラッシュログを先に確認すると、調べる場所を絞れます。次はPowerShellでの例です。アプリIDと端末IDは説明用なので、手元のものへ置き換えてください。

```powershell
$device = '<device-id>'
$appId = 'com.example.app'
adb -s $device shell dumpsys package $appId |
    Select-String 'versionCode=|versionName='
adb -s $device logcat -b crash -d
```

ログは他アプリの情報を含むこともあるため、共有するなら対象アプリの該当箇所に絞ります。端末操作の詳細は[Android公式のadbドキュメント](https://developer.android.com/tools/adb)にまとまっています。

題材は、自作の進路サービス「まなびマップ」です。通える高校を地図で見ながら、親子で決める。

- [何ができるかの紹介ページ](https://ishizakahiroshi.com/work.html?id=manabi-map)
- [リポジトリ。Starをいただけると励みになります](https://github.com/ishizakahiroshi/manabi-map)

Web版は[まなびマップ](https://manabi-map.app/)をブラウザで開き、地点を検索して使えます。今回のAndroid版は、ストア公開前の試作です。

以前、[サービス名とURLを分ける方針](https://ishizakahiroshi.com/articles/2026/2026-09-24_manabi-map-brand-url-split/)を書きました。そのWebサービスに、手元のAndroid端末から入れる入口を付けてみたのが今回です。

## サイトは開くのに、アプリは繰り返し停止する

検証に使ったのはRedmi 12 5G、Android 15。Android Browser Helperは2.7.3です。初版のAPKはインストールでき、ホーム画面にアイコンも出ました。でも押すと「繰り返し停止しています」と表示されます。

端末ログで見えた例外は、次のものでした。アプリ側の識別子は省略しています。

```text
IllegalArgumentException:
Component class ...ManageDataLauncherActivity does not exist
```

呼び出し元を追うと、`LauncherActivity`からサイト設定用ショートカットを扱う処理へ進み、`ManageDataLauncherActivity`の有効・無効を切り替えるところで失敗していました。

ライブラリにクラスが入っていても、アプリのManifestにActivityとして登録されていなければ足りません。今回使った版の実バイナリと、[Android Browser Helperの公式サンプル](https://github.com/GoogleChrome/android-browser-helper/blob/main/demos/twa-basic/src/main/AndroidManifest.xml)を突き合わせました。

修正の中心は次の部分です。既存の`application`へ属性とActivityを追加する抜粋で、このままManifest全体を置き換えるものではありません。URLは説明用です。

```xml
<application
    android:manageSpaceActivity="com.google.androidbrowserhelper.trusted.ManageDataLauncherActivity">
    <activity
        android:name="com.google.androidbrowserhelper.trusted.ManageDataLauncherActivity"
        android:exported="false">
        <meta-data
            android:name="android.support.customtabs.trusted.MANAGE_SPACE_URL"
            android:value="https://example.com/" />
    </activity>
</application>
```

修正後は、ソースXMLだけでなく、出来上がったAPKのManifestにも宣言が入っていることを確認しました。ビルドが通ることと、端末で起動できることは別でした。

## 修正版を作った。でも、端末では旧版が動いていた

修正版を用意しても、端末では同じ停止が続きました。ここでワイヤレスADBを使い、実際に入っている版を調べると、まだ初版でした。

![手元の修正版と端末の旧版](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-27_android-web-pilot-crash/03_2026-09-27_android-pilot_illustration-versions.png)

PCに修正版があることと、スマホがその版を実行していることは別です。インストール後のバージョンまで確認すると、修正そのものと配布の取り違えを分けられます。

USBケーブルが手元になかったので、端末のワイヤレスデバッグから接続しました。ペア設定用と接続用でポートが別になるため、画面に出た数字を混ぜずに使います。接続後は、同じ署名とアプリIDの修正版を上書きしました。

```powershell
adb -s $device install -r '.\app-fixed.apk'
adb -s $device shell dumpsys package $appId |
    Select-String 'versionCode=|versionName='
```

アプリのアンインストールやChromeのデータ消去は行っていません。上書き後に実効版を確認し、アイコンから開くと、高校ホーム画面が表示されました。起動後の短い観測範囲では、対象アプリのクラッシュも出ていません。

なお、これは長時間利用や全機能の合格を意味しません。ここで確認できたのは、修正版で起動してWeb画面に到達するところまでです。

## 開いたら、もうログインしていた

動いた画面を見ると、すでにログイン済みでした。端末のChromeで、先にWeb版へログインしていたためです。

今回の試作でWeb画面を描いていたのは、端末に入っているChromeでした。[Custom Tabsはブラウザの状態を共有する仕組み](https://developer.chrome.com/docs/android/custom-tabs)なので、同じブラウザ・同じオリジンの既存セッションを使えます。Chrome自体をAPKに詰め込んだわけではありません。

![アプリの入口とWebの役割](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-27_android-web-pilot-crash/04_2026-09-27_android-pilot_fig1.png)

Android側がURLを開き、ブラウザが既存のWeb画面を動かします。Webの配信、認証、データ保存は従来の役割を引き継ぐ構成です。

まなびマップでは、Web配信にCloudflare、認証と利用者データの保存にSupabaseを使っています。Google・LINE認証の入口もWeb側にあります。Android用の認証SDKへ丸ごと作り直す必要はありませんでした。

ただし、「ログイン済みの画面が開いた」と「Google・LINEで新しくログインし、アプリへ戻る試験に通った」は別です。後者や、保存・再起動・ログイン取消の確認は残っています。

## 今回は、まだ全画面TWAではない

実機で開いた画面にはURLバーがあり、Custom Tabsとして動いていました。TWAの所有確認はまだ整えていません。

[TWAではDigital Asset Linksでアプリとサイトの関係を確認します](https://developer.chrome.com/docs/android/trusted-web-activity)。対応ブラウザが確認できれば、ブラウザのUIを出さずにWeb画面を表示できます。ただし、Android側がブラウザのCookieやlocalStorageを直接読めるようになるわけではありません。

そして高校版には、サブドメインへ移す計画があります。先に正式なURLを決め、Webの認証・保存・移行を確かめ、そのURLに対してアプリとサイトの対応付けを仕上げる順にします。オリジンが変われば、今のログイン状態がそのまま移るとは限らないためです。

iPhoneは当面Web版で利用してもらう方針です。AndroidとWebを育て、継続的な収益で年会費や保守の負担を支えられるようになったら、iPhoneアプリも改めて考えます。

![試作から公開までの順番](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-09-27_android-web-pilot-crash/05_2026-09-27_android-pilot_fig2.png)

起動できた試作を足場に、URLの確定、Web側の検証、Androidの公開準備へ進みます。見た目を仕上げる前に、開く先と認証の戻り先を揃える方針です。

## まなびマップを試すなら

- 通える範囲の高校を、地図で見比べたい方
- 説明会や通学経路の感想を、学校ごとにメモしたい方
- 数字だけで決めず、情報の根拠も確認したい方

[Web版](https://manabi-map.app/)はブラウザから使えます。[紹介ページ](https://ishizakahiroshi.com/work.html?id=manabi-map)に概要を、[GitHubリポジトリ](https://github.com/ishizakahiroshi/manabi-map)にコードを置いています。Starをいただけると開発の励みになります。不便なところはIssueでも教えていただけるとうれしいです。

## あわせて読みたい

- [同じバージョンなら、同じ中身だと思っていた](https://ishizakahiroshi.com/articles/2026/2026-06-14_same-version-different-binary-4-channels/)。作った成果物と、配られている実物を照合する話です。
- [Cloudflare PagesのSPAフォールバックが308になる](https://qiita.com/ishizakahiroshi/items/0b4e0b146ec0cbeb00fe)。サーバーで修正しても、既存ブラウザでの見え方が変わらなかった記録です。

## おわりに

今回は、Web版をAndroidから開く入口まで作れました。起動しなかったときに確かめるべきものは、コード、出来上がったAPK、端末に入っている版の三つでした。

次は高校版のURLを決めるところからです。スマホで動いた手応えは得られたので、公開用の仕上げはWeb側の移行と順番を合わせて進めます。

---

※ ヘッダー画像とインフォグラフィックの絵は AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
