---
title: "自作の小さな金庫に、パスキー管理まで載せられるか。1Password・Bitwarden・KeePassXC と比べて、立ち位置を決める前に調べたこと"
tags:
  - セキュリティ
  - パスキー
  - 1Password
  - Bitwarden
  - Go
private: false
updated_at: ''
id: ''
organization_url_name: ''
slide: false
ignorePublish: false
---

![ヒーロー：机の上の小さな金庫と閉じたノートPC、セキュリティキー](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-10-30_kinko-passkey-position/01_2026-10-30_kinko-passkey-position_hero.png)

先に、結論を3行で書きます。

- 自作の金庫 kinko にパスキー管理を載せることは、技術的にはできそうです。ただ、4 桁の PIN は消えません。パスキーの本人確認を受け持つのは Windows Hello だからです。
- パスキーの保管とブラウザの自動入力は、無料の Bitwarden や KeePassXC にもうあります。kinko がどこまで広げるかは、まだ決めていません。
- PIN が面倒な原因は、kinko ではなく手元の環境にありそうです。本人確認をする Windows が仮想マシンの中にあり、指紋も顔認証も使えない場所だったからです。

![インフォグラフィック：パスキー管理を載せても PIN は消えない。保管と自動入力は無料にある。PIN の原因は手元の環境](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-10-30_kinko-passkey-position/02_2026-10-30_kinko-passkey-position_infographic.png)

パスキーでログインするたびに、4 桁の PIN を打っています。Chrome のパスキーでも Microsoft のものでも同じで、毎回ほんの少しだけ面倒です。

最初は軽い質問でした。使っていない USB メモリにパスキーを入れて、挿すだけでログインできないか。そこから話がふくらんで、自作の金庫 kinko にパスワード管理の仕事まで任せたくなり、何が足りないのかを調べました。調べものは Claude Code に手伝ってもらいました。

