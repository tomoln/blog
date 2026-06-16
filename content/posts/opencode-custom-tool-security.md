---
title: "opencodeのCustom Toolで個人情報を保護する"
slug: "opencode-custom-tool-security"
date: 2026-06-16
draft: true
categories: ["ツール活用"]
tags: ["opencode", "セキュリティ", "Custom Tool", "プライバシー", "LLM"]
---

## 背景

AIコーディングエージェント「opencode」を使っていると、APIキーや顧客データ、メールアドレスといった個人情報・機密情報を含むファイルを解析したい場面が出てきます。たとえばCSVのユーザーデータに重複がないか確認したい、JSONの設定ファイルに秘密情報が混入していないかチェックしたい、といったケースです。

しかしopencodeのチャットはopencode.aiのサーバーを経由し、最終的にはLLMプロバイダ（OpenAI、Anthropicなど）に送信されます。つまり、チャットの入力欄に直接ファイルの中身を貼り付けて「このデータを分析して」と頼むことは、機密情報を外部に送信することと同義です。

## 発生した問題

例えば以下のようなチャットを送信した場合、氏名やメールアドレスがLLMプロバイダのサーバーに送られてしまいます。

```text
以下のCSVデータを分析してください。
name,email,phone
田中太郎,tanaka@example.com,090-1234-5678
山田花子,yamada@example.com,080-9876-5432
```

これは開発の効率を大きく損ねます。データを扱いたいけどチャットには書けない。かといって手動分析では生産性が上がらない。このジレンマをどう解決するかが本記事のテーマです。

## 原因

opencodeのアーキテクチャを理解する必要があります。opencodeは75以上のLLMプロバイダをサポートしており、プロバイダによってデータ経路が異なります。特にopencodeがホストするモデル（例: opencode-go シリーズ）を使用する場合、チャットの内容はopencodeのサーバーを経由してLLMプロバイダに送信されます。直接APIキーを指定してプロバイダ（Anthropic、OpenAI等）に接続する場合でも、チャットの内容はLLMプロバイダのAPIに送信されることに変わりはありません。

つまりどのプロバイダを使う場合でも、チャットに入力したテキストは少なくともLLMプロバイダのサーバーで処理されます。企業のコンプライアンスポリシーによっては、この経路に顧客データを流すことが禁止されているケースも少なくありません。

## 解決策

opencodeには **Custom Tool** という機能が用意されています。`.opencode/tools/` ディレクトリにTypeScriptファイルを配置するだけで、LLMが呼び出し可能なツールとして自動登録されます。この仕組みを使えば、個人情報をチャットに書くことなく、ローカルで安全にデータ処理ができます。

### Custom Toolの動作原理

重要なのは、Custom Toolの実装コードがどのようにLLMに渡されるかです。従来のプロンプト注入方式とは違い、Custom Toolの定義はLLM APIの `tools` パラメータ（関数定義スキーマ）として送信されます。

つまりLLMには以下の情報しか見えません。

- ツール名
- 説明文
- 引数名とその型・説明

ツールの実装コード（TypeScriptの処理内容）は完全にローカルで実行され、**戻り値のみがLLMに返されます**。設定変更も不要で、`opencode.json` に何かを追記する必要は一切ありません。

### 実際のコード例

以下はJSON/CSVファイルを解析するCustom Toolの実装例です。重要なのは**生データを戻り値に含めない設計**になっている点です。

```typescript
// .opencode/tools/file-analyzer.ts
import { tool } from "@opencode-ai/plugin";
import * as fs from "fs";
import * as path from "path";

export default tool({
  description: "ファイルを解析し構造と統計のみを返す（生データ非含む）",
  args: {
    path: tool.schema.string().describe("解析するファイルのパス"),
  },
  async execute(args) {
    const absolutePath = path.resolve(args.path);
    const raw = fs.readFileSync(absolutePath, "utf8");
    const lines = raw.trim().split("\n");

    // 統計情報のみを抽出（生の値は含めない）
    const headers = lines[0].split(",");
    const stats = {
      行数: lines.length - 1,       // ヘッダー除く
      カラム数: headers.length,
      カラム名: headers,
      欠損: lines.slice(1).filter((line) => {
        return line.split(",").some((col) => col.trim() === "");
      }).length,
    };

    // サンプル表示時も値をマスク
    const mask = (val: string) =>
      val.length > 4 ? val.slice(0, 2) + "***" : val;

    const sample = lines.slice(1, 3).map((line) =>
      line.split(",").map(mask)
    );

    return JSON.stringify({ stats, sample }, null, 2);
  },
});
```

## 経路の違いを理解する

ここで最も重要なのは2つの異なるデータ経路を正しく理解することです。

**経路A（危険）: チャットに直接データを書く**
- ユーザーのテキスト → opencode.aiサーバー → LLMプロバイダ
- 個人情報が丸見えで外部に送信される ❌

**経路B（安全）: Custom Toolを使う**
- ユーザーがツールを呼び出し → ローカルopencodeプロセスで実行
- 戻り値（マスク済み）のみLLMに送信
- 生データはローカルに留まる ✅

Custom Toolがあっても、チャットに個人情報を書けば経路Aを通じて漏洩します。Custom Toolは「チャットに書かずにローカルの値を安全に使う」ための仕組みです。「LLMに値を読ませる」のではなく「LLMに値を使わせる」という設計思想が安全につながります。

## 学び・補足

### opencodeのその他のセキュリティ機構

opencodeにはCustom Tool以外にも個人情報を保護する仕組みが備わっています。

**Permission制御**: デフォルトで `.env` ファイルの読み取りは拒否されます（`"*.env": "deny"`）。ファイルの読み取りやコマンド実行も細かく制御可能です。

**Config変数**: `{env:VAR}` や `{file:path}` の構文でAPIキーを安全に設定できます。チャットにベタ書きする必要はありません。

**ローカルモデル**: OllamaなどのローカルLLMを使えば、すべてのデータがマシン内で完結します。外部へのデータ送信が一切発生しないため、最も厳格なセキュリティ要件にも対応できます。

### 安全なCustom Tool設計のポイント

1. **戻り値に生データを含めない**: 統計情報やスキーマ情報のみを返す設計にする
2. **マスク処理を徹底する**: どうしてもサンプル値を返す必要がある場合は、必ずマスクする
3. **不要な権限を与えない**: ツールには必要最小限のファイルアクセス権限のみ設定する
4. **ログにも出力しない**: ツール内のconsole.logなどにも個人情報を出力しない

この設計パターンを確立しておけば、opencode上で機密データを扱うあらゆる作業が安全に行えるようになります。Custom Toolは単なる便利機能ではなく、**セキュリティと生産性を両立するためのアーキテクチャ要素**です。チャットに直接データを貼り付ける時代は終わりました。安全なツール経由で、LLMの能力を最大限に活用しましょう。
