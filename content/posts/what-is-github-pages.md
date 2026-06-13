---
title: "GitHub Pagesって結局なんなの？仕組み・セキュリティ・運用のリアル"
slug: "what-is-github-pages"
date: 2026-06-12T22:00:00+09:00
lastmod: 2026-06-12T22:00:00+09:00
draft: false
categories: ["tech"]
tags: ["GitHub Pages", "Hugo", "静的サイト", "セキュリティ", "ブログ"]
description: "GitHub Pagesの仕組みを「リポジトリとの関係」「セキュリティ」「ブログとしての実用性」の3軸で解説。実際にHugoブログを構築した経験を交えながら、静的サイトでも普通のブログサービスと同じように運用できることを示す。"
tldr: "GitHub Pagesはリポジトリのコードを自動で静的サイトとして公開する仕組み。DBも管理画面もないが、その分セキュリティリスクが極小。Hugoと組み合わせれば、トップページ・個別記事URL・カテゴリ分類・タグ検索など、普通のブログサービスと同等の機能を実現できる。"
---

## この記事でわかること

- GitHub Pagesとは何か — 「リポジトリをブログとして公開する」という核心
- 実際の構築手順（Hugo + GitHub Actions + gh-pages）
- リポジトリのPublic公開とセキュリティのバランス
- 「静的サイトでブログって成立するの？」のリアルな答え

## 背景

「ブログを始めたい。でもWordPressは重いし管理が面倒。はてなブログやnoteは自由が効かない。できれば無料で、技術ブログらしい内容を自分好みにカスタマイズしたい。」

こう考えたとき、最初に候補に上がるのが**GitHub Pages**です。実際に私も、AIエージェント開発パイプライン「Agent Arc」のブログ基盤として、Hugo + GitHub Pages を採用しました。

ですが、GitHub Pagesに対してはこんな声もよく聞きます。

- 「GitHubってプログラマー向けのツールでしょ？ブログ書くのに使うの？」
- 「リポジトリがPublicになるってことはコードが丸見え？セキュリティ大丈夫？」
- 「静的ページってことは、ただのHTMLが置いてあるだけ？ブログとして成り立つの？」

本記事では、**実際にHugoブログをGitHub Pagesで構築した経験**をベースに、これらの疑問に一つずつ答えていきます。

## GitHub Pagesの構築手順（今回やった具体例）

まずは実際の手順を追ってみましょう。Hugoでブログを作り、GitHub Pagesで公開するまでの流れです。

### 1. GitHubでリポジトリを作成

通常のリポジトリ作成と同じです。ブログのソースコードを置くリポジトリを一つ作ります。

```bash
# ローカルでHugoサイトを初期化
hugo new site blog --format yaml
cd blog

# Gitリポジトリとして初期化
git init
git add .
git commit -m "Initial commit"

# GitHubで作成したリポジトリと接続
git remote add origin git@github.com:username/blog.git
git branch -M main
git push -u origin main
```

ここで重要なのは、**リポジトリ名に特にルールはない**という点です。`blog` でも `my-site` でも構いません。ただし、後述するUser siteとProject siteの違いでURLが変わります。

### 2. Hugoでテーマを導入・ビルド確認

Hugoのテーマ（今回はPaperMod）を導入します。

```yaml
# hugo.yaml
baseURL: "https://username.github.io/blog/"
languageCode: ja-JP
title: "My Tech Blog"
theme: PaperMod

permalinks:
  posts: "/posts/:slug/"

params:
  description: "技術ブログです"
  ShowReadingTime: true
  ShowShareButtons: true
```

```bash
# テーマをサブモジュールとして追加
git submodule add https://github.com/adityatelange/hugo-PaperMod themes/PaperMod

# ローカルプレビュー
hugo server -D
```

`-D` フラグは下書き（draft）の記事も表示するオプションです。記事を書きながらリアルタイムでプレビューできます。

### 3. GitHub Actionsのワークフロー設定

