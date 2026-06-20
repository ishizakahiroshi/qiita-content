---
title: "note だけに書いていた記事スキルを Qiita・Zenn にも対応させた話"
tags:
  - ClaudeCode
  - Claude
  - 生成AI
  - AIエージェント
  - 個人開発
private: false
---

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260620_write-article-skill-recommendation_hero.png)

## note だけに投稿していたことに気づいた

Claude Code に `/write-article` というカスタムスキルを作っていて、記事の下書きをAIに書かせている。

しばらく使っていて気づいたのが、出力先がずっと note だったこと。ハウツー記事も、デバッグ記録も、ツール紹介も、全部 note に流れていた。

Qiita や Zenn を使っているエンジニアが、note をメインに読むことはあまりない。技術的な内容を書くなら、届けたい人が集まっている場所に出すべきだった。単純なことなのに、スキルが note に固定されていたせいで気づくのが遅れた。

## Qiita と Zenn にも出すことにした

プラットフォームによって届きやすい読者が違う。整理するとこうなる。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260620_write-article-skill-recommendation_fig1.png)

それぞれ向いている題材がある。Qiita に書いてもいいものを note に出し続けていたのは、動線がおかしかった。

## スキルを拡張した

Claude Code のカスタムスキルは、`~/.claude/skills/<スキル名>/SKILL.md` というファイルに自然言語の指示を書くだけで定義できます。コードは一切書きません。条件分岐も、出力先の切り替えも、フォーマットの指定も、すべて Markdown の日本語で書く構造です。

`/write-article` もこの仕組みで動いていて、実体は約700行の Markdown ファイルです。「note 書いて」「Qiita で書いて」といった自然な日本語を受け取り、どのモードで何を出力するかを自律で判断します。

### asset-skills リポジトリについて

スキルのリポジトリはここに公開しています。

https://github.com/ishizakahiroshi/asset-skills

`asset-skills` は複数のカスタムスキルをまとめたリポジトリです。現在こんな構成になっています。

```
asset-skills/
├── SKILLS.md                  ← スキル索引（一覧・呼び出し方）
└── skills/
    ├── write-article/
    │   └── skill.md           ← 記事執筆スキル（今回の話題）
    ├── release/
    │   └── SKILL.md           ← タグ駆動リリース管理
    ├── github-actions/
    │   └── SKILL.md           ← GitHub Actions ワークフロー整備
    ├── npm-publish/
    │   └── SKILL.md           ← npm パッケージ公開
    ├── make-icon/
    │   └── SKILL.md           ← アイコン・favicon 生成
    ├── review-sheet/
    │   └── SKILL.md           ← 協議資料 HTML 生成
    ├── repo-consistency/
    │   └── SKILL.md           ← README・ライセンス整合チェック
    ├── slash-commands-update/
    │   └── SKILL.md           ← slash コマンド鮮度管理
    └── portfolio-works/
        └── SKILL.md           ← ポートフォリオ管理
```

`SKILLS.md` がカタログになっていて、どのスキルをどういう言葉で呼ぶか・どこに出力されるかが一覧できます。スキルを増やしたいときもここにフォルダを足すだけです。

### 自分の環境への導入方法

clone してジャンクション（Windows のシンボリックリンク相当）を1本張るだけです。

```powershell
git clone https://github.com/ishizakahiroshi/asset-skills C:\dev\workshop
```

clone 先を `~/.claude/skills` として参照させます。

```powershell
New-Item -ItemType Junction `
  -Path "$env:USERPROFILE\.claude\skills" `
  -Target "C:\dev\workshop\skills"
```

Claude Code を再起動すれば、全プロジェクトで `/write-article`・`/release`・`/make-icon` などが使えるようになります。

この仕組みのポイントは、実体が `C:\dev\workshop\skills\` にある点です。`~/.claude/skills` はそこへのジャンクションなので、git でスキルを更新すれば即座に全プロジェクトへ反映されます。スキル側のファイルを触る必要はありません。

### 今回追加した Qiita・Zenn モードの中身

既存の `skill.md` に節を追加しました。Qiita モードは3つの節で構成しています。

**① 出力先とファイル命名**

```markdown
### 出力先・ファイル命名
- 出力先: `docs/qiita/<プロジェクト>/<YYYY-MM-DD>_<和文トピック>/<YYYYMMDD-slug>.md`
- slug 命名: `YYYYMMDD-topic-in-english`
```

note と同じ構成にしました。`docs/qiita/<プロジェクト>/<YYYY-MM-DD>_<和文トピック>/` の形でフォルダを切り、md と画像を同居させます。

**② frontmatter テンプレート**

Qiita は frontmatter でタグを管理します。毎回手で書かなくていいようにテンプレートを指示に入れています。

```yaml
---
title: ""
tags:
  - name: ""
