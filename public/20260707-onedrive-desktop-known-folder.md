---
title: OneDrive を消したのにデスクトップが OneDrive のまま。犯人はレジストリの既知フォルダー設定だった
tags:
  - Windows
  - トラブルシューティング
  - OneDrive
  - レジストリ
private: false
updated_at: '2026-07-07T12:38:42+09:00'
id: 91489b13b6983043e39e
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

Windows 11 で OneDrive をアンインストールした後、エクスプローラーのクイックアクセスを整理していたら、デスクトップの参照先がなぜか二重になってしまいました。「OneDrive を消したのにデスクトップが OneDrive のまま」という、検索してもドンピシャの記事が出てこないやつです。最終的な原因は Known Folder（既知フォルダー）と User Shell Folders というレジストリの設定でした。同じ状態でハマっている人のために、切り分けの流れをそのまま残しておきます。

![OneDrive を消した後にデスクトップが迷子になる話](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260707-onedrive-desktop_hero.png)


## クイックアクセスに「デスクトップ」を登録すると、別のデスクトップが開く

きっかけは本当に些細な整理でした。

クイックアクセスに「デスクトップ」を登録したかっただけです。開いてほしいのは、いま実際に使っている

```
C:\Users\<ユーザー名>\OneDrive\デスクトップ
```

のほう。ところが登録してみると、開くのは

```
C:\Users\<ユーザー名>\Desktop
```

でした。何度登録し直しても `Desktop` に化ける。中身も違うので、素直に「あれ、これ困るな」と。

最初は「クイックアクセスがバグってるんだろう」と思っていました。今思えば、この最初の見立てがずれていたんですが。

## 「Windows が使っているデスクトップ」を確認したら、話が合わない

いったん、Windows 自身がどっちをデスクトップだと思っているのかを確認しました。PowerShell で一発です。

```powershell
[Environment]::GetFolderPath('Desktop')
```

返ってきたのは、

```
C:\Users\<ユーザー名>\OneDrive\デスクトップ
```

OneDrive 側でした。つまり Windows は「デスクトップ ＝ OneDrive のほう」だと思っている。

なのにクイックアクセスから開くと `Desktop` が開く。ここで手が止まりました。片方は OneDrive、もう片方はプロファイル直下。同じ「デスクトップ」という言葉が、参照する先で食い違っている。正直、ここでけっこう混乱しました。

![GetFolderPath とクイックアクセスの参照先が食い違う概念図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260707-onedrive-desktop_fig1.png)

上の図のように、Windows の内部 API が指すデスクトップと、クイックアクセスが開くデスクトップが別々の場所を向いていました。同じ名前なのに宛先が二つある、という状態です。これが「意味が分からない」の正体でした。

## OneDrive の設定から戻そうとして、そこで詰まる

こういうときの定番は「OneDrive の設定 → バックアップ（同期）を停止」で、デスクトップをプロファイル直下に戻す、というやつです。検索するとだいたいこれが出てきます。

でも、その手が使えませんでした。OneDrive はすでにアンインストール済み。タスクトレイの雲アイコンもない。設定を開く入口そのものが存在しないんです。

念のため、アドレスバーが何を指しているかも `Ctrl+L` で見てみました。表示上は「デスクトップ」としか出ませんが、展開すると中身は OneDrive 側。クイックアクセスから開き直したり、フォルダーをドラッグして登録し直したり、思いつくことはひと通り試しましたが、`Desktop` への化けは直りませんでした。ここで 30 分くらい溶かした気がします。

## 見方を変える。「OneDrive が悪い」ではなく「設定だけ残っている」

ここでようやく、疑う場所を変えました。

OneDrive というアプリが悪さをしているんじゃなくて、OneDrive を消したあとに Windows 側の設定だけが取り残されているんじゃないか、と。

Windows には Known Folder（既知フォルダー）という仕組みがあって、「デスクトップ」「ドキュメント」といった特別なフォルダーの実際の場所を、レジストリで持っています。OneDrive はバックアップを有効にすると、この「デスクトップの場所」を OneDrive 配下に書き換えます。問題は、アプリを消してもこの書き換えは元に戻らないこと。参照先の住所だけが OneDrive のまま残るわけです。

その住所が書かれているのが、ここでした。

```powershell
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\User Shell Folders" /v Desktop
```

![User Shell Folders が OneDrive 側を指したまま残る図](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260707-onedrive-desktop_fig2.png)

図にすると単純で、住所録（User Shell Folders）の「デスクトップ」欄が OneDrive 配下を指したまま固まっていた、という話です。アプリを消しても住所録は書き換わらない。だから Windows は OneDrive 側を見にいき続けていました。

## 直し方は 1 行。プロファイル直下に戻す

原因がはっきりすれば、あとは住所を書き戻すだけです。デスクトップの参照先を、標準の `%USERPROFILE%\Desktop` に戻します。

```powershell
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\User Shell Folders" /v Desktop /t REG_EXPAND_SZ /d "%USERPROFILE%\Desktop" /f
```

`REG_EXPAND_SZ` にしておくのがポイントで、`%USERPROFILE%` を環境変数として展開してくれる型です。ここを普通の文字列型にすると `%USERPROFILE%` がそのまま文字として入ってしまうので、型は合わせておきます。

書き換えたら、エクスプローラーを再起動して反映します。

```powershell
Stop-Process -Name explorer -Force
```

:::note warn
`Stop-Process -Name explorer -Force` を実行するとデスクトップやタスクバーが一瞬消えます。多くの環境では自動で復帰しますが、戻らない場合は `Ctrl + Shift + Esc` でタスクマネージャーを開き、「ファイル」→「新しいタスクの実行」から `explorer` と打てば戻せます。作業中のウィンドウは事前に保存しておくと安心です。
:::

これで、クイックアクセスもデスクトップの参照先もそろって正常化しました。あっけないくらいでした。

## この件でわかったこと

- OneDrive を消しても、Windows の Known Folder の設定は元に戻らない。参照先の住所だけが OneDrive のまま残る
- だから「OneDrive が悪い」ではなく「設定だけ取り残された」と捉えると筋が通る
- `GetFolderPath('Desktop')` とクイックアクセスの挙動が食い違ったら、実体（レジストリの参照先）を疑う
- アンインストール済みだと「バックアップ停止」の入口が消えているので、レジストリを直接見にいくのが早い

最初に「クイックアクセスがバグってる」と決めつけたのが、遠回りの原因でした。症状の出ている場所と、設定が壊れている場所が別だっただけ。こういうとき、AI に切り分けを手伝ってもらうと「まず Windows がどっちを見てるか確認しよう」と一歩引けるので、そこは助かりました。

まだ Known Folder まわりは、User Shell Folders と Shell Folders の役割の違いなど、ちゃんと理解しきれていない部分が残っています。次に同じところでつまずいたとき困らないように、時間をとって ProcMon あたりで挙動を追ってみようと思います。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
