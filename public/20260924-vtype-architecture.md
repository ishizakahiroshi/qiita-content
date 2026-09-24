---
title: API キーなしの音声入力を Web Speech API で作る。Chrome 拡張（MV3）とデスクトップ版（Rust）を共通コアから分解した
tags:
  - 音声入力
  - chrome-extension
  - WebSpeechAPI
  - Rust
  - TypeScript
private: false
updated_at: '2026-09-24T10:14:32+09:00'
id: efa7d306ceeb7dd9b2d7
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/01_2026-09-24_vtype-architecture_hero.png)

9 月 19 日に空のリポジトリを作り、24 日にデスクトップ版の 0.1.0 を npm に出しました。そのあいだのコミットは 70 本です。

作ったのは vtype という**音声入力**のツールで、**Chrome 拡張**とデスクトップアプリの 2 つの形があります。どちらも、認識は Chrome に最初から入っている **Web Speech API** を借りています。だから API キーも、アカウントも要りません。vtype の側には、1 日何分という上限もありません。

この記事では、その中身を分解します。Chrome 拡張（Manifest V3・TypeScript）とデスクトップ版（Rust）のそれぞれで、どこを共通コアにして、どこをアプリごとに書いたのか。言語とフレームワーク、UI と使い方の工夫、それから、元は many-ai-cli という自作ダッシュボードの音声入力だったものを外に出して共通コアにした経緯まで。

長いです。5 万字くらいあります。コーヒーを淹れてからどうぞ。

## vtype について

