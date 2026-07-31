---
title: >-
  secret manager を自作するために、1Password と age と SOPS と OS
  標準の秘密保管を白書ベースで読み解いた（そして車輪の再発明を宣言する）
tags:
  - 1Password
  - Security
  - 暗号
  - CLI
  - Go
private: false
updated_at: '2026-07-31T22:29:43+09:00'
id: b999eff900b43a7a8b52
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![ヒーロー（記事トップ）](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-24_secret-manager-jisaku/2026-07-24_secret-manager-jisaku_hero.png)

![記事の要約（インフォグラフィック）](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-24_secret-manager-jisaku/2026-07-24_secret-manager-jisaku_infographic.png)

## 事故の日

Windows の開発機で AI CLI を 6 匹並行で動かしています。Claude Code、Codex、OpenCode、Cursor、Grok、Ollama。この 6 匹に、たまたま起きた事故があります。

VPS の `/etc/myapp/config/staging.toml` に `vision_api_key` が設定されているか確認したかっただけ。ただそれだけの用事だったのに、コマンドを雑に打ちました。

```bash
grep -iE 'vision|gemini|ai' /etc/myapp/config/staging.toml
```

`-iE 'vision|gemini|ai'`。この `ai` が余計でした。マッチした行がまるごと吐き出されて、`chatgpt_api_key = "sk-proj-..."` と `gemini_api_key = "AIzaSy..."` の生値がコンソールに、そして会話ログに、そのまま流れました。

今回は Claude Code だったので、流れた先は Anthropic の処理サーバー。信用境界の内側なので実害はゼロです。ただし。ただし他 5 匹の CLI で同じことが起きたら、xAI、Cursor、OpenCode、Ollama のそれぞれのプロバイダに秘密が届く。

それって、そもそも構造的に間違ってないか。

## 構造の話をしないと、また同じ事故を起こす

AI に「気を付けてね」と頼むアプローチは、6 CLI 併用の環境では信頼できません。各 CLI が違う挙動をするし、grep のスコープを自分で絞れないのは私だけの問題でもない。

AI エージェント自体が自律行動する時代に、ローカルディスク上に平文で秘密を置いておくのは、玄関マットの下に家の鍵を置いて「見ないでね」と AI にお願いしているのと同じです。仕組みでロックしないと。

そこで金庫を買う話になります。secret manager の話です。

## 選択肢を並べたら 8 個あった

社内・オフィス用途で名前がよく出るツールを並べてみます。個人開発者・Windows 環境・6 CLI 併用・秘密数 30 個以上、という要件で見ていきます。

- 1Password（`op` CLI）
- Bitwarden（`bw` CLI）
- Doppler（開発者向け環境変数管理サービス）
- Infisical（OSS の secret manager・self-host 可能）
- HashiCorp Vault（overkill 候補）
- KeePassXC + キーファイル運用
- Windows Credential Manager（OS 標準）
- SOPS + age（暗号化ファイルとしてコミット可能）

全部触ってから決めるのは無理なので、まずは技術的な仕組みを一次情報で読むことにしました。ここからが本題です。**車輪の再発明をやると決めているけど、まず車輪を分解して観察する**、という順序です。

## 1Password の中で何が起きているか

1Password の暗号化設計は Security Design White Paper に載っています。PDF ですが本文抽出は難しいので、サポート記事の方を主に読んでいきます。

- Security Whitepaper（PDF・存在確認済）: <https://1passwordstatic.com/files/security/1password-white-paper.pdf>
- About the 1Password security model: <https://support.1password.com/1password-security/>
- About your Secret Key: <https://support.1password.com/secret-key-security/>
- What the Secret Key does: <https://1password.com/blog/what-the-secret-key-does>

一番おもしろいのが **Two-Secret Key Derivation**。要は、ユーザーが覚えるアカウントパスワードと、デバイスに保存される Secret Key、この 2 つを組み合わせて暗号鍵を作ります。

- アカウントパスワード: 覚えられる程度の強度。エントロピーは 40 bit 前後（推測でこれくらい、というのが業界の相場）
- Secret Key: **34 文字**の英数字（ダッシュ区切り）。エントロピー **128 bit**。デバイスにしか保存されない、ユーザーが覚える必要のない鍵