GitHub Pagesへの自動デプロイには、GitHub Actionsを使います。`.github/workflows/gh-pages.yml` を作成します。

```yaml
name: Deploy Hugo site to GitHub Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "latest"
      - name: Build
        run: hugo --minify
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public
  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

ここでのポイントは2つ。

- `actions/checkout@v4` に `submodules: recursive` を指定して、PaperModテーマ（サブモジュール）も正しく取得する
- `actions/upload-pages-artifact@v3` で `./public`（Hugoのビルド出力先）をアップロードする

### 4. GitHub Pagesの有効化

GitHubのリポジトリ設定画面（Settings → Pages）で、**「GitHub Actions」をソースに指定**します。

または、GitHub APIを使ってコマンドラインからも設定可能です。

```bash
# GitHub CLIでPagesを有効化
gh api repos/:owner/:repo/pages \
  --input - <<< '{"build_type": "workflow"}'
```

### 5. 初回デプロイの注意点（最初は404）

ここで一つ、**必ずと言っていいほど遭遇する壁**があります。

初回のデプロイが完了しても、`https://username.github.io/blog/` にアクセスすると**404 Not Found**が表示されることです。

理由はシンプルで、GitHub Pagesがデプロイを検知してから実際にルーティングが反映されるまでに**数分のタイムラグ**があるからです。

私の場合も、GitHub ActionsのワークフローはSuccessになったのに、ブラウザで開くと404。焦ってActionsのログを何度も見直しました。結果的には、もう一度ワークフローを手動で再実行（`workflow_dispatch`）したら無事に表示されました。

**教訓: GitHub Pagesはデプロイ直後に一呼吸置くか、再実行が確実。**

## GitHub Pagesはリポジトリ上ではどういう扱い？

ここが一番重要で、かつ一番誤解されているポイントです。

### 「リポジトリをブログとして公開する」が正しい理解

GitHub Pagesを一言で表すなら、**「Gitリポジトリの内容をWebサイトとして公開する仕組み」**です。

具体的な流れはこうなります。

```
あなたのPC（ローカル）
  ↓ git push
GitHubリポジトリ（mainブランチ）
  ↓ GitHub Actionsが検知
Hugoビルド（public/ を生成）
  ↓ gh-pagesブランチ or actions-artifact
GitHub PagesのCDNがホスティング
  ↓
全世界に公開 🎉
```

つまり、リポジトリの中身と公開されるWebサイトの関係はこんな感じです。

| リポジトリ側 | 公開サイト側 |
|------------|------------|
| `content/posts/hello.md`（原稿） | `/posts/hello/`（HTML記事） |
| `hugo.yaml`（設定） | サイト全体の設定として反映 |
| `themes/PaperMod/`（テーマ） | デザインとして反映 |
| `archetypes/posts.md`（テンプレート） | 新規記事作成時のひな形 |
| `.github/workflows/gh-pages.yml` | デプロイパイプライン |

**リポジトリのコード＝ブログの「原稿」であり「ビルド設定」であり「公開コンテンツの管理基盤」**なのです。

### Project site vs User site

GitHub Pagesには2種類のサイトがあります。

| 種類 | リポジトリ名 | 公開URL |
|------|------------|---------|
| **User site** | `username/username.github.io` | `https://username.github.io/` |
| **Project site** | `username/blog` など任意 | `https://username.github.io/blog/` |

User siteは1アカウントに1つだけ。リポジトリ名がそのままドメインになります。Project siteは無制限に作れて、サブディレクトリで運用されます。

ブログ用途ならProject siteで十分です。User siteはポートフォリオサイトなどトップレベルドメインが欲しい場合に向いています。

## セキュリティ的にはどうなの？

ここがGitHub Pages最大の魅力であり、同時に注意すべきポイントです。

### 良い点：攻撃対象が圧倒的に少ない

**静的サイトなので、DBがない、サーバーサイドの処理がない＝攻撃対象が極小。**

WordPressやはてなブログと比較してみましょう。