自作で [vtype](https://github.com/ishizakahiroshi/vtype) という音声入力のツールを作っています。Web でも、パソコンのどのアプリでも、声で書ける。回数制限も、アカウントも無い。

- 何ができるかの紹介ページ: https://ishizakahiroshi.com/work.html?id=vtype
- リポジトリ（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/vtype

同じ悩みを持っている方は、下記で入ります。

**Chrome 拡張**は、Chrome ウェブストアで「Chrome に追加」を押すだけです。Chrome 116 以降が必要です。続いて開くタブでマイクを 1 回だけ許可すれば、あとはどのサイトでも聞かれません。

https://chromewebstore.google.com/detail/vtype/nngfilimeplngdjdmgkddlhbdjpmikgn

**デスクトップ版**（Windows・macOS・Linux）は、下記のどれかで入ります。Google Chrome が入っている必要があります（理由は後半で書きます）。

```bash
# Node.js がある場合（Windows / macOS / Linux 共通）
npm i -g @ishizakahiroshi/vtype

# macOS で Homebrew を使う場合
brew install ishizakahiroshi/tap/vtype
```

Windows は GitHub Releases の zip でも入ります。Debian と Ubuntu は `.deb` もあります（`sudo apt install ./vtype_0.1.0_amd64.deb`）。

起動の仕方は、OS と入れ方で少し違います。

Windows の zip なら、展開して `vtype.exe` をダブルクリックします。npm で入れた場合は、ターミナルで `vtype` を実行します。

macOS と Linux は、ターミナルで `vtype` です。`.deb` なら次のサインインから自動で起動します。macOS は「システム設定 > プライバシーとセキュリティ > アクセシビリティ」で vtype を許可しないと、ほかのアプリへ文字を入れられません。Linux の Wayland では画面に浮かぶマイクが出ないので、トップバーのアイコンを使います。

最初にショートカットを押すと、小さな Chrome の窓が 1 回だけ開きます。そこで同意してマイクを許可したら、あとは Windows と Linux は Ctrl+Alt+Space、macOS は Control+Option+V を押して話すと、前面のアプリのカーソル位置へ文字が入ります。止めるときも同じキーです。

正直に書いておくと、macOS 版と Linux 版は CI でのビルドとテストまでで、実機ではまだ動かせていません。Microsoft Store 版は、この記事を書いている時点では審査の結果待ちです。

この話には前があります。前回は note で、vtype の紹介と、デスクトップ版を Microsoft Store に出すまでに躓いたところを書きました。

前回の記事: [回数制限のない音声入力「vtype」を作りました。デスクトップ版の Microsoft Store 申請は、2 回目なので 1 時間弱で出せた](https://note.com/ishizakahiroshi/n/ndd1204ce78a5)

今回は、その中身の話です。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/02_2026-09-24_vtype-architecture_infographic.png)

全部読むと長いので、目当てに合わせて飛ばしてください。

- なぜ無料で動くのかだけ知りたい: 「『無料』の正体は、Chrome に入っている Web Speech API」
- 共通コアの切り出し方: 「全体の地図」と「共通コア vtype-core」
- Chrome 拡張の実装: 「Chrome 拡張の分解」から 3 節
- デスクトップ版の実装: 「デスクトップ版の分解」から 3 節

記事の中のコードは、どれもリポジトリからの抜粋です。長いものは途中を省き、英語のコメントは記事のために日本語へ書き換えたものがあります。

## 元は、many-ai-cli の入力欄にあったマイクボタンだった

vtype の出どころから書きます。

[many-ai-cli](https://github.com/ishizakahiroshi/many-ai-cli) という、複数の AI コーディング CLI を並列で走らせて、承認をブラウザの 1 タブにまとめるダッシュボードを作っています。その入力欄の右端に、マイクのボタンがありました。

many-ai-cli の変更履歴を遡ると、5 月の v0.1.3 の時点で、すでに音声入力の修正が入っています。「波形を動かすためだけに、2 本目のマイクを開かない」。ブラウザの音声認識とマイクを取り合ってしまうから、という修正です。その後、スマホから話した音声を PC の Whisper へ中継する経路や、認識が止まったときの診断パネルが足されていきました。

毎日使っていました。並行して動かしている AI への指示は、今もかなりの部分を声で出しています。

困ったのは、そのマイクが many-ai-cli の入力欄にしか無いことでした。

ChatGPT の入力欄、Gmail の本文、GitHub の Issue、業務システムのフォーム。声で書きたい場所は、ブラウザのあちこちにあります。ブラウザの外、エディタやチャットのアプリにも。

音声入力の Chrome 拡張を探すと、多くに「1 日何分まで」の上限が付いていて、その先は有料版でした。いや、many-ai-cli のマイクは、Chrome に入っている音声認識をそのまま呼んでいるだけで、上限なんて持っていない。同じことを拡張でやればいいだけでは、と思いました。

それで、many-ai-cli の音声入力のうち「認識」の部分だけを切り出して、`vtype-core` という共通の部品にしました。そのうえで、使う側を 3 つにしています。

- Chrome 拡張（Web ページのテキスト欄へ）
- デスクトップ版（どのアプリへも）
- many-ai-cli 自身（切り出した部品を読み込み直す）

切り出したコミットのメッセージには、移植元のファイル名がそのまま残っています。

```text
feat(core): many-ai-cli の音声入力エンジンを vtype-core として切り出す

pnpm workspace を作り、packages/core（npm 名 vtype-core）へ認識エンジンを移す。
移植元は many-ai-cli の web/src/app/voice.ts と voice-whisper.ts。core は描画を
一切行わず、認識の制御と状態・結果の通知だけを持つ（tsconfig の lib から DOM を
外し、document を使うとコンパイルが通らない形にしている）。
```

many-ai-cli の側は、切り出した分だけ軽くなりました。コミットの記録では、`voice.ts` が 1021 行から 410 行に、`voice-whisper.ts` が 594 行から 310 行になっています。残ったのは、ボタン、音声バー、波形、診断の表示、トースト、ショートカットといった画面の側だけです。

見た目も持ってきました。拡張のパネルに並ぶ「× / 送信 / マイク」の 3 つのボタンは、many-ai-cli の入力バーと同じ並びです。配色のトークンも、波形の動かし方の数値も、many-ai-cli から移しています。同じ作者の MIT ライセンスのコードなので、遠慮なく持ってこられる。自分で書いたものを自分で再利用するのは、気が楽です。

進め方も書いておくと、コードの大半は Claude Code に書いてもらっています（一部の実験は別の AI）。自分は計画を切って、実機で触って、「これは使いづらい」と方針を変える側に回りました。この記事の途中に何度か出てくる「使ってみて変えた」は、だいたいそれです。

## 「無料」の正体は、Chrome に入っている Web Speech API

いちばん聞かれそうなところを、先に書きます。なぜ API キーもお金も要らないのか。

Chrome には、**Web Speech API** の音声認識（`SpeechRecognition`。Chrome では `webkitSpeechRecognition`）が入っています。Web ページの JavaScript から呼べる標準の API で、呼ぶ側は API キーを持ちません。

ただし、認識そのものは PC の中で完結していません。MDN にも、Chrome では音声がサーバー側の認識エンジンへ送られるので、オフラインでは動かない、と書いてあります。

https://developer.mozilla.org/en-US/docs/Web/API/SpeechRecognition

つまり Chrome は、話した声を Google の音声認識サービスへ送って、結果を受け取っています。vtype は、その流れに乗っているだけです。声が Google に届くことは、拡張ではマイクを許可するボタンの上に、デスクトップ版では最初の同意画面に書いています。

では、無料で上限なしなのか。ここは正確に書いておきます。

**上限を持たないのは、vtype の側の話です。** vtype のコードには、回数や時間を数えて止める仕組みがありません。一方で、Google が Chrome の音声認識に上限を設けているかどうかは、公式の文書には書かれていません。根拠として見つかったのは、Chromium の開発者の発言だけです。

- 2014 年と 2015 年の書き込み: 50 回/日の上限があるのは Chrome 開発用の音声 API のほうで、Chrome で動く Web サイトから使う Web Speech API は、Chrome の利用規約を守る限り無料で使える
  https://groups.google.com/a/chromium.org/g/chromium-dev/c/TJRsxtxkB_Y
- 2016 年の書き込み: 開発用の音声 API は Chrome の開発にしか使えないが、Web Speech API を使うのは別の話で、「there's no limit to that」
  https://groups.google.com/a/chromium.org/g/chromium-dev/c/5PrGai_wOZU

どちらもメーリングリストでの発言で、規約の条文ではありません。古い発言なので、今も同じだという保証まではない。そこは割り引いて読んでください。実績として言えるのは、毎日ほぼずっと使っていて、制限がかかったことはまだ一度も無い、というところまでです。

2016 年の発言は、裏返すと「やってはいけないこと」も教えてくれます。Chrome の中に入っている開発用の API キーを抜き出して、Google の音声サーバーを直接叩くこと。これは駄目です。vtype はそれをしません。Chrome を改変せず、Web ページから標準の API を呼ぶだけにしています。デスクトップ版で、わざわざ Chrome を裏で動かしているのも、この線を守るためです（後半で書きます）。

### Firefox で動かないのは、Firefox が認識を有効にしていないから

リポジトリの説明には「Chrome and Firefox」とありますが、今動くのは Chrome だけです。

Firefox は `SpeechRecognition` を既定で無効にしています。caniuse でも、Firefox は「Disabled by default」の扱いです。

https://caniuse.com/speech-recognition

加えて、拡張の要になっている offscreen document（後で書きます）に当たる API が、Firefox の WebExtensions にはありません。MDN の WebExtensions の API の一覧にも offscreen は載っていません。

https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API

Firefox 版を作るなら、認識エンジンを別に持ち込む必要があります。ここはまだ手を付けていません。

### 採らなかった案

Web Speech API に決める前に、ほかの道も並べました。

| 案 | 採らなかった理由 |
|---|---|
| クラウドの有料の音声認識 API | API キーと料金が要る。「アカウントも上限も無い」という vtype の存在理由に反する |
| Chrome の中の API キーで、音声サーバーを直接呼ぶ | 規約に反する。そのキーは Chrome の開発用 |
| OS 内蔵の音声認識 | OS ごとに精度も作りも違う。Linux には標準のものが無い |
| Whisper などのローカル認識 | 筋は良いが、大きなモデルのダウンロードと、PC の性能に左右される |
| headless の Chrome で認識させる | Web Speech API が動くという公式の根拠が見つからない |

Whisper は、many-ai-cli では実際に使っています（後で少し触れます）。vtype の既定にしなかったのは、入れた直後に大きなモデルを落としてくる道具にはしたくなかったからです。入れたらすぐ話せる、を優先しました。

### 最小のコード

Web Speech API の呼び方そのものは、拍子抜けするくらい短いです。vtype-core の中でも、認識の設定はこれだけです。

```ts:packages/core/src/recognition.ts
function configureRecognition(rec: SpeechRecognitionLike): void {
  rec.interimResults = true;   // 話している途中の結果も受け取る
  rec.continuous = false;      // 1 回話したら終わる（ここが後で効いてくる）
  rec.maxAlternatives = 1;
  rec.lang = getLang();        // ja-JP / en-US など
}
```

`new webkitSpeechRecognition()` を作って、これを設定して `start()` を呼べば、`onresult` に文字が届きます。

難しいのは、ここから先でした。途中の結果をどこに出すか。止まったまま戻ってこないときにどうするか。マイクを誰が持つか。文字をどこへどう入れるか。この記事の大半は、その先の話です。

## 全体の地図: 共通コア 1 つと、使う側が 3 つ

リポジトリは pnpm の workspace で、TypeScript のパッケージが 2 つと、pnpm の外に置いた Rust の crate が 1 つでできています。

```text
vtype/
├── packages/
│   ├── core/              vtype-core。認識エンジン。DOM を触らない（TypeScript）
│   │   └── src/
│   │       ├── recognition.ts   Web Speech API の制御と診断
│   │       ├── whisper.ts       録音して WAV で送る経路（many-ai-cli 用）
│   │       ├── input-modes.ts   入力モード（英語・カタカナ）と置き換え表
│   │       └── index.ts         createVoiceInput（2 つのエンジンの排他）
│   ├── extension/         Chrome 拡張（MV3・TypeScript）。dist/ が出荷物
│   │   ├── _locales/      利用者が読む文字列（en / ja）。Rust もここから読む
│   │   └── src/
│   │       ├── background/        ルーター（service worker）
│   │       ├── offscreen/         認識を回す唯一の場所と、カタカナの辞書
│   │       ├── content/           欄の検出、マイクの配置、文字の差し込み、送信
│   │       ├── ui/                パネル、ボタン、波形（shadow root の中）
│   │       ├── speech/            デスクトップ版が自分の Chrome で開く認識ページ
│   │       └── desktop-settings/  デスクトップ版の設定ページ
│   └── native/            デスクトップ版（Rust）。pnpm の外
│       ├── build.rs       認識ページと文言を実行ファイルに埋め込む
│       └── src/
│           ├── daemon.rs         常駐本体。判断は全部ここ
│           ├── speech_host.rs    127.0.0.1 の HTTP と WebSocket
│           ├── chrome_launch.rs  専用プロフィールで Chrome を起動
│           └── platform/         windows / macos / linux の差分
├── spike/                 使い捨ての実験（出荷しない）
└── scripts/               検査、ストア用のパッケージ作成、secrets-scan
```

デスクトップ版の認識ページと設定ページが、Rust の側ではなく拡張のパッケージの中にあるのに気づいたでしょうか。ここは意図的です。理由はデスクトップ版の節で書きます。

言語とフレームワークを並べると、こうなります。

| 部分 | 言語 | 主な道具 |
|---|---|---|
| vtype-core | TypeScript（ESM） | tsc、vitest。実行時の依存はゼロ |
| Chrome 拡張 | TypeScript | Manifest V3、esbuild（1 本ずつ IIFE に焼く）、vitest と happy-dom、kuromoji（カタカナ変換の辞書） |
| デスクトップ版 | Rust（1.98 に固定） | tray-icon、global-hotkey、tungstenite（WebSocket）、arboard（クリップボード）、tiny-skia（マイクの描画）、OS ごとに windows-rs / objc2 / gtk・x11rb・ashpd・atspi |
| many-ai-cli（使う側の 1 つ） | Go と TypeScript | vtype-core の写しを vendor に置いて読む |

規模の感覚も書いておきます。9 月 19 日から 24 日までのコミットが 70 本。ソースは空行を除いて、TypeScript（core と拡張）がおよそ 8,900 行、Rust がおよそ 17,100 行です。テストは、vitest が core で 31 件、拡張で 461 件。Rust は `#[test]` の関数が 238 個あります。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/03_2026-09-24_vtype-architecture_fig1.png)

図の上半分が共通の部分、下半分がアプリごとの部分です。共通コアは認識と入力モードの変換だけを持ち、画面には 1 ピクセルも描きません。描くのは全部、使う側です。

### 共通にしたのは 4 つ

1. **認識エンジン（vtype-core）**: Web Speech API の制御、止まったときの立て直し、診断、入力モードの変換。3 つの使う側が同じコードを読む
2. **認識のセッション管理（offscreen.ts）**: 1 回の録音をどう続けて、どう終わらせるか。拡張の offscreen document と、デスクトップ版が Chrome で開く認識ページが、同じ `createOffscreen` を使う
3. **利用者が読む文字列（_locales）**: 拡張の `messages.json` が唯一の置き場。デスクトップ版の Rust も、ビルドのときに `native_` で始まるキーを取り込む
4. **見た目と動きの考え方**: ボタンの並び、配色、それから「波形はマイクの音量ではなく、認識のイベントで動かす」という決まり。拡張は canvas で、デスクトップ版は Rust で、同じ考え方を別々に実装している

### アプリごとに書いたのは、画面と「出口」

拡張は、Web ページの欄を見つけてマイクを置き、そこへ文字を差し込みます。デスクトップ版は、Chrome を裏で起動し、前面のアプリへキー入力として打ち込みます。many-ai-cli は、自分の入力欄とボタンを持っています。

入口（声）と認識は同じで、出口（文字をどこへどう入れるか）だけが違う。vtype の構成を一言で言うと、そうなります。

## 共通コア vtype-core: DOM を一切触らない認識エンジン

### tsconfig から DOM を抜いて、描画をコンパイルエラーにした

vtype-core は、画面に何も描きません。要素を作らない、クラス名を使わない、メッセージも出さない。認識を回して、状態と結果とエラーを通知するだけです。

これを「気を付けて守る」のではなく、コンパイラに守らせています。

```jsonc:packages/core/tsconfig.json
{
  "compilerOptions": {
    // DOM を lib に入れない。core から document を触ると、コンパイルが通らない
    "lib": ["ES2022"],
    "types": [],
    "strict": true
  }
}
```

理由は 2 つあります。

1 つは、拡張の offscreen document の中でも動かす必要があることです。offscreen document には `window` はあっても、アプリの画面はありません。

もう 1 つは、使う側の画面を壊さないこと。many-ai-cli には既存のボタンもトーストもあるので、core が勝手に何か描くと二重になります。

`SpeechRecognition` や `navigator` のようなブラウザのグローバルは、型を付けた `globalThis` の見え方を 1 か所に作って、そこからだけ読みます。

```ts:packages/core/src/recognition.ts
interface HostGlobals {
  readonly SpeechRecognition?: SpeechRecognitionConstructor;
  readonly webkitSpeechRecognition?: SpeechRecognitionConstructor;
  readonly navigator?: NavigatorLike;
  setTimeout(handler: () => void, ms?: number): unknown;
  clearTimeout(handle: unknown): void;
}

function host(): HostGlobals {
  return globalThis as unknown as HostGlobals;
}
```

`SpeechRecognition` のコンストラクタ、`getUserMedia`、`AudioContext`、`fetch`、時計まで、全部を外から差し替えられるようにしてあります。そのおかげで、ブラウザの無い vitest で、認識の状態遷移をテストできます。イベントの通知も `EventTarget` を使わず、型付きの小さな emitter を自前で持っています。1 つのリスナーが例外を投げても、ほかのリスナーとエンジン自身の状態遷移は止まりません。

### 止めるたびに、認識のインスタンスを作り直す

many-ai-cli の頃にいちばん苦しめられたのが、Chrome の音声認識が「止まったまま戻ってこない」現象でした。

マイクは音を拾っている。なのに、結果が一向に届かない。ページを読み込み直すと直る。

原因を追うと、一度終わった（あるいは中断した）認識のインスタンスを使い回すと、Chrome の内部ではそれがまだ「開始済み」のまま残っていることがある、というところに行き着きました。もう一度 `start()` すると音は取り込むのに、結果は二度と届かない。

だから、止めるたびに作り直します。many-ai-cli の頃からの対策で、日本語のコメントもそのまま移植されています。

```ts:packages/core/src/recognition.ts
function stopVoice(): void {
  // ...
  if (!isRecording) return;
  const stoppedId = recognitionId;
  isRecording = false;
  setAudioActive(false);
  processing = false;
  emitter.emit('stop', { recognitionId: stoppedId });
  // 次回のためにインスタンス作り直し（Chrome stuck 対策）
  // Chrome can keep a finished/aborted instance internally "started"; starting it again
  // then captures audio but never delivers a result. Always start from a fresh instance.
  if (!disposed) createInstance();
  emitState();
}
```

インスタンスには連番の `recognitionId` を振ります。作り直すと番号が 1 つ進む。これが後で効いてきます。

Chrome は、止めた後の古いインスタンスから、最後の確定結果を遅れて届けてくることがあります。結果には `isCurrent`（今のインスタンスからの結果か）を付けて通知し、捨てるかどうかは使う側に決めさせています。many-ai-cli は遅れて届いた確定結果も入力欄に入れていたので、その振る舞いを変えないためです。この手の「core はこう約束する」は、README に契約の注意として 8 項目並べてあります。

「音は取れたのに結果が来ない」を見つける仕組みもあります。`audioend`（マイクの取り込みが終わった）から 20 秒たっても、結果も終了もエラーも来なければ、診断の状態を「プロフィールか音声認識が詰まっている疑い」にします。直近 80 件のイベント（時刻、どのインスタンスか、結果の文字数）を記録していて、診断の報告として JSON でコピーできます。記録するのは文字数だけで、話した言葉そのものは入れません。

### 2 つのエンジンに、マイクを同時に持たせない

vtype-core には、認識のエンジンが 2 つあります。ブラウザの Web Speech API と、Whisper に送るための録音です。

Whisper の経路は、many-ai-cli のためのものです。スマホから話す場合のために、many-ai-cli は `getUserMedia` で録音して 16 kHz の WAV にし、Hub を経由して PC の Whisper サーバーへ送る経路も持っています。`MediaRecorder` を使わないのは、iOS では AAC で出てきて、サーバー側に変換の手間が要るからです。vtype 自身（拡張とデスクトップ版）は、この経路を使っていません。

問題は、この 2 つを同時に動かしたときです。Chrome はマイクを片方にしか渡さず、もう片方は黙って何も受け取りません。エラーにもならない。これは many-ai-cli で実際に踏んでいて、5 月の修正（波形のために 2 本目のマイクを開かない）も、同じ種類の話でした。

なので、2 つのエンジンは `createVoiceInput()` という入口からしか作らせず、そこで排他をかけています。

```ts:packages/core/src/index.ts
const whisperActive = () => !!whisper && whisper.isActive();
const speechActive = () => !!rawRecognizer?.isActive() || !!hotword?.isActive();

whisper = createWhisperRecorder({
  ...options.whisper,
  // ブラウザの認識が動いている間は、Whisper の録音を開かせない
  canStart: () => (speechActive() ? ENGINE_BUSY : null),
});

recognizer = {
  ...raw,
  start: (): StartOutcome => {
    // 逆向きも同じ
    if (whisperActive()) return { started: false, error: ENGINE_BUSY };
    return raw.start();
  },
};
```

断られた側には `engine_busy` が返ります。テストは両方向で 8 件。「たぶん同時には押さないだろう」ではなく、コードで止めています。

`isActive()` が「録音中」だけでなく「start() を呼んだが、まだ開始の通知が来ていない間」も含むのが、地味に大事なところです。開始を待っている間にもう片方がマイクを開くと、同じ取り合いが起きるからです。

### 入力モードとカタカナ変換も、core に置いた

vtype には入力モードが 3 つあります。通常、英語、カタカナ。デスクトップ版の 0.1.0 から入っていて、拡張には次の版で入れる予定です（ストアに出ている拡張の 0.1.0 には、まだ入っていません）。

- **通常**: ブラウザの言語で認識する
- **英語**: 認識の言語を `en-US` にする
- **カタカナ**: 日本語で認識して、結果を全角カタカナにする

変換は純粋な関数だけで書いて、core に置きました。拡張の offscreen と、デスクトップ版の認識ページが、同じ実装を使います。順番に意味があります。

```ts:packages/core/src/input-modes.ts
// 置き換え表 → （カタカナなら）読み → toKatakana の順。
// 置き換えで入れた語（固有名詞など）は、後の段が触らない
export async function transformTranscript(text: string, options: AsyncTransformOptions): Promise<string> {
  const { mode, rules, reading } = options;
  if (mode !== 'kana' || !reading) return transformTranscriptSync(text, { mode, rules });
  const segments = replaceToSegments(text, rules);
  const parts = await Promise.all(
    segments.map(async (s) => (s.replaced || s.text === '' ? s.text : toKatakana(await reading(s.text)))),
  );
  return parts.join('');
}
```

置き換え表（「A と聞こえたら B と書く」）を先に当てるのは、登録した固有名詞をカタカナに崩させないためです。たとえば「vtype」を置き換え表に登録しておけば、カタカナモードでも「vtype」のまま残ります。置き換えは長い `from` から順に、左から重ならないように当てます。英字は大文字と小文字を区別しません。最大 200 件です。

漢字をカタカナにするには、読みが要ります。そこは kuromoji と IPADIC の辞書を同梱しました。辞書の読み込みには少し時間がかかるので、カタカナモードで最初に確定結果が出たときに、初めて読み込みます。ほとんどの録音はカタカナモードを使わないからです。読み込みに失敗したら、ひらがなだけをカタカナにする変換に落とします。何も出ないよりはましなので。

ひらがなからカタカナへは、文字コードを `0x60` ずらすだけです。半角カタカナの濁点の合成（ｶﾞ → ガ）も、同じ関数でやっています。

```ts:packages/core/src/input-modes.ts
if (code >= 0x3041 && code <= 0x3096) {
  out += String.fromCharCode(code + 0x60);  // ぁ..ゖ → ァ..ヶ
}
```

話している途中の結果は同期の変換（置き換えとひらがなのカタカナ化だけ）で出し、確定した結果だけを辞書に通します。途中の結果は次々に置き換わるので、辞書を待つ意味が無いからです。

### many-ai-cli には、npm ではなく「写し」で戻した

切り出した core を many-ai-cli に戻すところで、1 つ判断が要りました。npm の依存として入れるか、ファイルを写すか。

写すほうにしました。理由は many-ai-cli の側の事情です。

- many-ai-cli の Web 画面は `bundle: false` でビルドしていて、`import 'vtype-core'` のような裸の指定子が、ブラウザで解決できない
- Hub の CSP が `script-src 'self'` なので、インラインの import map も使えない
- `package.json` に書くと、CI の `bun install --frozen-lockfile` が vtype の場所を解決できずに、全配布経路のリリースが止まる

そこで、xterm などと同じく `src/vendor/vtype-core/` にビルド済みの JS と型定義を置いてコミットし、相対パスで import しています。写すのはスクリプト 1 本で、`--check` を付けると、写しが古いときに失敗します。

```bash
node scripts/sync-vtype-core.mjs          # 隣に置いた vtype から写す
node scripts/sync-vtype-core.mjs --check  # 写しが古ければ exit 1
```

正本は vtype の側の `packages/core` です。vendor の写しを手で直すことはしません。直すなら vtype で直して、ビルドして、写す。

many-ai-cli の側は、こう呼ぶだけになりました。

```ts:many-ai-cli/web/src/app/voice-engine.ts
export const voiceInput = createVoiceInput({
  engine: () => getVoiceEngine(),   // 'browser' | 'whisper' | 'off'
  recognition: {
    lang: () => appLangToRecognitionLang(localStorage.getItem(STORAGE_LANG_KEY)),
  },
  whisper: {
    endpoint: '/api/voice/transcribe',
    token,
    recorderWorklet: { url: RECORDER_WORKLET_URL, processorName: 'many-ai-cli-whisper-recorder' },
  },
});
```

送り先の URL も token も、core は持っていません。使う側が渡します。core の中に URL が 1 つも無いのは、拡張としてストアに出すときにも気が楽でした。

## Chrome 拡張の分解: 認識は offscreen document の中だけで回す

ここから Chrome 拡張です。構成は Manifest V3 の素直な形で、要求する権限は 2 つだけです。

```json:packages/extension/manifest.json
{
  "manifest_version": 3,
  "minimum_chrome_version": "116",
  "permissions": ["offscreen", "storage"],
  "background": { "service_worker": "background.js" },
  "content_scripts": [
    {
      "matches": ["http://*/*", "https://*/*"],
      "js": ["content.js"],
      "run_at": "document_idle",
      "all_frames": true
    }
  ]
}
```

`tabs` も、ホスト権限も要求していません。後で書く「サイトごとにオフ」の機能も、background がタブのサイトを知らないまま実現しています。

### content script で認識すると、サイトごとにマイクの許可を聞かれる

いちばん最初に決めたのは、「認識をどこで動かすか」でした。

素直に作るなら、ページに差し込む content script の中で `webkitSpeechRecognition` を呼びます。でもこれをやると、マイクの許可はそのページのオリジンに対して求められます。Gmail で 1 回、ChatGPT で 1 回、業務システムで 1 回。訪れたサイトの数だけ、許可のダイアログが出る。それは使えない、と思いました。

そこで、認識は拡張自身のページで回すことにしました。Manifest V3 では、画面に出ない拡張のページとして **offscreen document** が使えます。`reasons` に `USER_MEDIA` を指定すれば、マイクなどのメディアを扱えます。

https://developer.chrome.com/docs/extensions/reference/api/offscreen

offscreen document のオリジンは `chrome-extension://<拡張の ID>` で、どのサイトを見ていても同じです。ここで一度許可を取れば、以後どのサイトでも聞かれないはず。

「はず」を確かめるために、本番のコードを書く前に、使い捨ての拡張を作りました（`spike/mic-permission`）。結果はこうです。

- 事前に許可を取らないと、offscreen の認識は `not-allowed` で即座に終わる。offscreen document は、許可のダイアログを出せない
- 拡張のオリジンのページで 1 回 `getUserMedia` を許可した後は、オリジンの違う 3 つのサイトで、許可のダイアログ 0 回のまま日本語を認識できた

それで、インストールした直後に拡張のページ（`permission.html`）を 1 回開いて、そこでマイクを許可してもらう形にしました。許可が取れたら、ストリームはすぐに止めます。欲しいのは許可だけで、録音は Web Speech API が自分でやるからです。

```ts:packages/extension/src/permission/permission.ts
const stream = await getUserMedia({ audio: true });
for (const track of stream.getTracks()) track.stop();  // 許可が取れたら、すぐ手放す
```

このページには、許可するボタンの**上**に「声は Google の音声認識へ送られる」と書いています。ボタンを押す前に、声の行き先が目に入る順番にしたかったからです。コードのコメントにも、わざとボタンの前に置いている、と残してあります。

### background は「ルーター」で、状態を持たない

3 つの登場人物の役割は、こう分けています。

- **content script**（各ページの各フレーム）: 欄を見つける、マイクを置く、文字を差し込む。音声や認識の API には触らない
- **background**（service worker）: 中継役。offscreen document を必要なときに作り、メッセージを運ぶ
- **offscreen document**: 認識を回す唯一の場所。セッションの状態も、ここが持つ

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/04_2026-09-24_vtype-architecture_fig2.png)