この 2 つが両方揃わないと復号できません。パスワードだけでは足りない。Secret Key だけでも足りない。合わせて **128+ bit の実効鍵**。

なぜこんな面倒くさい構成にするのか。答えは「サーバー側総当たり攻撃への防御」です。1Password のサーバーが仮に侵害されて、暗号化された vault データが全部流出したとしても、攻撃者の手元にあるのは「弱いパスワードでハッシュされた鍵」だけではなくて、**そこに 128 bit の Secret Key を足したもの**が正解として要求される。これを総当たりで破るのは物理的に無理、という設計です。

Secret Key の先頭 8 文字だけ、サポート用途で 1Password 側も持っています（バージョン識別子 + アカウント識別子）。残り 26 文字は端末にしかない。パスワード忘れたときに「復旧」できないのは、この構造の必然です。

続いて **SRP（Secure Remote Password）**。ログイン時にサーバーへ送るのは、パスワードそのものではなく検証子です。サーバーはパスワードを知らないまま「あなたのパスワードは正しい」を判定できる。PAKE（Password-Authenticated Key Exchange）と呼ばれる分野で、TLS 応用版が RFC 5054 にあります（SRP-6 系のプロトコル原典は Stanford の Tom Wu 論文と RFC 2945 系。5054 は TLS 応用側）。

- RFC 5054 Using SRP for TLS Authentication: <https://datatracker.ietf.org/doc/html/rfc5054>

そして **鍵派生**。1Password のサポート記事では PBKDF2 による key strengthening を明記しています。Argon2 への切替は現時点で明言されていない（サポート記事レベルでは）。Bitwarden は既に PBKDF2-SHA256 のデフォルト iteration を 600,000 回まで上げていて、Argon2id も選択可にしています。

- Bitwarden Security Whitepaper: <https://bitwarden.com/help/bitwarden-security-white-paper/>

要するに 1Password は「認証（SRP）」「鍵派生（PBKDF2）」「暗号化（AES-GCM 256）」の 3 層を独立に組んでいて、それぞれが単独で破られても他が残っている、という多重防御になっている。この設計思想は、あとで自作のリファレンスとしても効きます。

![1Password の Two-Secret Key Derivation 図解](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-24_secret-manager-jisaku/2026-07-24_secret-manager-jisaku_fig1.png)

上の図のとおり、アカウントパスワード（覚える・約 40 bit）と Secret Key（デバイス側・128 bit）の 2 つが両方揃わないと復号できません。サーバー側が侵害されても Secret Key を持たない攻撃者は復号不能、というのがこの設計の要諦です。

## age は「複雑さを断つ」という設計判断

秘密ファイルを暗号化する専用ツール、age。作者は Filippo Valsorda（元 Cloudflare、Go チームの暗号専門家）です。

- 公式サイト: <https://age-encryption.org/>（v1 の spec は `/v1` パス）
- リポジトリ: <https://github.com/FiloSottile/age>
- 作者ブログ「age and Authenticated Encryption」: <https://words.filippo.io/dispatches/age-authentication/>

age が面白いのは、**GPG に対する「引き算」**として設計されている点。GPG の `-c`（パスワード暗号化）と `-e`（公開鍵暗号化）だけを再設計して、他は全部削っています。署名機能すら意図的に外している。「Confidentiality と Integrity のみ、Authentication は範囲外」と明言している。

プリミティブはこんな感じ。

- 鍵合意: X25519
- 対称暗号: ChaCha20-Poly1305（AEAD）
- 鍵派生: HKDF-SHA256
- パスフレーズ用: scrypt
- チャンク: 64 KiB の **STREAM** スキームで seek 可能

一次資料。

- RFC 8439 ChaCha20-Poly1305: <https://datatracker.ietf.org/doc/html/rfc8439>
- RFC 7748 X25519 / X448: <https://datatracker.ietf.org/doc/html/rfc7748>
- RFC 5869 HKDF: <https://datatracker.ietf.org/doc/html/rfc5869>
- RFC 9106 Argon2（age は使わないが、業界の現行標準として）: <https://datatracker.ietf.org/doc/html/rfc9106>

