---
title: "MCPブラウザ自動化サーバーを比較する"
slug: "mcp-browser-automation-comparison-2026"
date: 2026-06-22
draft: false
categories: ["技術調査"]
tags: ["MCP", "Playwright", "Chrome DevTools", "Puppeteer", "ブラウザ自動化", "AIエージェント", "Model Context Protocol"]
description: "【2026年版】AIエージェントのブラウザ操作精度を最大化するMCPサーバー比較。Playwright MCP（Microsoft公式、34.2k⭐）とChrome DevTools MCP（44.1k⭐）の精度・機能・マルチブラウザ対応を徹底比較し、ユースケース別の選び方を解説する。"
tldr: "MCPでブラウザを操作するサーバーは、Playwright MCP（Microsoft公式）とChrome DevTools MCP（44.1k⭐）の2強。要素特定精度は互角だが、デバッグ・パフォーマンス分析・既存ブラウザアタッチが必要ならChrome DevTools MCP、FirefoxやWebKit対応が必要ならPlaywright MCPを選ぶ。コミュニティ製のExecuteAutomationは存在意義が薄れている。"
wordCount: 0
timeRequired: "PT10M"
affiliate: false
llm_summary: "MCP（Model Context Protocol）でブラウザを自動操作するサーバーを比較した記事。Microsoft公式のPlaywright MCP（34.2k⭐）とChrome DevToolsチームのChrome DevTools MCP（44.1k⭐）を中心に、精度の仕組み、マルチブラウザ対応、デバッグ機能、既存セッション利用の可否を比較。Chrome限定で良いならDevTools MCP、クロスブラウザが必要ならPlaywright MCPを推奨。"
---

## この記事でわかること

- MCP経由でブラウザを操作するサーバーの現状（2026年6月時点）
- Microsoft公式 **Playwright MCP**（34.2k⭐）と **Chrome DevTools MCP**（44.1k⭐）の精度の違い
- それぞれの得意・不得意と、ユースケース別の選び方
- 実際のMCP設定ファイルの書き方

## 背景

AIエージェントに「Webページを開いて情報を取得する」「フォームに入力する」「スクリーンショットを撮る」といったブラウザ操作を任せたい場面が増えている。opencode や Claude Code などのAIコーディングエージェントは **MCP（Model Context Protocol）** を通じてブラウザを操作できるが、そのバックエンドにはいくつかの選択肢がある。

2026年6月現在、実質的な選択肢は**3系統**。どれを選ぶかで、AIエージェントのブラウザ操作精度が大きく変わる。本記事では各サーバーを技術的に比較し、目的別の選び方を整理する。

## TL;DR（結論）

<!-- AIクローラーが引用する要約 -->

1. **Chrome DevTools MCP（44.1k⭐）が総合力でリード** — ツール数40以上、CDP直操作による高精度、既存ブラウザへのアタッチ対応、パフォーマンス/メモリデバッグまでカバー。ただしChrome専用。
2. **Playwright MCP（Microsoft公式、34.2k⭐）はクロスブラウザが必要な場合の第一候補** — Chromium/Firefox/WebKit対応。Accessibility Treeベースの要素特定は正確だが、視覚的操作やデバッグの深さでは劣る。
3. **Chromeだけで良く、かつ高精度な操作とデバッグが欲しいなら Chrome DevTools MCP 一択**。Firefox/WebKitでのテストが必要なら Playwright MCP を選ぶ。

## 詳細

### 現状の3択

2026年6月時点で、MCP経由でブラウザを操作するサーバーは実質3系統に収束している。

| # | サーバー | Stars | エンジン | メンテナー | 状態 |
|---|---------|-------|---------|-----------|------|
| 1 | **@playwright/mcp** | 34.2k⭐ | Playwright | **Microsoft公式** | ✅ 現行・活発 |
| 2 | **chrome-devtools-mcp** | **44.1k⭐** | Puppeteer + CDP | **Chrome DevTools公式** | ✅ 現行・活発 |
| 3 | @executeautomation/playwright-mcp-server | 5.6k⭐ | Playwright | コミュニティ | ✅ 現行（存在意義薄） |
| — | @modelcontextprotocol/server-puppeteer | — | Puppeteer | 旧MCP公式 | ❌ アーカイブ済 |

---

### Playwright MCP（@playwright/mcp）

Microsoft公式が提供するPlaywrightベースのMCPサーバー。**Accessibility Tree（アクセシビリティツリー）** の構造データを利用して要素を特定するのが最大の特徴だ。

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": [
        "-y",
        "@playwright/mcp@latest",
        "--caps",
        "vision"
      ]
    }
  }
}
```

**精度の仕組み**: ピクセル座標ではなく、DOMのアクセシビリティ構造を参照して操作する。そのため「画面上のこの辺り」ではなく「ログインボタン」という論理的な指定が可能。

**vision機能**: `--caps vision` フラグで実験的にスクリーンショットのピクセル座標ベース操作も可能。

**ツール数**: 約25種類（クリック、ナビゲート、フォーム入力、スクリーンショット、コンソールログ取得、ネットワークリクエスト一覧など）。

**マルチブラウザ**: ✅ Chromium / Firefox / WebKit すべて対応。これはChrome DevTools MCPに対する最大のアドバンテージ。

**精度の限界**: Accessibility Treeに依存するため、以下が苦手：
- 画像の視覚的な位置確認
- デザインのレイアウト検証
- 複雑なCSSアニメーションの状態取得

READMEでも「CLI + SKILLSの方がトークン効率が良い」と自ら認めており、あくまで「MCPインタフェースとして使える」という位置づけ。

---

### Chrome DevTools MCP（chrome-devtools-mcp）

Chrome DevToolsチーム公式（44.1k⭐）。**Puppeteer + Chrome DevTools Protocol（CDP）** を直操作する。Star数でPlaywright MCPを10k上回っており、コミュニティの支持も厚い。

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": [
        "-y",
        "chrome-devtools-mcp@latest",
        "--experimentalVision"
      ]
    }
  }
}
```

