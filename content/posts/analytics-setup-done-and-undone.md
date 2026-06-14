---
title: "ブログにGA4とSearch Consoleを導入する"
slug: "analytics-setup-done-and-undone"
date: 2026-06-14T16:30:00+09:00
lastmod: 2026-06-14T16:30:00+09:00
draft: false
categories: ["tech"]
tags: ["GA4", "Google Analytics", "Search Console", "アクセス解析", "Hugo", "PaperMod", "Known Agents"]
description: "前回の記事で「アクセス解析未導入」と診断された反省を胸に、GA4とSearch Consoleを実際に導入した記録。できたこと・できなかったことを正直にレポートする。"
tldr: "Hugo + PaperMod のブログに GA4 と Search Console を導入した手順を解説。analytics.html は既にテンプレートにあったが extend_head.html が未作成だったため動いていなかった。Known Agents は GitHub Pages の静的サイト構成では導入できず断念。Measurement ID の探し方が地味にハマった。"
---

## この記事でわかること

- Hugo + PaperMod に GA4 を導入する具体的な手順
- Search Console を GA4 連携で最小工数でセットアップする方法
- Known Agents（AIクローラー分析ツール）が静的サイトでは使えない理由
- Google Analytics の Measurement ID の正しい探し方（初心者向け）

## はじめに

前回の記事「自分のブログを3段階で分析する」で、自分のブログを「アクセス解析」「SEO」「技術的品質」の3軸で診断しました。その結果、一番重い課題として挙がったのが**「アクセス解析未導入」**でした。

> 「アクセスが来ているのか、どこから来ているのか、まったく見えていない。まずはGA4を入れよう」

——そう締めくくった手前、「よし、やるか」と重い腰を上げました。本記事はその**実践レポート**です。上手くいったこと、いかなかったことをそのまま書きます。

## できたこと：GA4 + Search Console の導入

### GA4 のセットアップ

結論から言うと、**テンプレート側は既に準備済みだった**ので、やることはほぼ設定だけでした。

このブログは Hugo + PaperMod で運用しています。プロジェクト内のテンプレートを見てみると、既に `analytics.html` が用意されていました。

```html
{{- if .Site.Params.monetization.enabled -}}
  {{- $provider := .Site.Params.monetization.analytics_provider | default "ga4" -}}
  {{- $analyticsID := .Site.Params.monetization.analytics -}}
  {{- if eq $provider "ga4" -}}
    {{- with $analyticsID -}}
    <script async src="https://www.googletagmanager.com/gtag/js?id={{ . }}"></script>
    <script>
      window.dataLayer = window.dataLayer || [];
      function gtag(){dataLayer.push(arguments);}
      gtag('js', new Date());
      gtag('config', '{{ . }}', {
        anonymize_ip: true,
        cookie_flags: 'SameSite=None;Secure'
      });
    </script>
    {{- end -}}
  {{- end -}}
{{- end -}}
```

GA4 と Plausible の両方に対応したスクリプトです。ただし、このファイルは `layouts/partials/head/` に置かれているものの、**PaperMod に読み込ませる仕組みがありませんでした。**

PaperMod でカスタムスクリプトを `<head>` に追加するには、`layouts/partials/extend_head.html` を作成する必要があります。このファイルがそもそも存在しなかったので、新規作成して `analytics.html` を include するようにしました。

```
layouts/
  partials/
    extend_head.html    ← 新規作成
    head/
      analytics.html    ← 既存（設定待ち）
```

`extend_head.html` の中身は以下の2行です。

```html
{{- partial "head/analytics.html" . -}}
{{- partial "head/known-agents.html" . -}}
```

（`known-agents.html` は後で使うかもしれないのでプレースホルダーとして置いています。設定が有効でなければ何も出力しません。）

次に `params.yaml` に設定を追加します。

```yaml
monetization:
  enabled: true
  analytics_provider: ga4
  analytics: "G-XXXXXXXXXX"
```

この状態で `hugo` ビルド → 出力された HTML を確認すると、`gtag` スクリプトが正しく挿入されていました。あとはデプロイするだけです。

### Search Console のセットアップ

Search Console は GA4 と違って、**HTMLファイルをサーバーに置かなくても認証できました。**

GA4 が既に導入されている場合、Search Console の「プロパティタイプ」選択で「Google Analytics」を選ぶと、GA4 のアカウント情報を元に自動で認証が通ります。GA4 のタグが挿入されたページが確認できれば、それだけで所有権が証明される仕組みです。