作者の設計哲学が README に短く書かれています。「cryptographic tools work best when they are specialized and opinionated」。設定オプションを持たない。UNIX パイプで扱える。鍵はコピペ可能な短い文字列。

![age の 4 プリミティブ構成図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-24_secret-manager-jisaku/2026-07-24_secret-manager-jisaku_fig2.png)

上の図のとおり、age は鍵合意（X25519）／ AEAD（ChaCha20-Poly1305）／ 鍵派生（HKDF）／ パスフレーズ（scrypt）の 4 つだけを使っています。「必要最小限に絞る」という判断が仕様の骨格そのものになっている、という珍しい設計です。

**「複雑さは脆弱性を呼ぶ」という判断を、実装ではなく設計段階で下している**。これは真似できる思想だなと感じました。自作するときの一番大きな指針になります。

なお age には Rust 実装の rage（`github.com/str4d/rage`）もあります。フォークではなく仕様互換の代替実装。同じ暗号ファイルをどちらでも扱える。この関係は TLS が OpenSSL / rustls / BoringSSL などで並列実装されているのと同じで、「仕様 1 個・実装複数」の教科書的な形になっています。

## SOPS は「構造は見えて値は隠れる」

age は 1 ファイル暗号化ツールでした。SOPS はその上に vault 的な UX を乗せます。

- リポジトリ: <https://github.com/getsops/sops>
- 公式ホーム: <https://getsops.io/>

SOPS の設計で一番好きなのは **部分暗号化**の考え方です。

> Keys are not encrypted, while values and comments are encrypted.

たとえば `.env` ファイルを SOPS で暗号化すると、こういう形になります。

```yaml
GEMINI_API_KEY: ENC[AES256_GCM,data:...,iv:...,tag:...]
DB_PASSWORD: ENC[AES256_GCM,data:...,iv:...,tag:...]
DEBUG: "false"
```

キー名（`GEMINI_API_KEY` / `DB_PASSWORD`）は平文。値だけが暗号化される。だから git diff で「新しいキーが追加された」「キー名が変わった」を追跡できる。値の中身は当然読めないけれど、構造の変化は見える。

これはリポジトリにコミット可能な secret 管理として絶妙な設計です。全部を単一の暗号バイナリにしてしまうと、コミットしても diff が「バイナリファイルが変わった」しか出ない。部分暗号化は、その粒度を思い切って値だけに切ることで、「暗号化しつつ git 管理可能」という一見矛盾する要件を両立させています。

バックエンドは age / GPG（オフライン）と AWS KMS / GCP KMS / Azure Key Vault / HashiCorp Vault / OpenBAO（オンライン）の複数対応。CNCF Sandbox project になっています（2015 年 Mozilla 発 → 2023 年 CNCF 寄贈）。

## OS 標準の秘密保管、意外と使える

秘密の保管に、実は各 OS が公式で用意している仕組みがあります。忘れがち。

### Windows: Credential Manager と DPAPI

- wincred.h（Credential Manager API）: <https://learn.microsoft.com/en-us/windows/win32/api/wincred/>
- dpapi.h（Data Protection API）: <https://learn.microsoft.com/en-us/windows/win32/api/dpapi/>

Windows には 2 段構えがあります。

- **Credential Manager**: ユーザーの credential セットに対する CRUD。`CredWriteW` / `CredReadW` / `CredEnumerateW` / `CredDeleteW`。CLI では `cmdkey`
- **DPAPI**: 現在の security context 配下でしか復号できない鍵管理。`CryptProtectData` / `CryptUnprotectData` で永続データを、`CryptProtectMemory` / `CryptUnprotectMemory` でプロセス内メモリを保護。Credential Manager が内部で DPAPI を利用している

PowerShell には `Microsoft.PowerShell.SecretManagement` / `SecretStore` モジュールもあります。追加インストール不要で、`Get-Secret <name>` で取り出せる。

ここで大事なのは「**Bash tool から Credential Manager は直接読めない**」ということ。AI CLI 隔離の一次防壁として、これは即日ゼロコストで導入可能です。うっかり自分の AI が cat したり grep したりする事故に対する免疫になる。

### macOS: Keychain と Secure Enclave