図のとおり、声はいつも offscreen document で文字になり、background を経由して、録音を始めた欄のあるタブの、そのフレームへ返ります。content script は、音声の API に一度も触りません。

background に状態を持たせなかったのは、Manifest V3 の service worker が、Chrome の都合で止められたり起こされたりするからです。今どのセッションが動いているか、誰が持ち主かは offscreen document が持っていて、そちらは動き続けます。background は、止まっても困らない作りにしました。

offscreen document の作り方には、細かい注意がいくつかあります。

```ts:packages/extension/src/background/index.ts
async function ensureOffscreen(): Promise<void> {
  if (creating !== null) { await creating; return; }  // 同時に 2 回作らない
  if (await offscreenExists()) return;                  // 起こし直された worker が、既存の document を見つける
  creating = chrome.offscreen
    .createDocument({
      url: offscreenUrl,
      reasons: ["USER_MEDIA"],
      justification: "Run the browser's speech recognition for voice input into web text fields.",
    })
    .catch(async (err: unknown) => {
      if (!(await offscreenExists())) throw err;  // すでにあったのなら問題ない
    });
  try { await creating; } finally { creating = null; }
}
```

offscreen document は、1 つの拡張につき同時に 1 枚しか開けません。Chrome の文書にもそう書いてあります。作っている最中にもう 1 回呼ばれる、起こし直された worker が既にある document を知らない、の 2 つを、両方潰しています。存在の確認は Chrome 116 以降の `runtime.getContexts` を使い、無ければ古い `offscreen.hasDocument` に落とします。最低バージョンを 116 にしているのは、このためです。

録音が終わって 30 秒たつと、offscreen document は閉じます。

offscreen document から使える拡張の API は `chrome.runtime` だけです（これも Chrome の文書にあります）。なので、入力モードや置き換え表は、`chrome.storage` を読める background が読んで、録音を始めるメッセージに毎回載せて渡しています。Chrome が worker を起こし直した直後に、間違ったモードで始めないよう、最初の読み込みが終わるまで開始を待たせています。

結果を返すときは、`tabs.sendMessage` に `frameId` を付けて、録音を始めたフレームだけに届けます。content script は `all_frames: true` で iframe の中にも入っているので、タブ宛てに投げるだけだと、関係ないフレームにまで届いてしまうからです。タブが閉じられたり、フレームが消えたりしたら、offscreen document に中断を伝えます。持ち主のいない録音を残さないためです。

録音は同時に 1 本だけで、別のタブで録音を始めると、前のタブの録音は「ほかに取られた（superseded）」として終わります。

### Chrome は 1 回話すと認識を終える。だから、つなぎ続ける

offscreen document の中身で、いちばん頭を使ったのがここです。

vtype-core は、認識を `continuous = false` で動かします。many-ai-cli の頃からそうで、ここは変えられないようにしてあります。このモードの Chrome は、ひとまとまり話し終えると（あるいは、しばらく黙ると）認識を終えてしまいます。

でも、使う人から見れば、マイクを押してからもう一度押すまでが 1 回の録音です。途中で黙って考えても、終わってほしくない。

そこで offscreen document は、「セッション」と「サイクル」を分けています。

- **セッション**: マイクを押してから止めるまで。同時に 1 つだけ
- **サイクル**: Chrome の認識 1 回ぶん。セッションが開いている間は、終わるたびに次を始める

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/05_2026-09-24_vtype-architecture_fig3.png)

図にすると、セッションの中でサイクルが回り続けて、3 つの条件のどれかでセッションが終わる形です。サイクルが替わるたびに、vtype-core がインスタンスを作り直しているので、止まったまま戻ってこない、に引きずられにくくなっています。

終わらせる条件は、この 3 つです。

- 利用者が止めた
- 致命的なエラー（`not-allowed`、`network`、`audio-capture` など）。ただし `no-speech`（何も聞こえなかった）はエラーではなく、無音のサイクルとして数える
- **文字が 1 つも出ないサイクルが 3 回続いた**。押したまま忘れた録音が、いつまでもマイクを握らないように

```ts:packages/extension/src/offscreen/offscreen.ts
export const SILENT_CYCLE_LIMIT = 3;
export const STOP_GRACE_MS = 1500;

function cycleEnded(s: Session, endedId: number): void {
  s.cycleActive = false;
  const err = s.lastError;
  s.lastError = null;
  // 今終わったインスタンスのエラーだけを数える。古いインスタンスの遅れたエラーは、普通の終わりとして扱う
  if (err !== null && err.recognitionId === endedId && !SILENT_ERRORS.has(err.code)) {
    end(s, "error", err.code);
    return;
  }
  if (!s.heardTextThisCycle) s.silentCycles += 1;
  if (s.silentCycles >= SILENT_CYCLE_LIMIT) {
    end(s, "silence");
    return;
  }
  // vtype-core は stop を出した直後にインスタンスを作り直しているので、次のサイクルはそちらで始める
  setTimeout(() => startCycle(s), 0);
}
```

ここでも `recognitionId` が効いています。セッションが始まる前のインスタンスから来た結果は、捨てる。古いインスタンスが遅れて出したエラーで今の録音が終わっても、それはエラーではなく普通の終わりとして扱い、次のサイクルを始める。番号が無いと、この区別ができません。