| 観点 | WordPress / はてなブログ | GitHub Pages |
|------|------------------------|-------------|
| DB | MySQL必要 | **なし** |
| サーバーサイド | PHPランタイム | **なし** |
| 管理画面 | `/wp-admin/` が攻撃対象 | **そもそも存在しない** |
| ログイン | パスワード認証必要 | **不要（Git pushで公開）** |
| コメント | SPAM対策必要 | **なし（外部サービスで代替）** |
| HTTPS | 自分で設定 or 有料 | **Let's Encrypt自動・強制** |
| DDoS耐性 | サーバー次第 | **GitHubインフラ（世界最大級）** |

特に「管理画面がない」というのはセキュリティ的に大きなアドバンテージです。ログインフォームが存在しないので、総当たり攻撃もセッションハイジャックもありえません。

HTTPSも完全自動で、Let's Encryptの証明書をGitHub側が管理してくれます。カスタムドメインを使っても同様です。

### 注意点：リポジトリがPublicだとソースコードは全世界に見える

ここがGitHub Pagesを使う上で**唯一と言っていいほどの大きな注意点**です。

無料プランでは、GitHub Pagesを有効にしたリポジトリは基本的に**Public**になります（Privateでも有料プランなら可能）。

つまり、以下のものが全世界から見えることを理解しておく必要があります。

- **全記事の原稿（Markdown）** — 公開前の下書きも含む
- **コミット履歴** — 過去に削除した記事の痕跡も残る
- **シークレットやAPIキーを間違ってコミットすると即座に流出**

対策はシンプルです。

1. **`.gitignore` を必ず設定する**（`.env`、シークレット情報、ローカルキャッシュ等）
2. **Hugoの `draft: true` を正しく管理する** — 下書きはビルドで除外される
3. **`.env` やAPIキーはリポジトリに置かない**（GitHub ActionsのSecretsを使う）

```gitignore
# .gitignore の例
public/
.hugo_build.lock
.env
*.local
node_modules/
```

また、**コミット履歴に一度でもシークレットを載せてしまうと、後から削除しても履歴からは完全には消えません**。`git filter-branch` や `BFG Repo-Cleaner` で履歴ごと消す必要が出てきます。事故る前に `.gitignore` で予防するのが最善です。

### カスタムドメインのDNS設定ミスに注意

独自ドメインを使う場合（例：`blog.example.com`）、DNSの設定ミスがセキュリティリスクになりえます。

- **CNAMEレコードの設定ミス**で、サブドメインハイジャックのリスク
- **GitHub PagesのIPアドレスが変わることがある**ので、Aレコード直指定よりCNAME推奨
- HTTPS証明書の発行に数分〜数十分かかる場合がある

GitHub PagesはカスタムドメインでもLet's Encryptの証明書を自動発行してくれますが、DNSの伝播待ちと証明書発行のタイミングで一時的にHTTPSが効かないことがあります。慌てずに待ちましょう。

## 静的ページなのにブログとして成立するの？

「静的サイト」と聞いて、HTMLファイルを直で置いただけの簡素なサイトを想像するかもしれません。しかし、Hugoのような静的サイトジェネレーターを使えば、**普通のブログサービスと遜色ない機能を実現できます**。

### トップページ — 作れます

PaperModテーマなら、トップページは自動生成されます。