private: false
---
```

生成された md にはこれが埋まった状態で出てくるので、タグを確認して投稿するだけです。

**③ トーン調整ルール**

note と Qiita では最適な書き方が違います。note は語り口を大事にした体験談が合う。Qiita は「どうやればいいか」を早く伝えることが大事です。

そのため Qiita モードにはこんな指示を入れました。

```
・手順・コマンドを冒頭に前出し（読者はまず答えを知りたい）
・タイトルはハウツー型（「〜した話」→「〜するときにやること」）
・太字・コードブロックを積極活用
・ツール紹介系ではフォルダ構成・GitHub URL・コマンド例を必ず含める
```

Zenn も同じ構造で、出力先 `docs/zenn/<プロジェクト>/<YYYY-MM-DD>_<和文トピック>/`・`emoji`・`type`・`topics` の frontmatter を指定しています。

### Qiita CLI と GitHub 連携で自動投稿している

Qiita CLI（`qiita pull` / `qiita publish` コマンド）は、ローカルの md をそのまま投稿・同期できる公式 CLI ツールです。GitHub との連携機能もあって、専用リポジトリ（`qiita-content`）に push するだけで自動投稿できます。

実際に導入しました。スキルが `docs/qiita/<プロジェクト>/<YYYY-MM-DD>_<和文トピック>/` に md を生成したあと、`public/` にコピーして git push する。それだけで GitHub Actions が走って Qiita に自動投稿されます。

```yaml
# .github/workflows/publish.yml（抜粋）
on:
  push:
    branches:
      - main
steps:
  - uses: increments/qiita-cli/actions/publish@v1
    with:
      qiita-token: ${{ secrets.QIITA_TOKEN }}
```

リポジトリに `QIITA_TOKEN` シークレットを登録しておくだけで動きます。`qiita pull` で既存記事を取り込んで同期する使い方もできます。

### 投稿ワークフロー

実際の流れはこうなっています。

```
/write-article qiita で書いて
     ↓ 生成
docs/qiita/<プロジェクト>/<YYYY-MM-DD>_<和文トピック>/YYYYMMDD-slug.md
     ↓ public/ にコピー
git push → GitHub Actions 起動
     ↓ 自動投稿
Qiita に記事が公開される
```

「コマンド1本で下書きが完成して、push するだけで投稿される」という状態です。

プラットフォームを指定しない場合は推薦が出ます（次の節）。

## おまけ：内容を読んでプラットフォームを推薦するようにした

Qiita・Zenn・note の3択が増えたことで、毎回どこに出すか考えるのが少し面倒になった。

「AIが題材を読んで推薦してくれれば、確認だけで済む」と思い、プラットフォームを指定しない場合は1問だけ確認するようにした。

```
Qiita がおすすめです。設定手順・ハウツー系なので検索で拾われやすく、エンジニアに届きます。
これでいいですか？ (Y/N)
```

Y で進む。N なら3択が出る。最悪でも2手で終わる。

推薦基準は `skill.md` に書いた。ハウツーなら Qiita、体験談なら note、設計判断なら Zenn、迷う場合は note（後から移植しやすいため）。

![](https://raw.githubusercontent.com/ishizakahiroshi/qiita-content/main/public/images/20260620_write-article-skill-recommendation_fig2.png)

## 使ってみて

出力先が変わっただけで、使い方は同じ。「この話は Qiita に出したほうがいい」と気づいたときに `qiita で書いて` と添えるだけ。

推薦の精度は思ったより高くて、ほとんど合っていた。外れたのは今のところ数回。

複数プラットフォームに書く習慣がなかったのは、出すのが面倒だったからでもある。スキルが対応してから、Qiita に出す記事が増えた。動線を直すだけで変わることは意外とある。

---

※ ヘッダー画像は AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
