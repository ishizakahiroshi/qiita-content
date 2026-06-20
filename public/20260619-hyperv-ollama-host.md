---
title: Hyper-VゲストからホストWindowsのOllamaをAPI経由で使う – many-ai-cli の base_url 設定まで
tags:
  - 個人開発
  - hyperv
  - ollama
  - ローカルLLM
  - many-ai-cli
private: false
updated_at: '2026-06-20T19:24:08+09:00'
id: 01256d3beb6db3a698a7
organization_url_name: null
slide: false
ignorePublish: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-06-19_hyperv-ollama-host_hero.png)

## 結論から

Hyper-V のゲスト Windows で Ollama の GPU 推論を動かそうとして 4 回失敗しました。
解決策は「ゲストで動かすのをやめる」でした。

- ホスト Windows: Ollama とモデルを置いて GPU で推論する
- ゲスト Windows: many-ai-cli と開発環境を置く
- ゲストからホストの Ollama API に HTTP で接続する

many-ai-cli の設定:

```yaml
ollama:
  base_url: "http://<host-ip>:11434"
```

この `base_url` をモデル一覧取得と CLI 起動時の両方で使います。

---

## そもそも、ゲストの中で全部やろうとしていた

やりたかったことは単純です。

Hyper-V のゲスト Windows に開発環境を置いて、その中でローカルLLMも動かす。many-ai-cli から Claude Code や Codex を起動して、モデルピッカーで Ollama Local を選んだら、そのまま手元の LLM が返事をする。

うまくいけば、かなり気持ちいい構成です。開発環境はゲストに閉じる。ホスト側は汚さない。失敗したら仮想マシンごと戻せる。

でも、現実はそうきれいにいかなかった。

Ollama は入る。モデルも入る。API もそれっぽく返る。けれど、GPU まわりで詰まる。ドライバなのか、仮想化なのか、Windows の機嫌なのか、ひとつ直したつもりで別のところが怪しくなる。

これを 4 回やった。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-06-19_hyperv-ollama-host_fig1.png)

## 考え方をひっくり返した

途中で、ふと気づきました。

ローカルLLMを「ゲストの中で動かす」必要は、本当にあるのか。

欲しいのは、ゲスト Windows の中から LLM を使えることです。Ollama のプロセスがどこで動いているかは、実はそんなに重要ではない。

だったら、GPU を持っているホスト Windows で Ollama を動かせばいい。ゲストは、その API に HTTP でつなぐだけにする。

- ホスト Windows: Ollama とモデルを持つ。GPU で推論する
- ゲスト Windows: many-ai-cli と開発環境を持つ
- ゲストからホストの Ollama API に接続する

これなら、GPU をゲストに渡す話をしなくていい。Ollama はホストの普通の Windows アプリとして動く。ゲストはただのクライアントになる。

これは効いた。

## localhost を信じすぎていた

many-ai-cli 側にも、ひとつ問題がありました。

Ollama Local は、もともと `localhost:11434` を見に行く前提でした。普通にホスト OS 上で使うなら、それで正しい。Ollama も Hub も同じマシンにいるからです。

でも、Hyper-V ゲストの中で many-ai-cli を動かしていると、`localhost` はゲスト自身を指します。ホスト Windows ではない。

ここでだいぶ混乱します。

ホスト側では Ollama が元気に動いている。ブラウザや PowerShell から叩けば返る。なのにゲストの many-ai-cli から見ると、Ollama がいないことになる。

「ローカル」と言っているけど、誰から見たローカルなのか。ここを間違えると、ずっと違う住所を見に行く。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/2026-06-19_hyperv-ollama-host_fig2.png)

## many-ai-cli 側の設定変更

`ollama.base_url` という設定を足しました。

```yaml
ollama:
  base_url: "http://<host-ip>:11434"
```

この値を 2 か所で使います。

- モデル一覧を取るときは `<base_url>/api/tags` を見る
- Claude Code や Codex を Ollama route で起動するときは、子プロセスに `<base_url>` を渡す

Codex は OpenAI 互換 API として見るので、内部では `/v1` を足します。Claude Code は Anthropic 互換 API として見るので、`/v1` は足しません。

設定値を一か所に寄せたことで、ゲストからホストの Ollama を自然に使えるようになりました。モデルピッカーにもホスト側の pull 済みモデルが出る。起動した Codex も、ゲスト内の空っぽの localhost ではなく、ホスト側の Ollama に向かう。

## Firewall はちゃんと絞る

この構成でいちばん気をつけるのは、Ollama API を外へ開けすぎないことです。

ホスト Windows 側の Ollama は、ゲストから届くように待ち受けを広げる必要があります。ただし、インターネットに出すものではない。

やること:

- Ollama をホストのネットワークインターフェイスで待ち受ける
- Windows Firewall で Hyper-V のゲスト側ネットワークだけ許可する
- ルーターのポート転送はしない
- ゲスト側から疎通確認して、many-ai-cli の `ollama.base_url` を設定する

LLM の API は、手元の道具として使うぶんには便利だけど、外から誰でも叩ける場所に置くものではない。

## まとめ

| 役割 | 場所 | 担当 |
|---|---|---|
| GPU 推論 | ホスト Windows | Ollama |
| 開発環境 | ゲスト Windows | many-ai-cli, Claude Code 等 |
| 接続 | Hyper-V 内部ネットワーク | HTTP API |

Hyper-V ゲストで GPU を使うのは、ドライバと仮想化レイヤーの問題が複合して難しいです。

「ゲストで全部やる」をやめて「ゲストはクライアント、ホストが推論係」に分けたら、考えることも分かれてシンプルになりました。

- `localhost` は、いま動いている側から見た自分自身
- GPU を仮想マシンに持ち込むより、API 越しに借りるほうが早いことがある
- 設定値は、モデル一覧取得と実行時 env の両方で同じものを使わないとズレる
- ネットワーク越しにするなら、Firewall の範囲は必ず絞る

---

書いた人: ishizakahiroshi
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X : https://x.com/ishizakahiroshi