- Keychain Services: <https://developer.apple.com/documentation/security/keychain_services>

macOS の Keychain は Security framework 配下で、パスワード・鍵・証明書・セキュアノートを暗号化ストアで管理します。

面白いのが Secure Enclave 連携。`kSecAttrTokenIDSecureEnclave` を指定すると、鍵が **Secure Enclave 内に生成・保持されて、外部にエクスポート不可**になります。Touch ID / Face ID unlock は `SecAccessControlCreateWithFlags` に `.biometryCurrentSet` や `.userPresence` を指定して要求する仕組み。

iCloud Keychain の同期は End-to-End 暗号。設計思想は 1Password の Two-Secret Key Derivation にも通じるものがあります（Apple ID のパスワード + デバイス側の鍵、という多要素の組み合わせ）。

### Linux: Secret Service API

- Secret Service specification: <https://specifications.freedesktop.org/secret-service/latest/>

Linux はデスクトップ環境の断片化があるので、GNOME Keyring と KDE KWallet が共同で D-Bus API を策定しています。データモデルは `Collection` / `Item` / `Session` の 3 階層。

秘密の転送アルゴリズムは `plain` と `dh-ietf1024-sha256-aes128-cbc-pkcs7`（Diffie-Hellman でセッション鍵を確立して AES-CBC で転送）。クライアント側は `libsecret` が事実上の標準ライブラリになっています。

3 OS で API はバラバラですが、思想は共通しています。「秘密をアプリケーション自身で持たず、OS の秘密保管サービスに預ける」。この抽象化を跨いで動く CLI を書くと、それだけで結構な数の書式変換ロジックが要ります。

## じゃあ既存で足りるのでは？の逡巡

さて。ここまで書いておいてなんですが、真面目に考えると、**答えは「1Password 買え」で終わる**んですよね。

- 個人プラン: 年払い時 $2.99/月（年 $35.88・約 5,400 円）、月払い時 $3.99/月（<https://1password.com/personal>・2026 年時点）
- Windows / macOS / Linux 全対応
- SSH agent 統合、PowerShell 統合、`op run --env-file=.env.tmpl -- <cmd>` でプロセス起動時に環境変数注入
- ブラウザ拡張、生体認証、多デバイス同期
- Secret Key + Master Password の Two-Secret 設計で、サーバー側総当たり不能

移行工数は 8〜12 時間と見積もれる。年 5,400 円の保険料で AI CLI 6 匹からの実値到達を構造的に遮断できる。**費用対効果的には完璧**です。

無料でいいなら SOPS + age。単一暗号鍵の管理さえ気を付ければ、git 管理可能な `.env` を暗号化してリポジトリに置ける。工数は 4 時間くらいで済む。

Windows Credential Manager だけでも足りる場面もあります。無料・即日・追加ソフト不要・Bash から読めない。3 時間で移行できる。

**要するに、選択肢は全部揃っていて、どれを選んでも合理的に「勝ち」なんです**。

じゃあ、なぜ自作するのか。

## それでも作る。理由は「作りたいから」

理由は……ないです。

いや、ないというのは嘘で、いくつかあります。

1. 仕組みが分かるから、自分で作りたい。読んで分かったつもりになっているのを、コードで確かめたい
2. 個人 OSS として置いておくと、他人にも配れる形になる（そしてバグ報告されて成長する）
3. Go / Rust の暗号ライブラリの触り心地を知りたい（age が Go 参照実装なので、Go で `filippo.io/age` を import する題材として丁度いい）
4. 「AI CLI 隔離」という現代的な要件に、既存ツールが微妙に噛み合わない箇所がある気がする（op は個人向けだけど CLI 統合の細部は場面によって重い、SOPS は複数バックエンド対応が個人には過剰、Credential Manager は Linux VPS には無い）
5. 単純に手作りしたい

正直、5 が本音です。他は後付けです。

暗号自体は絶対に自作しません。`filippo.io/age` をライブラリとして呼ぶだけの、薄いラッパーを書きます。これなら「暗号のバグで秘密が漏れる」リスクは age 側に閉じ込められる。自分が書くのは CLI UX、鍵管理、OS 別の統合層、環境変数注入。それくらいなら、暗号のバグで死ぬ心配はほぼゼロです。