自作で [kinko](https://github.com/ishizakahiroshi/kinko) という秘密管理 CLI を作っています。秘密は暗号化して、必要なひとつだけ。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=kinko
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/kinko

同じ悩みを持っている方は、下記で入ります。AI CLI や検索コマンドが設定ファイルを丸ごと読んだときに、関係のない秘密まで画面やログへ流れる事故を減らすための道具です。

```sh
go install github.com/ishizakahiroshi/kinko@latest
```

入れたら、保管庫を作って、秘密を 1 件足して、環境変数として子プロセスにだけ渡せます。

```sh
kinko init
kinko add example-service/api-token
kinko run --env-file .env.tmpl -- your-command --flag
```

現在は prerelease です。バージョン付きの配布物はまだなく、端末解除（Windows Hello、macOS のキーチェーン、Linux の Secret Service）も実機での確認が終わっていないので、対応済みとは書きません。

この話には前があります。金庫を自作する前に、1Password と age と SOPS と OS 標準の秘密保管を一次資料で読み比べて、それでも作ると宣言した記事です。

前回の記事: [secret manager を自作するために、1Password と age と SOPS と OS 標準の秘密保管を白書ベースで読み解いた（そして車輪の再発明を宣言する）](https://qiita.com/ishizakahiroshi/items/b999eff900b43a7a8b52)

## そもそも kinko（自作の金庫）は何をするものか

kinko は、開発用の秘密を暗号化してしまっておき、必要なときに名前で 1 件ずつ取り出す、Go 製のコマンドラインの道具です。名前は日本語の「金庫」から取りました。

出発点は、前回の記事に書いた事故です。AI CLI を並行で動かしている環境で、確かめたかったのは 1 つの設定だけなのに、検索の絞り込みが広すぎて、API キーの生値が画面と会話ログまで流れました。AI CLI や検索コマンドが設定ファイルを丸ごと読めること自体が、事故の前提です。だったら、実値の入った設定ファイルを置かない形にすればいい。そういう発想です。

仕組みは 3 つです。

1. 保管庫を、age のパスフレーズ暗号化で 1 つのファイルにする。秘密の値だけでなく、名前も暗号文の中に入れるので、どのサービスを使っているかも、ファイルを読んだだけでは分からない
2. 取り出すときは、名前を指定して 1 件だけ。`kinko list` は名前だけを返し、値は出さない
3. コマンドに渡すときは `kinko run`。参照名だけを書いたテンプレートを読み、実値は、起動する子プロセスの環境変数にだけ渡す。実値を埋めた `.env` は作らない

テンプレートは、こう書きます。実値を含まないので、リポジトリに入れても構いません。

```dotenv
API_TOKEN=${kinko:example-service/api-token}
APP_MODE=development
```

`kinko run` の流れは、こうなっています（リポジトリの `docs/reference_architecture.md` の図を、英語のラベルのまま写しました）。

```text
vault.age                 .env.tmpl
(age encrypted)           (names only)
    |                         |
    | decrypt                 | read
    +-----------+-------------+
                |
          kinko run (memory)
          expand ${kinko:name}
                |
                | only the needed values
                | are added to the environment
                v
          child process
```

ディスクに置くのは、暗号化した保管庫と、参照名だけのテンプレートです。実値は、`kinko run` が動いている間のメモリと、子プロセスの環境にしかありません。暗号そのものは自作せず、Go の age ライブラリに任せています。

限界も先に書いておきます。渡した秘密は、子プロセスからは見えます。子プロセスが環境変数を表示したり、外へ送ったりすれば漏れます。侵害された OS や、悪意のあるプログラムから守る道具ではありません。パスワードを忘れたら復旧できないので、`kinko export` で別のパスワードのバックアップを作り、別の場所に置く運用です。

つまり kinko は、「AI に設定ファイルを丸ごと読まれても、関係のない秘密まで流れない」ための、狭い道具です。1Password のようなパスワード管理アプリの代わりを、最初から目指したものではありません。この記事の問いは、その狭い道具を、どこまで広げられるか、です。

## USB メモリに、パスキーは焼けなかった

最初の質問は、これでした。使っていない USB メモリにパスキーを入れて、挿すだけで自動ログインできないか。

結論は、できません。USB メモリはデータを置く場所でしかなく、ログインのたびに鍵で署名する計算をする部品がありません。署名の計算は、鍵を持っている側（端末やセキュリティキー）が行います。署名する部品が入った専用の USB は、セキュリティキーという別の製品です。そのセキュリティキーの鍵は、機器の外に出ない作りになっています。

しかも、挿しただけの自動ログインにもなりません。パスキーのログインには、触る、指紋を当てるといった操作が挟まる作りです。「触るだけ」までは近づけても、「何もしない」にはなりません。

kinko にも似た案がありました。普通の USB メモリを鍵ファイルにして、金庫を開ける案です。いまは保留にしています。鍵ファイルはコピーできるので、コピーされたら合鍵になる。パスキーの代わりにはなりません。

![挿絵：中身が空っぽの USB メモリと、署名の部品が入ったセキュリティキー](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-10-30_kinko-passkey-position/03_2026-10-30_kinko-passkey-position_illustration-usb.png)

左の普通の USB メモリは、データを置くだけの箱です。右のセキュリティキーには署名の部品が入っていて、鍵が外に出ません。見た目が似ていても、中身の役目がまるで違います。

## そもそもパスキーとは何か。パスワードと mTLS と並べてみる

ここまで「鍵」「署名」「本人確認」と書いてきましたが、パスキーそのものの説明を飛ばしていました。先に基本を並べておきます。

パスキーは、FIDO2（WebAuthn と CTAP）にもとづく認証の仕組みです。パスワードの代わりに、公開鍵暗号を使います。流れは次のとおりです。

1. サイトごとに、端末の中で秘密鍵と公開鍵のペアを作る
2. サイトには公開鍵だけを渡す。秘密鍵は端末の中に置く
3. ログインのときは、サイトが出した問題（チャレンジ）に、端末が秘密鍵で署名して返す
4. サイトは、持っている公開鍵で署名を確かめる

サイトが持つのは公開鍵だけです。サイトから漏れても、そのままではログインに使えません。WebAuthn の仕様には、秘密鍵は鍵を管理する装置の外に出ない前提で、持ち主にさえ見せない、と書いてあります。しかも鍵は、作ったサイトにひもづきます。偽サイトには使えないので、フィッシングに強い、というのが設計の狙いです。

- FIDO Alliance のパスキーの説明: https://fidoalliance.org/passkeys/
- passkeys.dev の入門: https://passkeys.dev/docs/intro/what-are-passkeys/
- W3C の WebAuthn 仕様（秘密鍵と RP ID の記述）: https://www.w3.org/TR/webauthn-3/

この「秘密鍵で署名して証明する」形は、mTLS と同じ系統です。mTLS は、通信（TLS）の握手の中で、サーバーだけでなくクライアントも証明書を出して、お互いを確かめる方式です。TLS 1.3 の仕様（RFC 8446）では、サーバーが求めたときにクライアントが証明書を送り、その証明書に対応する秘密鍵で握手全体に署名して、持ち主であることを示します。API や機器同士の認証でよく使われます。

並べると、違いはこの辺りです。

| | パスワード | パスキー | mTLS |
|---|---|---|---|
| 何で証明するか | 知っている文字列を送る | 端末の秘密鍵で署名する | 証明書の秘密鍵で署名する |
| サーバーが持つもの | 確かめるための情報 | 公開鍵 | クライアント証明書を検証する設定 |
| 働く場所 | ログイン画面 | ログイン画面（WebAuthn） | 通信の握手（TLS） |
| 主な相手 | 人 | 人 | 機器・サービス |
| 運用で重いところ | 使い回しと漏えい | 端末側の管理 | 証明書の発行・更新・失効 |

- TLS 1.3（RFC 8446）: https://datatracker.ietf.org/doc/html/rfc8446
- mTLS の運用の重さ（ベンダーの解説記事）: https://apisix.apache.org/learning-center/what-is-mutual-tls/

パスキーの話に戻ります。ログインのたびに出る PIN や指紋は、サイトに送るものではありません。端末の中で、この人が秘密鍵を使ってよいかを確かめるためのものです。FIDO Alliance の説明では、サイトに伝わるのは確認が成功したという結果だけで、生体情報は送られません。持っているもの（端末の秘密鍵）に、知っているもの（PIN）か、本人であること（指紋・顔）を重ねている形です。

サイトが本人確認まで求めたときは、指紋・顔か PIN が必要になります。求めないサイトでは、仕様上、そこに人がいるかを確かめる程度の操作で済む場合があります。WebAuthn の仕様は、この確認の操作に、存在の確認と本人確認の両方があり得る、と書いています。

パスキーの置き場所は、大きく 2 通りです。Chrome や Microsoft のアカウントで端末をまたいで使う同期型と、セキュリティキーのような機器に固定されるものです。同期型でも、同期は暗号化されると FIDO Alliance は説明しています。

これで前の節がつながります。USB メモリで足りなかったのは、「署名する部品」でした。そして PIN を聞くのは、パスキーの鍵を使ってよいかを確かめる、端末側の仕事です。サイトに任せる仕事ではありません。Windows のプラグインの仕組みでは、この確認を Windows Hello が受け持ちます。

## 1Password は有料、無料で足りるものはもうあった

kinko を作った動機のひとつは、正直に言うと、1Password が有料だからです。個人プランは月 $3.99 で、新規なら初年度は月 $2.99（年払い）。無料枠の記載はなく、14 日間の試用があるだけでした。2026-10-30 に見た価格ページの数字です。

- 1Password の料金: https://1password.com/pricing/password-manager

そこで、無料で足りるものを先に見ました。

- Bitwarden は無料プランが「Always free」で、パスキー管理とデバイス無制限が含まれます。内蔵の認証コード（ワンタイムコード）の生成は、有料の Premium（年 $19.80）の項目に載っていて、無料側には載っていませんでした。https://bitwarden.com/pricing/
- KeePassXC は GPLv3 の無料ソフトです。暗号化したファイルを好きな場所に置く方式で、クラウドも課金もありません。パスキーには 2024 年 3 月の 2.7.7 から対応しています。https://keepassxc.org/ と https://keepassxc.org/blog/2024-03-10-2.7.7-released/

ブラウザ連携も、どちらにもあります。KeePassXC は専用の拡張機能（ユーザーガイドの Browser Integration）、Bitwarden はブラウザ拡張でパスキーを保存して、サイトのログインに使えます。つまり、パスキーの保管も、ブラウザの自動入力も、お金をかけずに今すぐ使える状態でした。

- KeePassXC のユーザーガイド: https://keepassxc.org/docs/KeePassXC_UserGuide
- Bitwarden のヘルプ（パスキーの保存と利用）: https://bitwarden.com/help/storing-passkeys/

kinko にそこまで載せる意味があるのか、まず考え込みました。

## 載せるなら、何が要るのか。小さい順に並べた

それでも、載せるとしたら何が要るのかを並べてみました。

いまの kinko の保管庫の中身は、名前と文字列の対応表です。暗号化する前の形は、こうなっています（リポジトリの `docs/reference_architecture.md` にある説明のままです）。

```json
{
  "version": 1,
  "vault_id": "16 bytes as lowercase hex, optional for legacy format 1",
  "secrets": {
    "example-service/api-token": "<secret value>"
  }
}
```

値は文字列だけです。パスキーのように構造のあるデータを入れるなら、形式のバージョンを上げる作業が先に要ります。

足すものを、小さい順に並べるとこうなりました。

1. ログイン情報（ID、パスワード、メモ）の項目。いまの形式の延長で作れそうです。
2. ブラウザの自動入力。ブラウザの拡張機能と kinko をつなぐ部品が要ります。
3. パスキー。Windows 11 には、第三者のパスキー管理アプリを登録する仕組み（プラグイン）があります。
4. 複数端末の同期とスマホ。いまの kinko は、同期も画面も最初から対象外にしています。

3 の中身が、思ったより重いものでした。Microsoft のサンプルは C++ で、アプリを Windows が呼び出す部品（COM）として登録します。`IPluginAuthenticator` を実装し、`WebAuthNPluginAddAuthenticator` で登録する形です。動かすには Windows 11 の 24H2 か 25H2 の、新しめの更新が要ります。しかも Microsoft のデモには、パスキーの作成と利用を試すためだけのもので、本番のパスキーには使わないように、と注意書きがありました。

- Microsoft のパスキー連携の資料: https://learn.microsoft.com/en-us/windows/apps/develop/security/third-party
- Microsoft のサンプル（Passkey Manager）: https://github.com/microsoft/Windows-classic-samples/tree/main/Samples/PasskeyManager

kinko は Go 製の CLI です。ここに 3 を足すなら、登録と確認の画面を持つ別のアプリが要ります。もう、ほぼ別の製品でした。

![図：kinko に足すものを小さい順に並べた4段](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-10-30_kinko-passkey-position/04_2026-10-30_kinko-passkey-position_fig1.png)

上の図のとおり、足すものは 4 段あります。ログイン情報の保存までは今の形式の延長で、自動入力から先は別の部品、パスキーは別のアプリ、同期は最初から対象外です。段が上がるほど、kinko の外に作るものが増えます。

## 載せても、PIN は消えない

ここが、一番予想と違ったところです。

Microsoft のサンプルは、パスキーを作るときも使うときも、本人確認を Windows Hello に任せています。呼び出しは `WebAuthNPluginPerformUserVerification` で、手順にも、Windows Hello で本人確認を完了する、と書いてあります。つまり、kinko 製のパスキー管理アプリを作っても、4 桁の PIN を聞くのは Windows Hello のままです。これは資料とサンプルから読み取った範囲で、実際に動かして確かめたわけではありません。

皮肉なことに、kinko 自身の端末解除も Windows Hello を使います。指紋や顔の機器がない機械では、Hello は PIN で本人確認します。同じ PIN を聞かれる見込みです。

ソフトだけで PIN を減らす道として、Bitwarden のブラウザ拡張でパスキーを使う手も考えました。ただ、Bitwarden の担当者が、いまの実装は金庫の解除方法と本人確認を結びつけている、と書いているのを見つけました。ヘルプにも、本人確認は Bitwarden のロック解除か、端末の PIN・パターン・生体認証になることがある、と書いてあります。PIN なしで通るかどうかは確認できていません。

- Bitwarden のコミュニティの議論: https://community.bitwarden.com/t/passkey-user-verification-independent-of-vault-unlock-method/68375?page=2
- Bitwarden のヘルプ（パスキーの保存と利用）: https://bitwarden.com/help/storing-passkeys/

## 本当の原因は、仮想マシンの中で本人確認をしていたこと

ここで、PIN が面倒な本当の原因に目が向きました。kinko のせいではなく、手元の環境です。

この開発機は Microsoft の仮想マシンで、リモート デスクトップ経由で入っています。読み取りだけで確かめました。モデル名が「Virtual Machine」で、セッション名はリモート デスクトップのものでした。仮想マシンの中の Windows には、指紋リーダーも顔認証カメラもありません。だから、本人確認が PIN になっている可能性が高い。そう見ています。

手元の PC には Web カメラがあります。ただ、Windows Hello の顔認証には赤外線を扱える専用のカメラが要り、普通の Web カメラでは使えないそうです。複数の解説が一致していましたが、Microsoft の公式ページは見つけられませんでした。

- 顔認証に赤外線のカメラが要る件（解説）: https://evezone.evetech.co.za/quick-bytes/why-your-webcam-may-work-windows-hello-face

手元の機器を使う道はあります。リモート デスクトップには「WebAuthn リダイレクト」という機能があり、仮想マシンの中のパスキー要求を、手元の PC の Windows Hello やセキュリティキーに送れます。手元の PC に、指紋・顔の機器かセキュリティキーがあるのが条件です。

- WebAuthn リダイレクトの設定（Microsoft）: https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-webauthn

これを試すには、パスキーの登録画面が出るテスト用のサイトが 1 つ要ります。候補に挙がったのが webauthn.io でした。見た瞬間に「怖い」となりました。当たり前の反応です。素性を調べると、作ったのは Cisco 傘下の Duo Labs で、ソースコードも GitHub で公開されていました。Microsoft の公式資料も、動作テストの例としてこのサイトを挙げています。ただ、プライバシーポリシーがなく、保存される情報の全部は確認できませんでした。

- webauthn.io: https://webauthn.io/
- ソースコード: https://github.com/duo-labs/webauthn.io

結局、使いませんでした。先に、手元の PC に指紋・顔の機器があるかだけを調べる方を選んだからです。何も登録せずに済み、機器がなければ、リダイレクトを試す意味そのものがなくなります。答えは「Web カメラと Bluetooth しかない」でした。顔認証に使えるカメラではなさそうなので、リダイレクトが効いても、PIN を打つ場所が手元に移るだけになります。だから、リダイレクトは試していません。

## 「できない」を 3 つに分けたら、整理がついた

調べ終えたあと、「結局、技術的にできないってことか」と思いました。正直、そう言い切りたくなる結果でした。でも、それは雑でした。分けるとこうなります。

- 仕組み上、無理と言い切れるもの: 普通の USB メモリにパスキーを入れること。何も操作しない自動ログイン。普通の Web カメラでの顔認証。
- いまの機器では無理でも、機器を足せば可能なもの: 指紋でのログイン。指紋付きのセキュリティキーなどを足し、リダイレクトが効くことが条件です。
- まだ試していないもの: 指紋なしのセキュリティキーで「触るだけ」にする方法。Bitwarden で PIN なしにパスキーを通す方法。

最後の 2 つは、「できない」ではなく「試していない」です。ここを混ぜたまま「無理でした」と書くと、嘘になります。

## それでも kinko は広げるのか

まだ決めていません。ただ、決める前に壁の高さは並びました。

kinko が置かれている場所は、ログイン情報を扱う 1Password や Bitwarden とは少し違います。開発用の秘密を、AI に丸ごと読まれない形で置いて、1 件ずつ出す。1Password の `op run` と同じ用途です。その `op run` の公式ページにも、同じユーザーで動くほかのプロセスが環境変数を読めることがある、という注意が書いてあります。kinko の README にも同じ限界を書いています。侵害された OS や、悪意のある子プロセスから守る道具ではありません。

- 1Password の `op run`: https://www.1password.dev/cli/secrets-environment-variables/

kinko の README は、自分で「1Password、SOPS + age、OS 標準の秘密保管で要件を満たせるなら、それらを選ぶ方が適切です。」と書いています。ここは変わりません。足りないところが見つかったときだけ、kinko を足す。その順番は崩さないつもりです。

広げるなら、最初の一歩はログイン情報の保存になりそうです。パスキーは一番最後でいい、というのが今の見立てです。理由は、パスキーを足しても PIN が消えないことと、無料の既存品がもうあることの 2 つです。

## この記事で押さえたかったこと

- 作る前に、作っても解決しない問題かを確かめる。PIN は kinko では直せませんでした。
- 本人確認をする場所は、OS が決めます。パスキー管理アプリを作っても、そこは動かせません。
- 無料の既存品は、作る前に 1 度は見る。パスキーも自動入力も、無料側にありました。
- 「できない」は 3 つに分ける。仕組み上無理、機器を足せば可能、まだ試していない。

## kinko はこんなときに刺さります

- AI CLI や検索コマンドが設定ファイルを丸ごと読んでも、関係のない API キーまで出ないようにしたい人
- 平文の `.env` を置かずに、必要な 1 件だけをコマンドの環境変数へ渡したい人
- 秘密の名前も暗号化して、使っているサービス名まで保管庫のファイルから読めないようにしたい人

いずれかに心当たりがあれば、`go install github.com/ishizakahiroshi/kinko@latest` と `kinko init` の 2 コマンドで保管庫を作れます。`kinko run` で使うテンプレートには、実値ではなく参照名だけを書きます。

- 紹介ページ（機能の一覧）: https://ishizakahiroshi.com/work.html?id=kinko
- リポジトリ: https://github.com/ishizakahiroshi/kinko

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## おわりに

PIN は、いまのところ打ち続けています。手元の PC に指紋の機器を足すかどうかは、また今度にします。

kinko をどこまで広げるかも、決めていません。ただ、調べたおかげで、「やらないこと」の輪郭は前より濃くなりました。次に何か足すなら、小さく。何が足りないのかを確かめてから考えます。

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