止めるときにも、小さな工夫があります。話している途中でマイクを押すと、まだ確定していない「途中の結果」が残っています。Chrome は `stop()` の後で、それを古いインスタンスから確定として届けてくることがある。だから、止めた直後に終わりにせず、途中の結果が出ていたときだけ、最大 1.5 秒その確定を待ちます。最後の一言が消えるのを防ぐためです。

もう 1 つ。カタカナモードの確定結果は、辞書を引くので非同期になります。その間に「セッション終了」のメッセージが追い越すと、最後の文字が落ちます。そこで、セッションのメッセージは全部、1 本の Promise の鎖に並べて、順番どおりに送っています。

```ts:packages/extension/src/offscreen/offscreen.ts
function post(s: Session, event: SessionEvent): void {
  s.outbox = s.outbox.then(() => send(s, event));
}
```

3 行です。でも、これが無いと、カタカナモードで最後の一言だけが消える、というとても腹の立つ不具合になります。こういう「最後の一言が消える」系は、使っている側からは一番いらいらするので、先に潰しておきました。

### ビルド: esbuild で、1 本ずつ IIFE に焼く

ビルドは esbuild で、スクリプトごとに依存を全部入れた 1 本の IIFE にしています。MV3 の content script はクラシックスクリプトで、`import` が使えないからです。

`build.mjs` は、検査も兼ねています。

- 出力に `import` / `export` が残っていたら、失敗する
- manifest や HTML が指しているファイルが無ければ、失敗する
- `package.json` と `manifest.json` の版がずれていたら、失敗する（版の正本は manifest）
- `__MSG_*` が指す文言が無ければ、失敗する

minify はしていません。ストアの審査の人が、出荷したコードをそのまま読めるようにするためです。

一度だけ、ここで冷や汗をかきました。CSS をテキストとして読み込む自作の esbuild プラグインで、解決したパスを絶対パスで返していたら、esbuild が出力の中に `// raw:<パス>` というコメントを残していて、**作者のフォルダ構成が、ストアに出す bundle に入っていました**。minify しないと、こういうものまで出てしまいます。パスを相対にして直しています（コミット `6509f9c`）。

```js:packages/extension/build.mjs
// esbuild は minify しない bundle で、モジュールの上に `// raw:<path>` を出す。
// だから渡すパスは相対にする。絶対パスだと、ビルドした人のディレクトリがストアへ出てしまう
b.onResolve({ filter: /\?raw$/ }, (args) => {
  const file = join(args.resolveDir, args.path.slice(0, -"?raw".length));
  return { path: relative(here, file).split(sep).join("/"), namespace: "raw", pluginData: { file } };
});
```

kuromoji にも手を入れています。kuromoji のブラウザ向けの辞書ローダーは、任意の URL を XHR で読める作りです。ストア用の zip を作る前の検査スクリプトは、外への通信を含むものを出荷させません。そこで、拡張の中のファイルしか読めない自前のローダーに差し替えました。

## 入力欄を見つけて、文字を入れる

content script の仕事は 3 つです。欄を見つける、マイクを置く、文字を入れる。置く話は次の節に回して、ここでは見つけ方と入れ方を書きます。

### 対象は「許可リスト」で決める。password は除外するのではなく、入れない

どの欄を対象にするかは、`<input>` の `type` の**許可リスト**で決めています。

```ts:packages/extension/src/content/detect.ts
export const TARGET_INPUT_TYPES: ReadonlySet<string> =
  new Set(["text", "search", "email", "url", "tel"]);
```

`password` を「除外する」のではなく、そもそも許可リストに入れていません。この違いは小さく見えて、大事です。除外リストだと、将来 HTML に新しい入力の種類が増えたとき、それは黙って対象になります。許可リストなら、新しい種類は何もしなくても対象外のままです。

それでもパスワードをすり抜けさせないために、あと 3 つ見ています。

- `autocomplete` が `current-password` / `new-password` の欄は除く。「パスワードを表示」のボタンで `type="text"` に切り替わっても、`autocomplete` は残るから
- `-webkit-text-security: disc` などで文字を伏せている欄は除く。`type="text"` に CSS で伏せ字をかけるサイトがあるから
- `readonly` と `disabled` の欄は除く

`contenteditable` は、HTML の継承のルールを、属性から自前で辿っています。一番近い祖先の値で決まり、`false` で止まる。shadow root は越えない。`isContentEditable` を使わないのは、テストで使う DOM の実装（happy-dom）の出来に、結果を左右されたくなかったからです。`contenteditable` の領域の中に `<input type="password">` があっても、`input` はそれ自身の `type` で判定するので、除外されたままになります。

小さすぎる欄（幅 60px 未満か、高さ 18px 未満）には、マイクを出しません。ツールバーの絞り込み欄や、表の 1 文字のセルや、隠れた 1×1 の input に、いちいちマイクが付くとうるさいからです。画面に同時に出すマイクは、最大 12 個にしました。

入れる直前にも、もう一度この判定を通します。変換を待っている間に、ページが欄をパスワード欄に変えることもありうるからです。対象の判定に関わる属性（`type`、`readonly`、`disabled`、`contenteditable`、`autocomplete`）は `MutationObserver` で見張っていて、変わったらマイクを外します。欄そのものは読むだけで、書き換えません。

### input と textarea には「ネイティブの setter」で入れる

文字の入れ方は、欄の種類で 2 通りあります。

`<input>` と `<textarea>` では、`field.value = ...` と代入してはいけません。React の制御コンポーネントは、インスタンスの `value` に自分の仕掛けを被せていて、代入しても state が更新されません。見た目には入ったのに、送信すると空。そういうことが起きます。

そこで、プロトタイプにある**本来の setter** を呼んでから、`input` イベントを投げます。

```ts:packages/extension/src/content/insert.ts
function nativeValueDescriptor(el: TextControl): PropertyDescriptor {
  // インスタンスではなく、その要素の realm のプロトタイプから取る。
  // React の value tracker はインスタンスに value を被せているので、それを迂回する
  const view = (el.ownerDocument.defaultView ?? globalThis) as typeof globalThis;
  const proto = el.localName === "input" ? view.HTMLInputElement.prototype : view.HTMLTextAreaElement.prototype;
  const descriptor = Object.getOwnPropertyDescriptor(proto, "value");
  if (descriptor?.get === undefined || descriptor.set === undefined) {
    throw new Error(`no native value accessor on ${el.localName}`);
  }
  return descriptor;
}

export function writeNativeValue(el: TextControl, next: string, inputType: string, data: string | null): void {
  nativeValueDescriptor(el).set!.call(el, next);
  dispatchInput(el, inputType, data);  // bubbles: true, composed: true の InputEvent
}
```

「その要素の realm の」がミソです。iframe の中の欄は、親のページとは別の `HTMLInputElement` を持っています。親の側のプロトタイプから取った setter では、うまく動きません。

`type="email"` の input には選択範囲の API がありません（`selectionStart` が `null`）。そのときは末尾に足します。

### contenteditable には execCommand("insertText")

Gmail の本文や、ProseMirror・Lexical・Draft.js のようなリッチエディタは、`contenteditable` の上に自分の文書モデルを持っています。DOM を直接いじると、元に戻されたり、無視されたりします。

ここは `document.execCommand("insertText")` を先に試します。非推奨の API ですが、ブラウザの編集の流れを通るので、エディタが聞いている `beforeinput` / `input` がちゃんと飛びます。`execCommand` が無いか、`false` を返したときだけ、`Selection` と `Range` で入れて、`input` イベントを投げます。

`execCommand` は、フォーカスのある編集領域の、今の選択範囲にしか効きません。パネルのボタンを押した後はフォーカスがそちらにあることもあるので、必要なら編集領域にフォーカスを戻し、選択範囲が領域の外にあれば、カーソルを中身の末尾に置いてから入れます。フォーカスを戻すときは `preventScroll: true` で、ページが勝手にスクロールしないようにしています。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/06_2026-09-24_vtype-architecture_fig4.png)

図のとおり、どちらの経路でも、入れる前に必ず「IME で変換している最中か」を確かめます。

### 日本語の変換中には、書き込まない

日本語の IME で変換している最中に、横から文字を入れると、変換中の文字列と混ざって壊れます。

`compositionstart` から `compositionend` までの間は書き込まず、待ちます。document のキャプチャ段階で聞いているので、open な shadow root の中の欄も拾えます（composition のイベントは composed だからです）。待つのは最大 10 秒で、それを過ぎたら入れずに諦めます。その場合も文字は捨てずに、パネルに残します。

まとめて 1 回入れる経路では、1 つの欄への書き込みを欄ごとの Promise の列に並べて、呼ばれた順に 1 回ずつ実行します。話している途中の書き換え（次の節）は、待っている間に来た更新を最後の 1 回にまとめます。書くのはいつも最新の状態なので、途中を全部書く必要がないからです。

変換が終わった後、すぐには書きません。`setTimeout(..., 0)` で 1 回だけ待って、ブラウザが変換した文字を確定させ終わってから入れます。これを抜くと、変換の確定と vtype の書き込みがぶつかることがありました。

### 話しているそばから、欄の中で書き換える

最初は、途中の結果をパネルに出して、止めたときにまとめて欄へ入れる作りでした。9 月 19 日に置き場所を検討したときは、そちらを選んでいます。途中経過を欄に流すと、サイトの検索候補や自動保存が何度も反応して暴れそうだったからです。

翌日、実機で使ってみて、やめました。

拡張のパネルの中に文字が展開されるのは、使いづらい。マイクを押した瞬間から受け付けて、聞き取れた文字をその場で欄に入れてほしい。入った結果を、そのまま送信できるほうがいい。リッチエディタだけは確定ごとに足す案もありましたが、話しているのが出てくればいい、全部の欄で途中から出す、と決めました。

それで今は、途中の結果を欄の中で直接書き換えています。`input` / `textarea` では、録音を始めた時点のカーソルの前後（`prefix` と `suffix`）を覚えておき、結果が来るたびに「前 + 確定した分 + 途中 + 後ろ」を組み立て直して、ネイティブの setter で書きます。

```ts:packages/extension/src/content/insert.ts
function writeTextControl(): void {
  let a = anchor as TextControlAnchor;
  // 前回書いた後に、誰か（利用者の手入力、ページのリセット、× ボタン）が欄を変えていたら、
  // 今の欄を新しい起点にする
  if (lastWritten !== null && readNativeValue(field) !== lastWritten) {
    confirmed = "";
    anchorNow();
    a = anchor as TextControlAnchor;
  }
  const body = confirmed + interim;
  const next = a.prefix + body + a.suffix;
  writeNativeValue(field, next, "insertText", interim === "" ? confirmed : interim);
  lastWritten = next;
  const caret = (a.prefix + body).length;
  field.setSelectionRange(caret, caret);
}
```

`lastWritten` との比較が、地味に効いています。録音中に利用者がキーボードで何か打ったら、vtype はそれを消さずに、今の欄を新しい起点として書き続けます。人が書いた文字を、機械が上書きしない。これは譲れないところでした。

`contenteditable` では、前回の途中経過の文字数ぶんをカーソルの前から選択して、`execCommand("insertText")` で置き換えます。ノードを組み立て直すことはしません。リッチエディタと喧嘩になるからです。カーソルが前回置いた場所から動いていたら（利用者が別の場所をクリックした）、置き換えずに、そこから新しく書き始めます。ここも、人が書いた文字を食べないためです。

リスクは承知のうえでの判断でした。自前の文書モデルを持つエディタでは、途中経過の書き換えで文字が戻ったり二重になったりする可能性がある。崩れるサイトが出てきたら、そのサイトだけ確定ごとに足す、を考える。今はまだ作っていません。

認識の区切りをつなぐときは、前が英数字（か英文の句読点）で終わり、次が英数字で始まるときだけ、半角スペースを入れます。Chrome の結果には区切りが付いてこないので、放っておくと「hello」と「world」がくっつきます。日本語にはスペースを入れません。

```ts:packages/extension/src/content/controller.ts
export function separatorBefore(previous: string, next: string): string {
  const trimmed = next.trim();
  if (previous === "" || trimmed === "") return "";
  return /[A-Za-z0-9.,!?;:)'"]$/.test(previous) && /^[A-Za-z0-9('"]/.test(trimmed) ? " " : "";
}
```

録音は、始めた欄が持ちます。録音中に別の欄をクリックしても、別のタブに移っても、文字の行き先は変わりません。止めるまでは、最初の欄に入り続けます。

### 送信ボタンは「送れた証拠」が無ければ送らない

パネルの送信ボタンは、文字を入れるだけでなく、ページの送信までやります。検索なら検索が走り、投稿フォームなら投稿されます。

そのぶん、押し間違えたときの被害が一番大きいボタンです。投稿系のサイトで意図しない送信が起きたら、取り返しがつきません。なので、**送信先が確実に分からないときは送らない**を既定にしました。「たぶんこれだろう」で送らない。

```ts:packages/extension/src/content/submit.ts
// 1. 欄が <form> の中なら form.requestSubmit()。form.submit() は使わない
//    （submit イベントが飛ばず、サイトの検証や SPA のハンドラを飛ばしてしまう）
//    送れた証拠: submit イベントが実際に飛んだこと
// 2. form が無ければ、Enter キーの keydown / keypress / keyup を欄に送る
//    送れた証拠: ページのハンドラが keydown を preventDefault() したこと
// どちらの証拠も無ければ、送信しない
```

合成したキーイベントは、ブラウザの既定の動作を起こしません。誰も聞いていなければ、何も起きない。だから「ページが受け取った」ことは、ハンドラが `preventDefault()` したかで判定しています。古いハンドラのために `keyCode` と `which` も 13 に見えるようにしてあります。`KeyboardEventInit` では設定できないので、`Object.defineProperty` で被せています。

`requestSubmit()` が HTML のバリデーションで止まったときは、Enter には落としません。同じフォームを、検証の裏から送ろうとすることになるからです。

テストで、実在のサイトに実際に送信して確かめることは、していません。合成した DOM の単体テストだけです。実サイトでの確認は、自分の手でやりました。

## UI の工夫: そこにいるけど、邪魔をしないマイク

拡張の見た目で一番気を遣ったのは、「ページを壊さない」と「目立ちすぎない」の両立です。

### closed な shadow root の中に、全部しまう

vtype は、ページの欄を 1 つも書き換えません。欄を包まない、属性を足さない、スタイルに触らない。欄に対してやるのは、位置を読むことと、属性の変化を見張ることだけです。

マイクとパネルは、`<html>` の直下に置いた自前の要素 `<vtype-root>` の、**closed な shadow root** の中にあります。

```ts:packages/extension/src/content/anchor.ts
host = doc.createElement("vtype-root");
// ページのスタイルシートに、大きさを変えられたり、隠されたり、ずらされたりしないよう、インラインの !important で固める
for (const [prop, value] of [
  ["all", "initial"], ["position", "fixed"], ["top", "0"], ["left", "0"],
  ["width", "0"], ["height", "0"], ["overflow", "visible"],
  ["z-index", "2147483647"], ["display", "block"],
] as const) {
  host.style.setProperty(prop, value, "important");
}
root = host.attachShadow({ mode: "closed" });
```

shadow root の中には、ページのセレクタが届きません。それでも境界を越えてくるものが 2 つあります。ホスト要素そのものへのルールと、継承されるプロパティです。ホストは `all: initial !important` で初期化し（shadow の中からの `!important` は、ページ側の `!important` に勝ちます）、中のサイズは全部 px で書いて、`box-sizing` も明示しました。

アイコンは、SVG を `createElementNS` で組み立てています。`innerHTML` は使いません。Trusted Types を強制しているページでは弾かれるし、絵文字や記号のアイコンは、ページのフォントに左右されるからです。

テスト用に、わざと意地悪な CSS を当てたページ（`testbed/hostile.html`）も用意して、崩れないことを確かめています。

### 薄いマイクを、欄の右の外側に

マイクは 24px の角丸の四角で、ふだんは不透明度 0.55 の薄い灰色です。ポインタが乗ると不透明になって、アクセントの色（インディゴ）になります。

最初の設計では、欄をクリックしたときだけ、マイクを薄く出すつもりでした。これも使ってみて変えました。欄をクリックしてからでないとマイクが出ないのは、使いづらい。今は既定で、画面に見えている欄すべてに薄く出しています。設定で「ポインタを乗せた欄とカーソルのある欄だけ」にも変えられます。

欄の中ではなく、右端の**外側**に置いたのは、欄の中にはサイト自身のボタンが置かれていることが多いからです。検索欄の × や、Google のマイクのボタン。狭い欄には、そもそも入りません。最初に置き場所を検討したとき、欄の中に 3 つとも重ねる案、欄の下にバーを出す案、画面の隅に置く案と並べて、「欄の横にマイク 1 個だけ、押すとパネルが開く」を選びました。細い検索欄でも成り立つのは、これだけだったからです。

### サイトのボタンと重なったら、自分でよける

それでも、外側に置いたマイクが、サイトのボタンと重なる例が出ました。欄の中の ×、それから、欄のすぐ右隣にある検索ボタンです。

どこに置いても当たるサイトはある。こればかりは何とも言えない。それで、置く前に「そこに何があるか」を調べることにしました。候補の場所を 4 つ、優先順に持っています。

```ts:packages/extension/src/content/anchor.ts
export const MIC_SPOTS = ["outside-right", "inside-right", "above-right", "outside-left"] as const;