「半年〜1 年バグ出しをしてから本番投入」も過剰でした。冷静に故障モードを分類すると、こうなります。

- 機能バグ（暗号化はできたが復号できない）: 初回使用で発覚。数日で潰れる
- データ破損バグ（vault ファイルが壊れる）: バックアップ運用してれば実害ゼロ
- 暗号アルゴリズムの実装ミス: age 側のコードが動くので発生しない
- OpSec ミス（一時ファイルに平文が残る、swap にダンプ）: 実装時のテストで拾える

つまり、**round-trip テストを書いて、元の `.env` を 1 週間バックアップに残して、並行運用すれば十分**。長期の burn-in は不要。

## 何を作るか、を 1 段階だけ書いておく

車輪の再発明とはいえ、目指す形は明確にしておきたい。

- 名前: `kinko`（仮）。日本語「金庫」から。crates / npm / PyPI / Homebrew / Chocolatey / Snap ぜんぶ空きなのを実測確認済み
- 言語: Go。`filippo.io/age` を import できる、本家踏襲、単一 .exe 配布、Windows Credential Manager 統合ライブラリあり
- スコープ:
  - `kinko init` : 新規 vault を作る（マスターパスワード or 生体認証で暗号化）
  - `kinko add <path> <value>` : 秘密を追加
  - `kinko get <path>` : 秘密を標準出力
  - `kinko run --env-file <template> -- <cmd>` : `.env` テンプレの `${kinko:path}` を実値に展開してプロセス起動（1Password の `op run` 相当）
  - `kinko list` : パスの一覧（値は出さない）
  - `kinko export` : 別 vault へエクスポート（バックアップ用）
- 暗号: age に丸投げ
- 鍵保管: 初期はマスターパスワード、あとで Windows Credential Manager 統合
- 配布: GitHub Releases で `.exe` 公開

![kinko run --env-file のデータフロー](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-07-24_secret-manager-jisaku/2026-07-24_secret-manager-jisaku_fig3.png)

上の図のとおり、ディスク上には age で暗号化された vault と、参照名だけの env テンプレしか置きません。実値は `kinko run` 起動時にプロセスのメモリだけへ注入されて、プロセス終了とともに消えます。AI CLI が cat / grep しても、ディスク側からは実値に到達できない、という発想です。

実装編は次回に書きます。この記事は「なぜ既存ツールで足りるはずなのに、それでも自作するのか」を白書ベースで自分に説明するために書いたものなので、コードは 1 行も出していません。

## この記事で押さえたかったこと

- 1Password は Two-Secret Key Derivation で「パスワードだけでは復元できない」設計にしている。Secret Key は 34 文字 / 128 bit エントロピー（<https://support.1password.com/secret-key-security/>）
- age は「複雑さを断つ」思想の教科書。GPG の `-c` と `-e` だけ再設計、署名は範囲外と明言
- SOPS は「キー名は平文・値だけ暗号化」で git diff と両立する部分暗号化
- OS 標準の秘密保管（Windows Credential Manager / macOS Keychain / Linux Secret Service）は無料で使える一次防壁。AI CLI 隔離の即日策としてはこれで足りる場面もある
- 費用対効果だけ考えると 1Password 買うのが正解。それでも自作する理由は「作りたいから」で通す

「作りたいから作る」という理由は、大人の説明としてはアホなんです。でも、そのアホさが原動力になるのを 20 年近いエンジニア人生で何度も経験しています。**車輪を分解して仕組を理解して、それでも自作するというアホさが、たまに新しい車輪を作る**。今回はそこまで行かないかもしれないけれど、まず作ってみます。

次回は Go で `kinko` の骨格を書いていきます。`filippo.io/age` の import から始めて、round-trip テストが通るまで。実装編でお会いしましょう。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-07-31_secret-manager-from-scratch/

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi

Web エンジニア 18 年目。バックエンド・インフラ・AI 連携が主戦場です。

- ポートフォリオ: <https://ishizakahiroshi.com/>
- GitHub: <https://github.com/ishizakahiroshi>
- X: <https://x.com/ishizakahiroshi>