登録した直後からインデックス状況が確認でき、**自分のブログは20ページが既にインデックス済み**でした。

## できなかったこと：Known Agents（AIクローラー分析）の導入断念

GA4 が入ったので、ついでに **Known Agents** という AI クローラー分析サービスも試してみようとしました。

Known Agents は、**GPTBot や ClaudeBot、Google-Extended など、AI 学習用のクローラーが自分のサイトにどのくらい来ているか**を可視化してくれるサービスです。

アカウントは作成しました。しかし、ここで壁に当たります。

Known Agents の接続方式は以下の3つが前提でした。

| 方式 | 対応環境 |
|------|---------|
| Cloudflare Workers | Cloudflare ユーザー向け |
| WordPress プラグイン | WordPress サイト向け |
| サーバーサイド SDK | サーバーサイドレンダリングが必要 |

**GitHub Pages のような純粋な静的サイトには、どの方式も対応していません。**

クライアントサイド（ブラウザ）で動く JS タグを埋め込む方式ではないため、静的ホスティングの構成ではそもそもデータを送信できないのです。Known Agents はサーバーのアクセスログを解析するアーキテクチャのため、ログを生成できない静的サイトは対象外でした。

導入するには、前段に Cloudflare を配置して Workers 経由にするか、あるいは Netlify や Vercel のサーバーサイド機能を使う必要があります。今回はそこまでの構成変更をする気はなかったので、**断念しました。**

## ハマったこと：Measurement ID の場所がわからない問題

GA4 の導入で一番時間がかかったのは実はこれです。

Google Analytics の管理画面を開いても、**Measurement ID（測定 ID）がどこに書いてあるのかパッと見つからない。**

最初は **Playwright でブラウザ操作を自動化**しようとしました。CLI から ID を取得できれば効率的だと思ったからです。しかし、Google のログインでは 2FA（二段階認証）が必要で、自動化スクリプトからログインセッションを引き継ぐのが難しい。ヘッドレスブラウザで毎回ログインするのは現実的ではありません。

結局、**最も確実な方法**に落ち着きました。

1. Google Analytics の管理画面をブラウザで開く
2. 「管理」→「データストリーム」→該当のストリームを選択
3. 「測定 ID（G-XXXXXXXXXX）」をコピー

もしくは、GA4 のトラッキングコード（タグスニペット）を表示させて、その中に書かれている ID をそのまま使う方法もあります。

**教訓：自動化にこだわらず、手動で確認 → 設定するのが確実。** 一度設定してしまえば二度と触る必要のない部分なので、凝りすぎないのが正解です。

## 現状の分析環境まとめ

今回の導入で、このブログの分析環境は以下のようになりました。

| ツール | 状態 | 見えるもの |
|-------|:----:|----------|
| GA4 | ✅ 導入済み | 訪問数・ページビュー・参照元・デバイス・ユーザー属性 |
| Search Console | ✅ 導入済み | 検索クエリ・表示回数・クリック数・インデックス状況（20ページ） |
| Known Agents | ❌ 導入できず | GitHub Pages の静的サイトには非対応。Cloudflare 前段化が必要 |

## まとめ

今回の実践で得た教訓をまとめます。

**できたこと**
- Hugo + PaperMod では `extend_head.html` を作成すれば任意のスクリプトを挿入できる
- GA4 の設定は Measurement ID さえわかれば一瞬で終わる
- Search Console は GA4 経由なら HTML ファイル不要で認証できる

**できなかったこと**
- Known Agents は静的サイトには非対応。サーバーサイドのアクセスログが必要
- 導入するなら前段に Cloudflare を置く構成変更が必要

**ハマったこと**
- Measurement ID の位置は「管理 → データストリーム → 該当ストリーム」の3階層下
- タグスニペットから直接抽出するのが最も手軽で間違いがない

前回の記事で「分析できるようにしたい」と書いてから、実際に手を動かして形にしました。GA4 と Search Console を導入しただけですが、ブログ運営において「どの記事が読まれているか」「どこから来た人が読んでいるか」が可視化されるのは大きな前進です。

次は、このデータをどう活用するか。例えば、よく読まれている記事を改善するとか、流入の多い経路に合わせたコンテンツを書くとか、分析結果をブログ運営にフィードバックするフェーズに入ります。またそのうちレポート記事を書こうと思います。