const PRESSABLE_SELECTOR =
  'button, a[href], input, select, textarea, summary, [role="button"], [role="link"], ' +
  '[role="menuitem"], [role="tab"], [role="checkbox"], [role="radio"], [role="switch"]';
```

各候補の中心で `document.elementsFromPoint()` を呼び、一番上にあるものが「押せるもの」なら、塞がっていると見なして次の候補へ移ります。文字や背景や飾りの箱の上なら、そのまま置いてかまいません。欄そのもの（と、欄を包んでいる箱）は数えません。マイクはその欄のためのものだからです。

どの候補も塞がっていたら、最初の候補に戻します。重なっていても、ドラッグでずらせる。でも、そこに無いマイクは、ずらしようがないからです。

この判定は、毎フレームはやりません。マイクが新しく出たときと、欄の大きさが変わったときだけです。スクロールでは、欄とその周りのボタンは一緒に動くので、判定し直す必要がありません。一度決めた場所は、次に判定を頼まれるまで変えない。フレームごとに場所がぱたぱた入れ替わるのを防ぐためです。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/07_2026-09-24_vtype-architecture_illustration-polite-mic.png)

先に座っていたサイトのボタンには、マイクのほうが席を譲る。vtype のマイクは、そういう控えめな置かれ方をしています。

自動でよけきれない場所のために、マイクはドラッグでずらせます。ポインタを 4px 以上動かしたらドラッグ、それ未満なら録音のボタンとして扱います。スマホやタブレットでは、ページがスクロールしてしまわないよう、マイクに `touch-action: none` を付けています。

ずらした量は「画面の座標」ではなく「欄からの距離」として、**サイトごと**に覚えます（最大 50 サイト）。ぶつかるボタンはサイトごとに違うからです。ドラッグでずらしたサイトでは、自動でよける判定をしません。人が手で決めた場所が勝ちです。

### 位置は毎フレーム読むが、数は増やさない

欄は、いろいろな理由で動きます。上に要素が差し込まれる、サイドバーが開く、CSS のトランジション。イベントでは拾えない動きも多いので、表示中のマイクについては、`requestAnimationFrame` で毎フレーム `getBoundingClientRect()` を読んでいます。

これが重くならないのは、表示するマイクを最大 12 個にしているからです。ページに欄が 100 個あっても、毎フレームの仕事は 12 回の読み取りで済みます。どの欄が画面に入っているかを探す `querySelectorAll` のほうは、DOM の変化とスクロールの後に 200ms ためてから走らせます。スクロールのイベントは、どのスクロール領域から来ても拾えるように、document のキャプチャ段階で聞いています（scroll はバブリングしないので）。

使わなくなったマイクの箱は、捨てずにプールして使い回しています。スクロールするページでは、欄が出たり消えたりが絶えないからです。

### パネルと波形

マイクにポインタを 350ms 乗せるとパネルが開き、離れて 500ms で閉じます。通り過ぎただけでは開かず、パネルとマイクの間の隙間を横切っても閉じない、という時間です。パネルが画面の右や下からはみ出しそうなら、左や上に向きを変えて開きます。

録音の始め方は、設定で 2 つから選べます。既定は「マイクを押すと始まる」。もう 1 つは「ポインタを乗せてパネルが開いたら始まる」で、これも実機で使って欲しくなった機能です。ホバーでパネルが開くのは 350ms とどまったときだけなので、通り過ぎただけで録音が始まることはありません。録音中はパネルを閉じません。止めるボタンと波形が消えてしまうからです。録音中に別の欄のマイクへポインタが行っても、パネルは付いていきません。

パネルには「× / 送信 / マイク」が並びます。many-ai-cli の入力バーと同じ並びです。× は欄を空にするボタンで、欄に文字があるときだけ出ます。

録音中に動く青い波形は、**マイクの音量では動いていません**。認識のイベント（音が始まった、話し始めた、話し終えた、文字が増えた）で目標の値を決めて、そこへなめらかに寄せています。

```ts:packages/extension/src/ui/waveform.ts
export const ACTIVITY_TARGETS = {
  soundstart: 0.55,
  speechstart: 0.9,
  speechend: 0.25,
  audioend: 0.03,
};
export const TRANSCRIPT_TARGET = 0.85; // 文字が増えたら、少なくともここまで持ち上げる
export const SMOOTHING = 0.18;         // 1 フレームで、目標との差の 0.18 倍だけ寄る
export const BAR_COUNT = 48;
```

音量で動かさない理由は、前に書いたマイクの取り合いです。波形のためだけに `getUserMedia` でマイクを開くと、Web Speech API とマイクを奪い合って、波形は元気に動くのに文字が一向に来ない、という最悪の状態になります。many-ai-cli で踏んだ穴を、拡張でも避けています。数値も many-ai-cli の実装から移しました。話し始めや文字が増えたときには、3 分の 1 秒で消える「蹴り」を足して、声に反応している感じを出しています。`prefers-reduced-motion` のときは、アニメーションせずに 1 回だけ描きます。

### サイトごとに切る。でも、どのサイトかは background に教えない

合わないサイトでは、ツールバーの vtype のアイコンを押すと、そのサイトで vtype を切れます。切ったサイトでは、content script が何も置きません。除外リストを先に読んでから始めるので、切ったサイトには要素が 1 つも差し込まれません。

面白いのは、background がどのサイトかを知らないまま、これをやっていることです。vtype は `tabs` 権限もホスト権限も持っていないので、background はタブの URL を読めません。アイコンが押されたら、background はそのタブのトップのフレームに「切り替えて」とだけ送ります。サイトのオリジンを知っているのは、そのページの中の content script です。iframe に送らないのは、iframe は自分の（別の）オリジンについて答えてしまうからです。

content script がいないページ（`chrome://` のページや、vtype を入れる前から開いていたタブ）では、誰も返事をしません。そのときは設定ページを開いて、除外リストを直接触れるようにしています。

1 つ落とし穴がありました。Chrome は、誰も返事をしなかったメッセージの送信を、届いていても失敗として報告してきます。そのままだと、アイコンを押すたびに設定ページが開いてしまう。そこで、ページ側から明示的な返事（ACK）を返させて、それだけを「届いた」と数えています。

### 言語はファイル 1 本で足せる

拡張の表示は英語と日本語で、ブラウザの言語に合わせて切り替わります。文言は `_locales/<言語>/messages.json` の 1 か所にしかありません。Chrome が manifest の `__MSG_*` のために読むのと同じファイルを、ビルドのときにバンドルの中にも取り込んでいます。

言語を足すのは、ファイルを 1 本置くだけです。キーが既定の言語（英語）と揃っていなければ、ビルドとテストとストア用の検査が、全部止めます。訳し忘れたキーがあると、翻訳されたページの中に英語の文が 1 つだけ混ざります。それを不具合として報告してくれる人はいないので、機械に止めさせています。

## デスクトップ版の分解: Chrome を裏で借りて、前面のアプリに打ち込む

拡張が動くのは、Chrome で開いた Web ページの中だけです。エディタやチャットのアプリでも話したくなって、デスクトップ版を作りました。

### 最初は、拡張とつなぐ作りだった

最初の版は、今とまったく違う形でした。

認識は Chrome の vtype 拡張がやる。結果を Chrome の **Native Messaging** で Rust の常駐アプリへ渡し、常駐アプリが前面のアプリへ打ち込む。拡張はもう動いているので、つなぐだけ。筋は良さそうに見えました。