![PaperModテーマのトップページ例](https://github.com/adityatelange/hugo-PaperMod/wiki/Screenshots)

デフォルトで表示されるもの：
- 最新記事の一覧（日付降順）
- Aboutページへのリンク
- カテゴリ・タグの一覧
- 検索バー（Fuse.jsベース）

トップページに何を表示するかは `layouts/index.html` や `hugo.yaml` の `params` で自由にカスタマイズできます。

### 記事の並び順 — 日付順に自動ソート

Hugoはデフォルトで記事を**日付降順**にソートします。新しい記事が上に来る、ごく自然なブログの挙動です。

```yaml
# hugo.yaml — 並び順のカスタマイズ
params:
  mainSections:
    - posts  # トップページに表示するセクション
```

カテゴリやタグでフィルタリングした一覧ページも自動生成されます。

```
/posts/                    # 全記事一覧
/categories/hugo/          # Hugoカテゴリの記事一覧
/tags/github-pages/        # GitHub Pagesタグの記事一覧
```

### 記事ごとのURL — 完全に自由

Hugoの `permalinks` 設定で、各記事のURLを自由に設計できます。

```yaml
# hugo.yaml
permalinks:
  posts: "/posts/:slug/"          # → /posts/what-is-github-pages/
  # 他のオプション例:
  # "/blog/:year/:month/:slug/"   # → /blog/2026/06/what-is-github-pages/
  # "/:slug/"                     # → /what-is-github-pages/
  # "/:year/:month/:day/:slug/"   # → /2026/06/12/what-is-github-pages/
```

**`:slug`** は各記事のfrontmatterで指定したslugが入ります。指定しない場合はタイトルから自動生成されます。

### 他のブログサービスとの違い（正直な比較）

| 機能 | はてなブログ / note | WordPress.com | GitHub Pages + Hugo |
|------|-------------------|---------------|-------------------|
| **記事作成** | ブラウザのエディタ | ブラウザのエディタ | **テキストエディタ or CLI** |
| **管理画面** | あり | あり | **なし（Git管理がすべて）** |
| **コメント** | 標準搭載 | 標準搭載 | **なし（SNSやUtterance等で代替）** |
| **検索機能** | 標準搭載 | 標準搭載 | **テーマ次第（PaperModは対応）** |
| **アクセス解析** | 標準搭載（簡易） | 標準搭載（簡易） | **自分で仕込む（GA4 / Plausible等）** |
| **デザイン** | テーマ選択のみ | テーマ＋CSS編集 | **完全自由（HTML/CSS/JS全部OK）** |
| **コスト** | 無料〜月500円 | 無料〜月1500円 | **完全無料（ドメイン代除く）** |
| **耐久性** | サービス終了リスクあり | サービス終了リスクあり | **Gitが残る限り消えない** |
| **記事の移植性** | エクスポート機能あり | エクスポート機能あり | **Markdownなので完全にポータブル** |

「管理画面がない」と聞くと不便に感じるかもしれませんが、慣れると**原稿をテキストファイルで管理する自由さ**がむしろ快適です。エディタで書き、`git push` するだけで公開完了。ブラウザを開く必要すらありません。

## まとめ

GitHub Pagesについて、3つの軸で整理しました。

1. **仕組み**: リポジトリのコードをGitHub Actionsがビルドし、静的ファイルとしてCDNから配信する。リポジトリ＝ブログの全て。
2. **セキュリティ**: DBも管理画面もない静的サイトは攻撃対象が極小。唯一の注意点はリポジトリがPublicになること。`.gitignore` と `draft` 管理で対策可能。
3. **ブログとしての実用性**: Hugo + PaperModで、トップページ・個別URL・カテゴリ・タグ・検索までカバーできる。コメントやアクセス解析は外部サービスで補完。

「GitHub Pagesってなんか難しそう」という最初の印象とは裏腹に、**一度仕組みを理解してしまえば、これほどシンプルでセキュアなブログ基盤はありません**。

実際に運用してみて感じるのは、管理画面がないことが「ブログを書く」という純粋な行為に集中できる環境を作ってくれるということ。エディタでMarkdownを書いて `git push` する。たったこれだけでブログが更新される感覚は、一度体験するとやめられません。

あなたもぜひ、今日からGitHub Pages + Hugoでブログを始めてみてください。

---

**関連リンク**

- [GitHub Pagesの公式ドキュメント](https://docs.github.com/ja/pages)
- [Hugo](https://gohugo.io/)
- [PaperModテーマ](https://github.com/adityatelange/hugo-PaperMod)
- [peaceiris/actions-gh-pages](https://github.com/peaceiris/actions-gh-pages)
