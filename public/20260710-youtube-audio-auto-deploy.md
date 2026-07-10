---
title: "NotebookLM の音声概要を YouTube へ自動投稿する。ffmpeg + YouTube Data API v3 の小さなパイプライン"
tags:
  - YouTube
  - NotebookLM
  - ffmpeg
  - Python
  - GoogleCloud
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![記事が、ラジオになる ヒーロー画像](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-10_youtube-audio-auto-deploy_hero.png)

![記事の要約](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-10_youtube-audio-auto-deploy_infographic.png)

技術記事を書き終えたあと、その md を NotebookLM に食わせると 17 分のラジオ風音声（音声概要 / Audio Overview）ができました。これが思ったより聞ける。じゃあ YouTube にも置きたい。でも毎回手でアップロードするのは絶対に続かない。

というわけで、**音声 m4a + 記事のヘッダー画像 1 枚 → YouTube 公開** までをコマンド 2 発にした話です。ffmpeg と YouTube Data API v3 の組み合わせで、スクリプトは合計 150 行くらい。OAuth まわりで 3 回ハマったので、そこも隠さず書きます。

## 結局、この 2 コマンドになりました

```powershell
# ① 静止画 1 枚 + 音声 → mp4（ffmpeg ラッパー）
& '.\audio-to-video.ps1' -Image hero.png -Audio voice.m4a -Out video.mp4

# ② YouTube へアップロード（初回だけブラウザ認証・以後は無人）
python upload_youtube.py --video video.mp4 --title "タイトル" `
  --description-file desc.txt --privacy public
```

実際に上がったのがこれです: https://youtu.be/Ei3EB3W4tvM

以下、それぞれの中身と、ハマった場所を順に。

## そもそも音声は YouTube に直接上げられない

最初の困りごとはここでした。YouTube が受けるのは動画だけで、m4a や mp3 は投稿できません。だから「絵を 1 枚貼った動画」に変換する工程が必要になります。

変換は ffmpeg 一発です。核心はこのオプション群でした。

```powershell
ffmpeg -y -loop 1 -framerate 1 -i hero.png -i voice.m4a `
  -c:v libx264 -tune stillimage -r 1 -pix_fmt yuv420p `
  -vf "scale=1280:720:force_original_aspect_ratio=decrease,pad=1280:720:(ow-iw)/2:(oh-ih)/2:color=white" `
  -c:a aac -b:a 192k -shortest -movflags +faststart out.mp4
```

- `-loop 1` + `-shortest`: 静止画をループさせ、音声が終わったら動画も終わる
- `-tune stillimage` + `-r 1`: 静止画向け最適化と 1fps 化。17 分の動画でも映像部分は 5MB 程度に収まる
- `-pix_fmt yuv420p`: これを忘れると再生できないプレイヤーが出る定番の罠
- `-movflags +faststart`: メタデータを先頭に移動（ストリーミング再生の立ち上がりが速くなる）

画像には記事のヘッダー画像をそのまま使います。動画の全編がその 1 枚なので、**サムネイル問題も同時に解決**するのが地味に気に入っています。カスタムサムネイルの設定 API を叩く必要がない。

![md から YouTube 公開までのパイプライン全体図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-10_youtube-audio-auto-deploy_fig1.png)

上の図が全体の流れです。記事 md を NotebookLM に渡して音声化するところだけが手動（Web UI 操作）で、そこから先の変換とアップロードは全部コマンドです。

## アップロード側は Python + YouTube Data API v3

アップロードは公式 API の王道構成です。

```python
from googleapiclient.http import MediaFileUpload

body = {
    "snippet": {
        "categoryId": "28",
        "title": args.title,
        "description": description,
        "defaultLanguage": "ja",
    },
    "status": {"privacyStatus": "public", "selfDeclaredMadeForKids": False},
}
media = MediaFileUpload(args.video, chunksize=-1, resumable=True)
request = youtube.videos().insert(part="snippet,status", body=body, media_body=media)
```

ポイントは認証トークンの永続化で、初回に `token.json` を保存しておけば 2 回目以降はブラウザなしで動きます。ここまでは各所の解説どおり。問題はその手前の Google Cloud 設定でした。

## OAuth 設定で 3 回ハマった

**1 つ目。「JSON をダウンロード」ボタンが反応しない。** OAuth クライアント作成直後のダイアログでダウンロードボタンを押しても無反応、という Google Cloud コンソールの不具合（らしきもの）を踏みました。しかもクライアントシークレットは**作成時に 1 回しか表示されない**仕様なので、ダイアログを閉じたら詰んだかに見えた。回避策は、クライアント詳細画面から**シークレットを再発行**することです。新しい値がその場で 1 回表示されるので、すぐコピーするか JSON を落とします。

**2 つ目。認証したら 403 access_denied。** OAuth 同意画面が「テスト」ステータスのアプリは、**テストユーザーに登録したアカウントしかログインできません**。自分しか使わないアプリでも、自分をテストユーザーに追加する一手間が必要でした。エラー画面には「デベロッパーに承認されたテスターのみがアクセスできます」と出ます。自分がデベロッパーなのに。

**3 つ目。これが一番見落としやすい。テストステータスのままだと、リフレッシュトークンが 7 日で失効します。** つまり週 1 でブラウザ認証をやり直すことになり、「無人で動く」が崩れる。対策は同意画面を**「本番にプッシュ」**することです。Google の審査を通していない状態でも本番化はでき、認証時に「このアプリは確認されていません」警告が出続けるだけで、トークンは失効しなくなります。自分専用ツールなら実害はありません。本番化の後に**トークンを一度作り直す**のも忘れずに（テスト期に発行したトークンは 7 日失効の性質を引き継ぎます）。

![OAuth 設定の 3 つの罠と回避策の対比図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-10_youtube-audio-auto-deploy_fig2.png)

図にするとこの 3 点です。どれも API のコードとは無関係の、コンソール設定側の罠でした。コードを書く時間より、この 3 つを抜ける時間の方が長かった。正直、参った。

もう 1 つ、未検証プロジェクト経由のアップロードは YouTube のポリシー上「非公開ロック」される場合があると知って身構えていたのですが、手元では public のまま通りました。ここは環境や時期で変わるかもしれないので、断定はしないでおきます。

## NotebookLM の動画概要は使わなかった

NotebookLM には音声だけでなく動画概要（Video Overview）を作る機能もあって、最初はそっちで済むのではと試しました。結果、不採用。スライドの組版自体は崩れないのですが、タイトルのフォントの野暮ったさが致命的で、チャンネルに並んだときに「どこかで見た汎用スライド」の顔になります。透かしも全編に乗っていて消せない。

記事・ブログ・動画で同じヘッダー画像が並ぶ方が「この人のコンテンツだ」と分かる。だから絵作りはこちらで握って、NotebookLM には音声だけ作ってもらう分担に落ち着きました。

## 学んだこと

- 音声コンテンツの YouTube 展開は「静止画 1 枚の動画化」で十分成立する。ffmpeg の `-loop 1 -tune stillimage -r 1` が核
- OAuth のハマりどころはコードではなくコンソール設定側に集中している。テストユーザー登録と本番プッシュの 2 つは先にやっておく
- テストステータスのトークン 7 日失効は「自動化したつもりが週 1 で止まる」形で効いてくる。無人運用したいなら本番化が前提
- 生成 AI に任せる部分と自分で握る部分の線引きは「ブランドの顔になるかどうか」で決めると迷わない

記事を書いたら音声ができて、コマンド 2 発で YouTube に並ぶ。この記事もたぶん、そうやって音声になります。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