9 月 22 日のコミットの時刻を見ると、昼前に Rust の骨組みを置いて、昼過ぎには Windows・macOS・Linux の 3 つの実装と、配布の準備まで入っています。

その日のうちに、作り直しました。

入れる人の手順が、「アプリを入れる」「拡張を入れる」「拡張の設定でデスクトップ連携を有効にする」の 3 つになる。自分で並べてみて、アプリも拡張も入れさせるのは二度手間で分からない、と思いました。

もう 1 つ、Microsoft Store の問題もありました。Native Messaging の登録を書くために、MSIX に `unvirtualizedResources` という制限付きの機能が要っていたのです。Microsoft の文書には、この機能は Microsoft と提携先が出す一部の PC ゲームなどのためのもので、それ以外の用途は想定していない、とあります。審査の一番の障害になりそうでした。

https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/app-capability-declarations

### アプリ 1 つで動かす: 画面の外の Chrome で認識する

作り直した形は、こうです。

- vtype の常駐本体（Rust）が、`127.0.0.1` で小さな HTTP と WebSocket のサーバーを開き、認識用のページを配る
- vtype 専用のプロフィールで Chrome を起動し、そのページを画面の外の窓で開く
- ページが Web Speech API で認識し、結果を WebSocket で常駐本体へ返す
- 常駐本体が、前面のアプリへ打ち込む

拡張は要りません。入れる人の手順は、「vtype を入れる」「最初の 1 回だけ同意して、マイクを許可する」の 2 つになりました。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/08_2026-09-24_vtype-architecture_illustration-backstage-chrome.png)

表で打っているアプリのために、画面の袖で Chrome が黙って聞き取りを続けている。そういう役割分担です。

これが本当に動くかは、作る前に測りました（`spike/hidden-chrome-speech`）。一時プロフィールの Chrome を `--app` で起動し、窓を画面の外（`--window-position=-32000,-32000`）に出して、認識結果を `127.0.0.1` の WebSocket で JSONL に書き残す。結果は、**画面の外で 600 秒、確定結果が届き続けました**。

測るのには、少し手間取りました。人が 10 分話し続けるわけにはいかないので、最初は Chrome のテスト用の起動オプション（`--use-file-for-fake-audio-capture`）で、WAV を偽のマイク入力として流そうとしました。ところが、その偽の入力は `getUserMedia` には届くのに、Web Speech API には届かなかった。結局、Windows の読み上げで作った WAV をスピーカーで再生して、本物のマイクに聞かせて測っています。最小化した窓、通常の窓、Edge では、10 分の計測はしていません。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/09_2026-09-24_vtype-architecture_fig5.png)

図の左から右へ、ショートカットを押してから文字が入るまでの流れです。常駐本体と Chrome の間のやり取りは、PC の外に出ません。外へ出るのは、Chrome が Google の音声認識へ送る声だけです。

### 認識ページは、拡張の offscreen のコードをそのまま使う

ここが、共通化が一番効いたところです。

デスクトップ版の認識ページ（`packages/extension/src/speech/speech.ts`）は、拡張の offscreen document の `createOffscreen` を、そのまま呼んでいます。セッションとサイクルの管理、無音 3 回での終了、最後の一言を待つ 1.5 秒、カタカナの辞書。全部、拡張と同じコードです。新しく書き直したものはありません。

違うのは、`createOffscreen` に渡す `chrome.runtime` だけです。`createOffscreen` はもともと `chrome.runtime` を引数で受け取る作りにしてあった（テストのためでした）ので、そこに「中身は WebSocket」という偽物を渡しています。

```ts:packages/extension/src/speech/speech.ts
const offscreen = createOffscreen({
  chrome: {
    runtime: {
      // offscreen が background へ送るつもりのメッセージを、WebSocket で常駐本体へ
      sendMessage: async (message) => {
        fromOffscreen(message);
        return undefined;
      },
      // 常駐本体から来た指示を、offscreen の onMessage へ
      onMessage: { addListener: (l) => void offscreenListeners.push(l) },
    },
  },
});
```

認識ページが拡張のパッケージの中にある理由が、これです。offscreen のコードと同じビルドで、同じ依存（vtype-core と kuromoji）を使って焼けるから。拡張の `build.mjs` に入口を 1 つ足しただけで、出力先は拡張の `dist/` ではなく `dist-desktop/` です。ストアに出す zip には入りません。

やり取りする JSON も、最初の版の Native Messaging のときと同じ語彙を残しました。Rust の型の名前も `ToExtension` / `FromExtension` のままです。拡張はもう関係ないので名前としてはおかしいのですが、付け替えると差分が大きくなるので、あえてそのままにしています。Rust と TypeScript の両側が、同じ見本のファイル（`nm-messages.json`）でテストされています。作り直しを数時間で終えられたのは、たぶんこの 2 つ（offscreen のコードをそのまま使う、やり取りの形を変えない）のおかげです。

Rust の側は `build.rs` で、この `dist-desktop/`（認識ページと辞書）を実行ファイルに埋め込みます。デスクトップ版の実行ファイル 1 つの中に、TypeScript で書いた認識ページが入っているわけです。

ついでに、デスクトップ版が表示する文言も、拡張の `_locales/*/messages.json` から取っています。`build.rs` が `native_` で始まるキーだけを抜き出して Rust のコードにし、英語と日本語でキーが揃っていないとき、ソースが使っているキーが辞書に無いときは、ビルドを失敗させます。

```rust:packages/native/build.rs
for (key, entry) in json.as_object().expect("messages.json must be an object") {
    if !key.starts_with("native_") {
        continue;
    }
    // ...
}
// en と ja で native_ のキーが違えば panic!
// ソースが t("native_…") で使っているキーが辞書に無ければ panic!
```

キーの打ち間違いを放っておくと、画面にキーの名前そのものが出ます。それをビルドで止めています。文言の置き場所は、拡張とデスクトップ版を合わせて 1 か所です。

### 127.0.0.1 のサーバーの守り方

PC の中とはいえ、ポートを開いてページを配る以上、ほかのプログラムや、ブラウザで開いている別のサイトから触られないようにする必要があります。`speech_host.rs` は、依存を増やさないよう `std::net` で最小限に書いていて（WebSocket だけ `tungstenite` を足しました。TLS 無し、同期で使える最小の機能だけ）、守りは次のとおりです。

- 待ち受けは `127.0.0.1` だけ。すべての網には開かない
- パスに、起動のたびに作る**合言葉**を入れる（`/t/<合言葉>/speech`）。OS の暗号用の乱数から 128 ビット（16 進で 32 桁）
- 合言葉が違う、無い、ファイルが無い、のどれにも**同じ 404** を返す。1 文字ずつ当てていくことをさせないため
- WebSocket は、`Origin` が `http://127.0.0.1:<ポート>` と完全に一致するときだけ受ける。ブラウザで開いているどこかのサイトが、つないでくるのを防ぐ
- `Host` ヘッダーが `127.0.0.1` か `localhost` でなければ、403
- 同時接続は 16 まで。リクエストの頭は 8 KB、5 秒まで

```rust:packages/native/src/speech_host.rs
/// 32 hex digits: 128 bits from the OS's secure random source. The token is what keeps other
/// local programs and web pages away from the page, so it must not be guessable.
fn new_token() -> Result<String> {
    let mut bytes = [0u8; 16];
    getrandom::fill(&mut bytes).map_err(|e| anyhow!("no secure random source for the speech page token: {e}"))?;
    Ok(bytes.iter().map(|b| format!("{b:02x}")).collect())
}
```

合言葉の最初の実装は、暗号用ではない乱数で作っていました。同じ日のうちに、OS の暗号用の乱数へ直しています（コミット `846a95a`）。合言葉は、ほかのプログラムやページを遠ざける唯一の鍵なので、推測できてはいけない。書いてから気づいた、というやつです。

同時接続の上限や `Host` の確かめ方は、リリースの直前に回した監査で足したものです。PC の中だから大丈夫、で済ませず、ここは外から叩かれる前提で固めました。

ポートは乱数ではなく、`47213`〜`47215` の 3 つから空いているものを使います。これには理由があります。Chrome はマイクの許可をオリジンごとに覚えていて、**オリジンにはポート番号が含まれる**。起動のたびにポートが変わると、そのたびに許可が消えてしまうからです。

### Chrome の起動: 専用プロフィールで、--app で、画面の外に

Chrome を起動する引数は、これで全部です。

```rust:packages/native/src/chrome_launch.rs
let mut args = vec![
    format!("--user-data-dir={}", profile_dir.display()),  // vtype 専用のプロフィール
    format!("--app={url}"),                                 // アドレスバーの無いアプリの窓
    "--no-first-run".to_string(),
    "--no-default-browser-check".to_string(),
];
match placement {
    WindowPlacement::Hidden => {
        args.push("--window-position=-32000,-32000".to_string());  // 画面の外
        args.push("--window-size=400,300".to_string());
    }
    WindowPlacement::Visible | WindowPlacement::Kept => {
        args.push("--window-position=240,160".to_string());
        args.push("--window-size=560,440".to_string());
    }
}
```

使わないと決めたものもあります。`--remote-debugging-port` と headless モードです。テストで「引数にこの 2 つが入っていないこと」まで確かめています。普段使っている Chrome のプロフィールには、一切触りません。認識用のプロフィールは vtype のデータフォルダの中にあり、Google へのログインも同期もしていません。

最初の 1 回だけは、窓を画面の中に出します。同意とマイクの許可は、人が見て押すものだからです。済んだら、窓を画面の外へ動かしたい。

ところが、ページから `window.moveTo` で画面の外へ動かそうとしても、**Chrome が画面の中へ引き戻します**。

そこで、ページには自分を閉じさせて、常駐本体が画面の外の位置で起動し直す形にしました。逆に、画面の外で開いたページが「同意かマイクの許可が足りない」と気づいたら、やはり自分を閉じて、常駐本体が画面の中へ起動し直します。窓の置き場所を変えたいときは、閉じて開き直す。回りくどいですが、これが一番確実でした。

ページが自分の状態を知らせる手段は、URL のクエリです。`consent=1`（同意済み）、`setup=1`（初回の窓。同意したら閉じる）、`stay=1`（閉じない窓）。常駐本体が起動のたびに組み立てます。

### マイクの許可は、Chrome の設定ファイルに書く

ここで、もう 1 つ壁に当たりました。

画面の中の窓でマイクの許可を聞くと、Chrome のいつものダイアログが出ます。そこで「今回のみ許可」を選ぶと、窓を閉じて画面の外で起動し直すたびに、許可が消えます。そして、同意の窓が何度でも開き直してしまいました。通しで試していて、何回目かでさすがに笑いました。

考えてみれば、デスクトップアプリを使う人にとって、「このサイトにマイクを許可しますか」というサイト向けの確認そのものが、意味を持ちません。対象はウェブサイトじゃない。vtype を使うと決めて、vtype の同意画面で同意した時点で、答えは出ています。

そこで、同意の後は、常駐本体が専用プロフィールの `Preferences` に、このページのマイクの許可を直接書くことにしました。Chrome の「サイトにアクセスしている間は許可」を選んだときに書かれるのと同じ形です。

```rust:packages/native/src/chrome_launch.rs
pub fn grant_microphone(mut prefs: Value, origin: &str, now_us: u64) -> Value {
    // profile.content_settings.exceptions.media_stream_mic の下へ（途中が無ければ作る）
    // ...
    node[format!("{origin},*")] = json!({ "setting": 1, "last_modified": now_us.to_string() });
    prefs
}
```

Chrome は終了するときに `Preferences` を書き直すので、書くのは Chrome がそのプロフィールを使っていない間だけです。初回の窓が閉じてから 3 秒待って書き、それから画面の外で起動し直します。Chrome の時計は 1601 年 1 月 1 日からのマイクロ秒なので、Unix 時刻に `11_644_473_600_000_000` を足しています。ほかの設定（別のサイトの許可など）は消さずに残すことも、テストで確かめています。

これは Chrome の内部の設定ファイルに頼る方法なので、Chrome の版が上がって形が変わると、効かなくなるかもしれません。そのときのための逃げ道も残しました。許可が効かなかったら、その回は画面に見えたまま使う窓で 1 回だけ聞き、同じことを繰り返しません。

### 同意画面は、Store の決まりから