**なぜ精度が高いのか**:

1. **CDP直操作** — Playwrightのような抽象化レイヤーを挟まないため、DevToolsが持つすべての機能に直接アクセスできる。レスポンスが正確で、操作の一段階深い情報まで取得可能。
2. **ツール数40以上** — Playwright MCPの約25種類を大きく上回る。単なるクリックやナビゲーションだけでなく、パフォーマンストレース、Lighthouse監査、ヒープスナップショット、メモリデバッグ、ネットワーク分析までカバー。
3. **既存ブラウザにアタッチ可能** — 実行中のChromeにそのまま接続できるため、ログイン状態・Cookie・LocalStorageを維持したまま操作可能。これにより「ログイン済みの状態でスクレイピング」が極めて簡単。
4. **同時セッション対応** — `--experimentalPageIdRouting` で複数エージェントの並列動作が可能。
5. **スリムモード** — `--slim` フラグで基本3ツールに絞って軽量運用もできる。

**デメリット**: Chrome / Chromiumのみ。FirefoxやWebKitでは動作しない。

---

### ExecuteAutomation Playwright MCP（5.6k⭐）

コミュニティ製。Microsoft公式のPlaywright MCPが出た現在、存在意義はかなり薄れている。

強いて挙げるなら**143のデバイスエミュレーションプリセット**が内蔵されている点だが、これは精度向上に直結するものではない。新規に導入する理由はほぼない。

---

### 精度比較サマリ

| 観点 | Playwright MCP | Chrome DevTools MCP |
|------|---------------|-------------------|
| 要素特定精度 | ⭐⭐⭐⭐⭐ Accessibility Tree | ⭐⭐⭐⭐⭐ DOM + CDP |
| 視覚的操作精度 | ⭐⭐（--caps vision は実験的） | ⭐⭐⭐⭐（--experimentalVision は実験的） |
| デバッグ精度 | ⭐⭐⭐ コンソール+ネットワークのみ | ⭐⭐⭐⭐⭐ パフォーマンス/メモリ/Lighthouse |
| マルチブラウザ | ⭐⭐⭐⭐⭐ Chrome/Firefox/WebKit | ⭐⭐ Chromeのみ |
| エラーハンドリング | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐  |
| 既存セッション利用 | ⭐⭐⭐（ログイン再現は手間） | ⭐⭐⭐⭐⭐（ブラウザに直接アタッチ） |
| ツールの豊富さ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 総合Star数 | 34.2k⭐ | 44.1k⭐ |

### 使うときの設定例

opencode で MCP サーバーを利用する場合の設定を紹介する。opencode の設定は `"mcpServers"` ではなく `"mcp"` キーを使う点に注意。

**~/.config/opencode/opencode.json** に以下のように記述する（両方を同時に登録も可能）：

```json
{
  "mcp": {
    "playwright": {
      "type": "local",
      "command": ["npx", "@playwright/mcp@latest"],
      "enabled": true
    },
    "chrome-devtools": {
      "type": "local",
      "command": ["npx", "-y", "chrome-devtools-mcp@latest"]
    }
  }
}
```

既存のブラウザ（ログイン状態を維持したChrome）にアタッチする場合は、以下のように `--browser-url` を追加する：

```json
{
  "mcp": {
    "chrome-devtools": {
      "type": "local",
      "command": ["npx", "-y", "chrome-devtools-mcp@latest", "--browser-url=http://127.0.0.1:9222"]
    }
  }
}
```

両方登録しておくと、用途に応じてエージェントに「Playwright MCPを使ってFirefoxでテストして」「Chrome DevTools MCPでパフォーマンス分析して」と使い分けさせられる。

## まとめ

### 目的別おすすめ

| シナリオ | おすすめ | 理由 |
|---------|---------|------|
| Chromeだけで良い、高精度な操作とデバッグが必要 | **Chrome DevTools MCP** | ツール数・精度・既存セッション利用のすべてで優位 |
| Firefox/WebKitも含めたクロスブラウザテスト | **Playwright MCP** | 唯一のマルチブラウザ対応 |
| ログイン状態を維持したまま操作したい | **Chrome DevTools MCP** | 実行中のブラウザに直接アタッチ可能 |
| 新規にこれから始める | **Chrome DevTools MCP** | Star数・機能・コミュニティ活発度でリード。特にこだわりがなければこちら |
| コミュニティ製でいいものがあるなら | 選ばなくて良い | ExecuteAutomationはMicrosoft公式が出た今、存在意義が薄い |

### 最終判断

- **Chromeだけで完結するなら、迷わず Chrome DevTools MCP**
- **Firefox / WebKit の対応が絶対条件なら Playwright MCP**
- **両方登録して、エージェントに状況判断させるのが理想的**

精度比較の核心は「Accessibility Tree vs CDP直操作」の違いにある。抽象化の恩恵（マルチブラウザ）を取るか、生のプロトコル操作による深い制御（高機能・高精度）を取るか。このトレードオフを理解した上で選んでほしい。

---

<!--
GEO対策メモ:
- description は 120〜160 文字で検索意図を明確に ✅
- tldr は 2〜3 文で記事の結論を要約（AIクローラーが引用） ✅
- 見出し構成は H2 → H3 の階層を守る ✅
- コードブロックには言語指定を忘れずに ✅
- 内部リンクを積極的に入れる（関連記事）
-->