最初の録音の前に、「話した声は Google Chrome の音声認識によって Google へ送られる」ことを説明して、「同意して始める」を押してもらいます。Microsoft Store の方針 10.5.2 には、利用者の個人情報を外部のサービスへ送るなら、その前に明示的な同意（opt-in）を取るように、とあります。声はそれに当たると考えました。

https://learn.microsoft.com/en-us/windows/apps/publish/store-policies

Store 版だけでなく、全部の OS、全部の入れ方で同じ同意を取っています。同意していないのに録音を頼まれたら、ページは何も認識せずに `consent-required` を返します。同意は `config.json` に残ります。同じ画面で「サインインしたら vtype を起動する」も選べます（最初はチェックが入っています）。

画面の外の窓にも、放っておくとタスクバーのボタンが出ます。vtype の居場所はトレイなので、見えない窓のボタンが並んでいる理由はありません。Windows では、Chrome が窓を出したらすぐにツールウィンドウの扱いへ切り替えて、タスクバーから外しています。

macOS と Linux では、まだ同じことをしていません。なので、見えない窓のボタン（macOS なら Dock のアイコン）が残ります。押しても見えるものは何も無いので、ページの側に手を打ちました。その窓が「一度フォーカスを失ってから、またフォーカスを得た」ときは、設定ページを開く。起動した直後に Chrome が窓にフォーカスを渡すことがあるので、最初のフォーカスは数えません。Windows でも、切り替えが間に合わなかったときの保険として、同じ仕組みが残っています。

設定ページは、同じローカルのサーバーが配るページを、**もう 1 つ別のプロフィール**の Chrome で開きます。最初は認識ページと同じプロフィールで開いていたら、設定の窓が、認識ページの画面外の位置と大きさ（400×300）を引き継いでしまいました。開き直すたびに 1 枚ずつ増えて、気づいたら画面の外に 4 枚たまっていた。見えないところで増えるのが、一番たちが悪いです。

## 文字を打ち込むところは、OS ごとに全部違う

認識までは共通でも、出口は OS ごとにまるで違います。デスクトップ版では、OS に触る部分を全部 `Platform` という trait の後ろに隠しました。

```rust:packages/native/src/platform/mod.rs
pub trait Platform: Send + Sync {
    /// Runs the OS event loop on the calling (main) thread until `quit` is called.
    fn run_event_loop(&self, events: Sender<PlatformEvent>) -> Result<(), PlatformError>;
    fn register_hotkey(&self, spec: &str) -> Result<(), PlatformError>;
    fn inject_text(&self, text: &str, method: InjectMethod) -> Result<InjectOutcome, PlatformError>;
    fn focused_field(&self) -> FieldInfo;
    fn show_icon(&self, state: IconState, position: Option<(i32, i32)>);
    fn show_bubble(&self, text: &str);
    fn launch_chrome(&self, args: &[String]) -> Result<(), PlatformError>;
    // ほかに、トレイ、自動起動、クリップボード、テンプレートの一覧など
}
```

判断は全部、常駐本体の `Core`（`daemon.rs`）が 1 本の作業スレッドで下し、OS にはこの trait を通してしか触りません。メインスレッドは OS のイベントループに渡します（トレイや常駐の窓が、それを必要とするので）。こうしておくと、`Core` の判断を、偽物の `Platform` でテストできます。開発機が Windows だけなので、これはかなり助かっています。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/10_2026-09-24_vtype-architecture_fig6.png)

表のとおり、同じ「文字を入れる」でも、OS ごとに使う仕組みも、だめだったときの逃げ方も違います。共通しているのは、パスワード欄には入れないことと、最後の手段がクリップボードであることの 2 つです。

### Windows: SendInput の Unicode 入力、だめなら貼り付け

Windows では、`SendInput` に `KEYEVENTF_UNICODE` を付けて、UTF-16 の単位ごとにキーを押して離します。キーボードの配列に関係なく、日本語も絵文字もそのまま入ります。絵文字のようなサロゲートペアは、2 つの単位を順に送ります。改行と Tab だけは、本物の Enter と Tab のキーとして押します。

```rust:packages/native/src/platform/windows/inject.rs
KeyAction::Unit(unit) => {
    inputs.push(key_input(0, unit, KEYEVENTF_UNICODE));
    inputs.push(key_input(0, unit, KEYEVENTF_UNICODE | KEYEVENTF_KEYUP));
}
KeyAction::Virtual(vk) => {  // Enter と Tab
    inputs.push(key_input(vk, 0, 0));
    inputs.push(key_input(vk, 0, KEYEVENTF_KEYUP));
}
```

`SendInput` は、受け付けたイベントの数を返します。管理者権限で動いている窓（UIPI）などで弾かれると、数が足りなくなる。全部通らなかったときは、クリップボードに文字を入れて Ctrl+V を押す方法に切り替えます。

貼り付けたら 300ms 待って、クリップボードを元に戻します。元が文字でも画像でも戻し、何も無かったら空にします。声で入れた文字が、クリップボードに残り続けないようにするためです。打ち込み方は設定で「自動（打つ、だめなら貼る）」「打つだけ」「貼るだけ」から選べます。

パスワード欄は、UI Automation で前面の要素を取り、`CurrentIsPassword` で見ます。加えて、Windows の資格情報の入力画面（`credentialuibroker.exe`、`consent.exe`、`logonui.exe`）には入れません。

### macOS: CGEvent に文字列を載せる

macOS は、`CGEvent` のキーボードイベントに、`keyboard_set_unicode_string` で文字列を載せます。1 回のイベントで UTF-16 の 20 単位までなので、長い文は区切って送ります。イベントの間には 2ms の息継ぎを入れます。忙しいアプリがイベントを落とさないように。

気を付けたのは修飾キーです。ショートカットの Control や Option がまだ押されたままのタイミングで文字を送ると、文字がショートカットとして解釈されてしまいます。なので、送るイベントには修飾キーのフラグを付けません。

アクセシビリティの許可が無いと、macOS はこれらのイベントを黙って捨てます。エラーにもなりません。なので、許可があるかを先に確かめています。実行ファイルに署名していないので、更新の後にもう一度許可が要ることがある、というのも README に書いてあります。

打てなかったときは、Windows と同じく、クリップボードと ⌘+V に切り替えます。パスワード欄は、アクセシビリティ API で前面の要素を取り、その種類が `AXSecureTextField` かどうかで見ています。

### Linux: X11 は XTest、Wayland は 3 段構え

X11 は XTest で打ちます。やり方は xdotool と同じで、今のキーボードでは何も入力しないキーに、打ちたい文字を一時的に割り当てて押し、終わったら元に戻します。だから、キーボードの配列に関係なく、どの文字でも打てます。打てなかったときは、クリップボードと Ctrl+V です。

問題は Wayland で、Wayland ではアプリがほかのアプリへキーを送ることが許されていません。そこで、3 つの経路を順に試します。

1. デスクトップのリモートデスクトップ用のポータル（`ashpd`）で、クリップボードに入れて Ctrl+V を押してもらう。初回は、デスクトップが許可を求めてくる
2. `ydotool` が入っていて、デーモンが動いていれば、それで打つ。ただしキーボードの配列を通して打つので、英字（ASCII）だけ
3. どちらもだめなら、クリップボードに入れて「Ctrl+V で貼ってください」と知らせる

どの経路で何が失敗したかは、診断情報に残します（文字そのものは残しません）。パスワード欄の判定は AT-SPI で、デスクトップのアクセシビリティ機能がパスワード欄だと判別できる範囲に限られます。

この順番と「何が失敗したか」の記録は、実際の経路から切り離したロジックとして書いてあるので、Windows の開発機でもテストが回ります。

```rust:packages/native/src/wayland_inject.rs
pub trait WaylandRoutes {
    fn portal_paste(&mut self, text: &str) -> Result<(), String>;
    fn ydotool_type(&mut self, text: &str) -> Result<(), String>;
    fn copy(&mut self, text: &str) -> Result<(), String>;
}
// inject_wayland() が、この 3 つを順に試して、試した跡（trail）と結果を返す
```

### ショートカットと、コマンドライン

ショートカットは `global-hotkey` の crate で登録します。既定は、Windows と Linux が Ctrl+Alt+Space、macOS が Control+Option+V です。設定ページで変えられます。

GNOME の Wayland では、アプリが自分でキーを横取りできません。そこで `vtype install` が、GNOME のカスタムショートカットとして「`vtype toggle` を実行する」を登録します。

`vtype toggle` のようなコマンドは、動いている常駐本体へ指示を送るだけです。Windows は名前付きパイプ、それ以外は Unix ソケットで、1 行に 1 つの JSON を流します。なので、どのランチャーからでもキーに割り当てられます。README には、AutoHotkey v2 の例を載せています。

```ahk
^!k::Run "vtype mode kana"
```

引数なしで `vtype` を起動すると、常駐を始めます。すでに動いていれば、代わりに設定ページを開きます。zip から `vtype.exe` をダブルクリックした人と、Store のスタートメニューから開いた人が、同じ動きになるようにしました。

### 途中の結果は、打ち込まない

拡張とデスクトップ版で、はっきり振る舞いを変えたところがあります。

拡張は、話している途中の結果を、欄の中で書き換えます。デスクトップ版は、**確定した結果だけ**を打ち込みます。途中の結果は、マイクの上の吹き出しに出すだけです。

理由は単純で、ほかのアプリへ打ち込んだ文字は、後から書き換えられないからです。拡張は欄の中身を読めるし、前後の文字を覚えておけるので、途中の結果を置き換えられる。デスクトップ版がやっているのは、キーを押すことだけです。打った文字を消すには、バックスペースを押すしかない。IME が動いているアプリや、補完が効くエディタでそれをやったら、何が起きるか分かりません。

もう 1 つ、Chrome は 1 回話すごとに認識を終えるので、1 回の録音から確定結果がいくつも出ます。英語の確定どうしがくっつかないよう、前に打った最後の文字と、次の最初の文字がどちらも英数字なら、間に半角スペースを入れます。拡張と同じ考え方を、Rust でもう一度書いています。

## デスクトップ版の UI と、使い方の工夫

### 浮かぶマイクは、フォーカスを奪わない

デスクトップ版の顔は、画面の右下あたりに浮かぶ、丸いマイクです。

これは、フォーカスを**絶対に取らない**窓でできています。Windows なら、`WS_EX_LAYERED | WS_EX_TOPMOST | WS_EX_NOACTIVATE | WS_EX_TOOLWINDOW`。クリックしても、文字を打ち込みたいアプリがフォーカスを持ったままでいられます。これが崩れると、マイクを押した瞬間に、打ち込み先がマイク自身になってしまう。音声入力のアプリとしては致命的です。

絵は、OS の描画機能ではなく、`tiny-skia` でソフトウェア描画しています。どの OS でも、同じピクセルになるようにするためです。記号もストロークで描いているので、フォントが要りません。

- 待機中: 不透明度 0.4 の灰色の円。ポインタが乗ると不透明に
- 録音中: オレンジになり、真ん中がマイクから白い ■ に変わる。押せば止まる
- 打ち込んだ直後: 緑のチェック

四隅には、小さなボタンがあります。左上がテンプレート、右上が入力モード（押すたびに 通常 → 英語 → カタカナ）、右下が送信、左下が欄を空にする。入力モードの印は、英語が「A」、カタカナが「カ」、通常は日本語の画面なら「あ」、英語の画面なら地球儀です。通常のモードはブラウザの言語で聞くので、英語の画面の人に「あ」を見せても意味が無いからです。

ポインタが乗ったボタンは、少し明るく大きくなります。0.7 秒とどまると、そのボタンが何をするかを吹き出しで教えてくれます。すぐに出すと、ポインタが通り過ぎるたびに吹き出しがちらつくからです。

マイクの大きさは、Ctrl を押しながらマウスのホイールで、0.5 倍から 5 倍まで変えられます（macOS は ⌘ でも）。最初は 2 倍までだったのを、使ってみて 5 倍に広げました。

話している間は、マイクの周りに波紋が広がります。これも拡張の波形と同じで、**マイクの音量では動いていません**。認識のイベント（音、話し始め、話し終わり、文字が来た）で動かしています。デスクトップ版の場合は、そもそもマイクを開いているのが Chrome で、vtype ではないからです。

```rust:packages/native/src/ripple.rs
impl VoiceCue {
    /// The Web Speech API event names the speech page forwards; the others do not move the ripple.
    pub fn from_activity(activity: &str) -> Option<VoiceCue> {
        match activity {
            "soundstart" => Some(VoiceCue::Sound),
            "speechstart" => Some(VoiceCue::Speech),
            "speechend" | "soundend" => Some(VoiceCue::SpeechEnd),
            _ => None,
        }
    }
}
```

拡張の波形（canvas・TypeScript）と、デスクトップ版の波紋（tiny-skia・Rust）。描き方は違っても、決まりは同じです。「共通にした見た目と動きの考え方」の中身は、これです。

### 知らせは、OS の通知ではなく、マイクの吹き出しで

「つないでいます…」「パスワード欄には入れません」「クリップボードに入れました。Ctrl+V で貼ってください」。こういう知らせは、OS の通知ではなく、浮かぶマイクの上の吹き出しに出します。

最初は OS の通知を使っていました。でも、通知が出るかどうかは OS の設定次第で、出ないこともある。打ち込み先のすぐ横にあるマイクに出すほうが、目に入ります。マイクが画面に無いとき（Wayland、マイクを非表示にしているとき、全画面のアプリの上）だけ、OS の通知に落とします。吹き出しの文字は、OS のフォントで描いています。

### 入力欄が無いところで話した言葉は、捨てずに取っておく

デスクトップ版で、使ってみて一番「これは要る」と思った機能です。

ショートカットを押して話し始めたのに、実はカーソルが入力欄に無かった。前面にあるのがファイルの一覧やボタンだったら、打ち込んだキーはショートカットとして解釈されて、何が起きるか分かりません。話した言葉も消えます。

そこで、OS が「今フォーカスがあるのは、テキストの入力欄ではない」と言っているときは、打ち込まずに、マイクの上の吹き出しに取っておくことにしました。吹き出しには「コピー」「入力欄に入れる」「✕」が付いています。欄をクリックしてから「入力欄に入れる」を押せば、そこに入る。取っておいた言葉はメモリの中だけに持ち、ディスクには書きません。

OS がどちらとも言えないとき、マイクを非表示にしているとき、Wayland のときは、今までどおり打ち込みます。分からないときに黙って飲み込むほうが、困るからです。

### 話し終えたら、止まる

「話し終えてから何秒で止めるか」を、1〜10 秒で選べます（既定は 3 秒、オフもあり）。新しい言葉が来なくなってから数え始めるので、話し始める前の沈黙は数えません。止まるだけで、送信はしません。

送信のボタンを録音中に押したときは、録音を止めて、最後の言葉が入り終わってから送信キーを押します。先に送信キーを押すと、最後の一言が送信の後に入ってしまうからです。

### マイクが、入力欄の近くへ来る（Windows・実験中）

Windows では、実験中の機能（既定はオフ）として、入力欄をクリックするか Tab で移ると、浮かぶマイクそのものがその欄の近くへ来ます。UI Automation のフォーカスの変化を聞いていて、文字や IME の候補の窓にかぶらない位置を選びます。

欄から離れて 0.7 秒たつと、マイクはポインタのある画面の右下へ戻ります。この 0.7 秒には根拠があります。開発機で測ったところ、フォーカスが欄へ戻ってきた 48 回のうち 13 回は 0.6 秒以内（Tab でボタンを通り過ぎただけ、など）で、残りは 1.2 秒以上でした。その間を取っています。録音中と、ポインタがマイクの上にあるときは戻りません。

ただ、アプリによっては、入力欄にフォーカスが移ったことを OS に報告してくれません。LINE のデスクトップ版は、欄をクリックしても窓のことしか報告してきません。あらためて聞き直すと、欄にフォーカスがあることは分かります。そういうアプリのために、メニューに「このアプリで入力欄の近くに出ない」という調べる項目を付けました。直前に使っていたアプリを 10 秒見張り、その間に欄をクリックしてもらって、どの型か（欄を報告する、窓だけ報告する、聞けば分かる、そもそも欄に見えない、など）を吹き出しで教えます。記録するのは要素の種類やクラス名や位置だけで、中の文字は記録しません。

### テンプレート

よく使う文章は、テンプレートに登録しておけます（最大 100 件）。マイクの左上のボタンで一覧が出て、選ぶと前面のアプリへ入ります。どのアプリでも、文字を選んだままマイクを右クリックして「選択中の文字をテンプレートに登録」を選べば、保存できます。中身は Ctrl+C（macOS は ⌘+C）を押して取っていて、クリップボードは元に戻します。

一覧の各行には、✏（設定ページで編集）と ✕（すぐ削除）があります。削除に確認は出しません。その代わり、消した行に「元に戻す」が残り、一覧は開いたままです。確認のダイアログを挟むより、戻せるほうが速いと思っています。

### 使い方の工夫

作った本人がどう使っているかも、少し書いておきます。

- **AI への指示**: many-ai-cli で並行して動かしている AI への指示は、かなりの部分を声で出しています。話して、送信ボタンで送る。デスクトップ版の送信キーは Enter か Ctrl+Enter（macOS は ⌘+Enter）を選べるので、Enter が改行になるアプリでは Ctrl+Enter にしておくと楽です
- **固有名詞は、置き換え表で**: リポジトリの名前のように、音声認識が知らない語は、置き換え表に登録しておくと楽です。前に書いた呼称索引の話と同じで、発音と正式名の間に 1 段挟むと、言い直しが減ります
- **カタカナは、入力モードで**: カタカナで書きたい語は、カタカナモードに切り替えて話す。AutoHotkey でモードの切り替えをキーに割り当てておくと速いです
- **拡張とデスクトップ版を両方入れるとき**: デスクトップ版の「入力欄の近くにマイクを出す」が Chrome の欄にも出ると、拡張のマイクと 2 つ並びます。設定の「Chrome の入力欄にも出す」を外しておけば、Chrome の中は拡張、外はデスクトップ版、と分けられます

どこでも使えるのはデスクトップ版ですが、Chrome の中なら、話しながら欄の中で文字が書き換わっていく拡張のほうが、自分は好きです。

## テストと CI と、配布

### テスト

拡張と core のテストは vitest です。拡張は DOM が要るので happy-dom を使い、`chrome.*` は自前の偽物（`tests/fake-chrome.ts`）で差し替えています。この記事を書く前に回したところ、core が 31 件、拡張が 461 件、全部通りました。Rust は `#[test]` の関数が 238 個あります。

ビルドそのものも、検査です。

- 拡張: 出力にモジュールの構文が残っていない、参照先のファイルが全部ある、版が揃っている、`__MSG_*` の文言がある、言語ごとのキーが揃っている
- デスクトップ版: `native_` の文言のキーが英語と日本語で揃っている、ソースが使うキーが辞書にある
- ストア用の zip を作る前の検査スクリプトは、拡張の中のファイルを読む `fetch(chrome.runtime.getURL(...))` 以外の通信を含むものを通さない

コミットの前には、秘密情報が紛れ込んでいないかを見る secrets-scan が走ります（pre-commit のフックと、GitHub Actions の両方で）。

### CI は 3 OS。macOS と Linux の保証は、ここまで

GitHub Actions では、Node のジョブで `pnpm -r build` → `typecheck` → `test`、Rust のジョブで Windows・macOS・Ubuntu の 3 つに `cargo fmt --check`、`cargo clippy -- -D warnings`、`cargo test` を回しています。Rust のジョブは、先に Node で認識ページをビルドします。`build.rs` が、それを埋め込むからです。

順番にも意味があります。拡張は `vtype-core` を、ワークスペースのパッケージのビルド済みの出力（`dist/`）から読みます。まっさらな clone には `dist/` が無いので、typecheck やテストより先にビルドを走らせる必要がある。`pnpm -r` はワークスペースを依存の順に回るので、`pnpm -r build` 1 回で、core → 拡張の順になります。

Rust の版は 1.98 に固定しています。新しい stable が出ると clippy の警告が増えて、`-D warnings` の CI が突然赤くなるからです。上げるときは、直しと一緒にわざと上げます。

開発機は Windows だけです。なので、macOS と Linux については、この CI の緑が保証のすべてです。README にも、実機ではまだ動かせていないと、正直に書いています。

### 配布

デスクトップ版は、`native-v*` のタグで release のワークフローが走り、Windows の zip と MSIX、macOS のユニバーサルな tar.gz、Linux の tar.gz と `.deb`、サードパーティのライセンスの一覧、チェックサム、Homebrew の formula を、下書きのリリースにまとめます。npm は、本体と OS ごとの実行ファイルに分けた 5 つのパッケージです。

npm の梱包では、一度やらかしかけました。Windows で梱包すると、Mac と Linux の実行ファイルから実行の印が落ちて、入れても起動しない。今は梱包を Linux の CI でやり、実行の印とハッシュを検査しています。このあたりは前回の記事に書いたので、ここでは省きます。

## まだ分かっていないこと

- **Google の側の上限**: 公式の文書に無い。今のところ当たったことは無い、としか言えません
- **macOS と Linux の実機**: CI の緑まで。Mac や Linux をお持ちの方に試していただけると、とても助かります
- **Preferences に書く許可**: Chrome の内部の形に頼っているので、Chrome の更新で効かなくなるかもしれない。逃げ道の窓はあります
- **リッチエディタでの途中経過の書き換え**: 今のところ崩れたサイトには当たっていないけれど、全部のエディタで試せているわけではありません
- **Firefox**: 認識エンジンを持ち込む必要があり、手を付けていません
- **Microsoft Store**: この記事を書いている時点では、審査の結果待ちです

動いた、動かなかった、どちらでも Issue で教えてもらえると嬉しいです。

## あわせて読みたい

- [並行リポ29個を音声で回す。呼称索引という小さな台帳を1個足しただけの話](https://qiita.com/ishizakahiroshi/items/f2f9a069b1ff641ee43d)（many-ai-cli の音声入力で、ふだん何をしているかの話です）
- [AI CLI を 7 本並べたら、次に要るのは「契約」と「引き継ぎ」と「委譲」だった。many-ai-cli v0.8.0](https://qiita.com/ishizakahiroshi/items/49f6a29a43bf64168c1a)（vtype の出どころ、many-ai-cli の本体の技術の話です）
- [Tauri アプリの Microsoft Store 初回申請で踏んだ4つの罠と回避手順（MSIX / Partner Center）](https://qiita.com/ishizakahiroshi/items/57e9b7933fe375fbb1e8)（MSIX と Partner Center の、1 本目の申請の記録です）

## vtype はこんなときに刺さります

- 1 日何分という上限を気にせず、音声入力を使いたい人
- 検索ボックスでも、リッチエディタの本文でも、iframe の中でも、同じように声で入れたい人
- サイトを開くたびに、マイクの許可を聞かれるのが面倒な人
- ブラウザの外のアプリ（エディタ、チャット、ターミナル）でも、ショートカットで声を入れたい人。こちらはデスクトップ版です

いずれかに心当たりがあれば、拡張は Chrome ウェブストアから、デスクトップ版は `npm i -g @ishizakahiroshi/vtype` か `brew install ishizakahiroshi/tap/vtype` で試せます。どちらもアカウントの登録は要りません。デスクトップ版は、最初に 1 回だけ、小さな窓で同意とマイクの許可があります。

- 紹介ページ（スクショと機能一覧）: https://ishizakahiroshi.com/work.html?id=vtype
- Chrome ウェブストア: https://chromewebstore.google.com/detail/vtype/nngfilimeplngdjdmgkddlhbdjpmikgn
- リポジトリ（Issue / PR 歓迎）: https://github.com/ishizakahiroshi/vtype
- npm: https://www.npmjs.com/package/@ishizakahiroshi/vtype

Star をいただけると、開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

## おわりに

分解してみて改めて思ったのは、認識そのものはほとんど書いていない、ということでした。Chrome に入っているものを呼んでいるだけで、苦労したのは全部、その手前と後ろ。マイクを誰に持たせるか。止まったらどう立て直すか。最後の一言をどう落とさないか。文字をどこへ、どう入れるか。

それと、使ってみて変えたところの多さ。パネルに出してからまとめて入れる案も、クリックしてからマイクを出す案も、拡張とつなぐデスクトップ版も、作ってから使って、やめています。最初の判断が外れるのは、たぶんこの先も変わりません。

外れたら、また直します。

---

📎 図解版・関連リンクをまとめたページがあります:
https://ishizakahiroshi.com/articles/2026/2026-09-24_vtype-architecture/

※ ヘッダー画像とインフォグラフィックの絵は AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
