---
title: "クラウドで個人メモツールを作ってみる"
slug: "building-personal-memo-tool-on-cloud"
date: 2026-06-13T10:00:00+09:00
lastmod: 2026-06-13T10:00:00+09:00
draft: false
categories: ["技術記事"]
tags: ["AWS", "Discord", "CLI", "TypeScript", "個人開発"]
description: "NotionやObsidianに不満を感じ、AWSサーバーレス（API Gateway + Lambda + DynamoDB）で個人メモツール「mycow2」を構築した記録。日付パーサー、CLIとDiscord Botの両対応、ed25519署名検証のハマりポイントを解説。"
tldr: "Notion/Obsidianの重さや同期コストに耐えかねて、自分用のメモ＆スケジュールツールをAWSサーバーレスでゼロから構築。API Gateway + Lambda + DynamoDBのシンプル構成で月額0〜1USD。CLI（commander.js）とDiscord Botの両方から操作可能。ed25519署名検証で地獄を見た話も含む。"
---

## はじめに

メモアプリを転々としていました。

**Notion**: 起動が重い。CLIがないのでPCでの入力を全部ブラウザかアプリに依存する。スマホ編集も微妙で、ちょっとした一言を残すだけなのに待たされる。

**Obsidian**: 動作は軽快だが、公式の同期サービスは有料。ローカルファイル管理が面倒で、複数端末間の同期を自分で運用するのが億劫になる。

どちらも優れたツールですが、「メールするだけ」「Discordに投げるだけ」でメモが残るツールが欲しくなりました。スマホからはDiscord、PCからはCLIでサッとメモれて、かつスケジュール管理もできて、運用コストが月額数百円以下。既存のサービスでは要件にピッタリ合うものがなかったので、自分で作ることにしました。

## アーキテクチャ

### AWSサーバーレス構成

構成は至ってシンプルです。

```
┌──────────────┐     ┌──────────────────┐     ┌────────────────────┐
│  CLI (PC)    │────▶│  API Gateway     │────▶│  API Lambda        │
│  commander   │     │  HTTP API        │     │  (api-handler)     │
└──────────────┘     │  /{proxy+}       │     └─────────┬──────────┘
                      │  /discord        │               │
┌──────────────┐     │                  │     ┌────────────────────┐
│  Discord Bot │────▶│                  │────▶│  Discord Lambda   │
│  (Slash)     │     └──────────────────┘     │  (discord-handler) │
└──────────────┘                               └─────────┬──────────┘
                                                          │
                                                          ▼
                                                   ┌────────────────┐
                                                   │  DynamoDB      │
                                                   │  (ペイロード)  │
                                                   └────────────────┘
```

- **API Gateway**: HTTP API（RESTより安価、月額0〜1USD）
- **Lambda**: Node.js 22、アーム64（Graviton）でコスト最適化
- **DynamoDB**: オンデマンドキャパシティ、個人利用なら月1円未満
- **CDK (TypeScript)**: インフラはコード管理。`cdk deploy` 一発で環境構築

個人利用が前提なので、すべて無料枠に収まる設計です。API Gatewayはリクエスト100万件/月まで無料、Lambdaは100万リクエスト/月まで無料、DynamoDBは25GBまで無料。自分一人で使う分には実質0円です。

### なぜCDKか

HugoブログのCLIをTypeScriptで書いた経験から、同じ言語でインフラまで管理できるCDKの体験に軍配が上がりました。AWS CLI + CloudFormationのYAML泥沼より、`aws-lambda` や `aws-apigatewayv2` のコンストラクトをimportしてTypeScriptで書く方が圧倒的に生産性が高いです。

```typescript
// CDK スタックの一部
const table = new dynamodb.Table(this, "EntriesTable", {
  partitionKey: { name: "PK", type: dynamodb.AttributeType.STRING },
  sortKey: { name: "SK", type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
  timeToLiveAttribute: "ttl",
});

// GSI: タイプ別一覧
table.addGlobalSecondaryIndex({
  indexName: "TypeIdx",
  partitionKey: { name: "GSI1PK", type: dynamodb.AttributeType.STRING },
  sortKey: { name: "GSI1SK", type: dynamodb.AttributeType.STRING },
});

// GSI: 日付範囲検索
table.addGlobalSecondaryIndex({
  indexName: "DateIdx",
  partitionKey: { name: "GSI2PK", type: dynamodb.AttributeType.STRING },
  sortKey: { name: "GSI2SK", type: dynamodb.AttributeType.STRING },
});

// API用Lambda（api-handler）
const apiHandler = new lambdaNodejs.NodejsFunction(this, "ApiHandler", {
  entry: "src/lambda/api-handler.ts",
  handler: "handler",
  runtime: lambda.Runtime.NODEJS_22_X,
  architecture: lambda.Architecture.ARM_64,
  environment: { TABLE_NAME: table.tableName },
});

// Discord Bot用Lambda（discord-handler）
const discordHandler = new lambdaNodejs.NodejsFunction(this, "DiscordHandler", {
  entry: "src/lambda/discord-handler.ts",
  handler: "handler",
  runtime: lambda.Runtime.NODEJS_22_X,
  architecture: lambda.Architecture.ARM_64,
  environment: {
    TABLE_NAME: table.tableName,
    DISCORD_PUBLIC_KEY: process.env.DISCORD_PUBLIC_KEY!,
  },
});

const api = new apigatewayv2.HttpApi(this, "MemoApi", {
  corsPreflight: {
    allowOrigins: ["*"],
    allowMethods: [apigatewayv2.CorsHttpMethod.ANY],
  },
});

// API Lambda用ルート（プロキシ統合）
api.addRoutes({
  path: "/{proxy+}",
  methods: [apigatewayv2.HttpMethod.GET, apigatewayv2.HttpMethod.POST,
            apigatewayv2.HttpMethod.PUT, apigatewayv2.HttpMethod.DELETE],
  integration: new apigatewayv2.HttpLambdaIntegration("ApiIntegration", apiHandler),
});

// Discord Bot用エンドポイント
api.addRoutes({
  path: "/discord",
  methods: [apigatewayv2.HttpMethod.POST],
  integration: new apigatewayv2.HttpLambdaIntegration("DiscordIntegration", discordHandler),
});
```

## 日付パーサーの実装

### 自由形式テキストから日付を自動認識

このツールのキモは「自由形式のテキストから日付や時刻を自動認識する」ことです。以下のような入力をパースできるようにしました。

- `2026/03/22 薬局` → 日付指定メモ
- `2026/07/20 - 2026/07/25 旅行` → 日付範囲（スケジュール）
- `2026/03/22 18:00-21:00 待機` → 時刻指定
- `明日 歯医者 10:00` → 相対日付

日本語の「今日」「明日」「明後日」にも対応しています。

### 実装の流れ

1. テキストから正規表現で日付パターンを抽出
2. 抽出した文字列をパースして `Date` オブジェクトに変換
3. 時刻の範囲（`HH:MM-HH:MM`）があればそれもパース
4. 残りのテキストをメモ本文として抽出

```typescript
export function parseEntry(input: string): ParsedEntry {
  // 相対日付の解決（parseRelativeDate()を使用）
  const RELATIVE_DATE_REGEX = /^(今日|明日|明後日|あした|あさって)\s*/;

  // 時刻パターン: HH:MM
  const TIME_PATTERN = /\d{2}:\d{2}/;

  // 日付パターン: YYYY/MM/DD または YYYY-MM-DD
  const DATE_PATTERN = /\d{4}[-/]\d{2}[-/]\d{2}/;

  // 複合パターンを優先順位で試行:
  // 1. YYYY/MM/DD HH:MM - YYYY/MM/DD HH:MM  （日付範囲＋時刻）
  // 2. YYYY/MM/DD HH:MM - HH:MM              （日付＋時刻範囲）
  // 3. YYYY/MM/DD HH:MM                      （日付＋時刻）
  // 4. YYYY/MM/DD - YYYY/MM/DD               （日付範囲）
  // 5. YYYY/MM/DD                             （日付のみ）
  // 6. 相対日付（今日/明日/明後日） → parseRelativeDate()で解決
  // ... 各パターンを順に試行してマッチしたものを返す
}
```

内部では `chrono-node` のような既存ライブラリは使わず、自前で実装しました。理由は「日本語の相対日付対応」と「日付範囲・時刻範囲のサポート」を両立するライブラリが案外なかったからです。

## CLIとDiscord両対応の工夫

### CLI (commander.js + chalk)

PCからの利用がメインです。爆速で起動して一発でメモを残せることを重視しました。

```bash
# メモを追加（一番よく使う）
mycow memo "2026/06/15 18:00 歯科予約"

# 今日の予定を表示
mycow today

# 今週の予定を表示
mycow week

# キーワード検索
mycow search "歯医者"

# 全メモ一覧
mycow list
```

CLIはパイプラインにも組み込めるように設計しており、`mycow memo "内容"` はAPIを叩いて結果をJSONまたはプレーンテキストで返します。これにより、スクリプトから呼び出したり、エイリアスに仕込んで超高速起動することも可能です。

```typescript
// CLIのエントリポイント（簡略化）
import { Command } from "commander";
import chalk from "chalk";

const program = new Command();

program
  .name("mycow")
  .description("Personal memo & schedule tool")
  .version("0.1.0");

program
  .command("memo")
  .description("Add a memo")
  .argument("<text>", "memo text")
  .action(async (text: string) => {
    const res = await fetch(`${API_URL}/entries`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ text, source: "cli" }),
    });
    const data = await res.json();
    console.log(chalk.green("✓"), "Memo saved:", data.id);
  });

program
  .command("today")
  .description("Show today's schedule")
  .action(async () => {
    const res = await fetch(`${API_URL}/entries?date=today`);
    const data = await res.json();
    for (const item of data.items) {
      console.log(chalk.cyan(item.time || ""), item.text);
    }
  });

program.parse();
```

### Discord Bot (Slash Commands + ボタン)

スマホからの利用はDiscord Botが担当します。Slash Commandsで直感的に操作できるようにしました。

```
/memo content:2026/06/15 18:00 歯科予約
→ 「メモを保存しました ✅」と返信

/today
→ 今日の予定を一覧表示（編集/削除ボタン付き）

/search query:歯医者
→ 検索結果を表示

/list
→ 全メモ一覧
```

一覧表示にはDiscordの**Buttonコンポーネント**を使い、各メモに「編集」と「削除」のボタンを付与しています。ユーザーがボタンをクリックすると、Botがインタラクションを受け取ってLambdaの該当エンドポイントを叩きます。

これにより、スマホでもタップだけでメモの編集・削除が完了します。

### 共通データモデル

CLIとDiscord Botは同じバックエンド（Lambda + DynamoDB）を共有しています。データモデルは1エンティティでメモとスケジュールを両方管理します。

```typescript
interface MemoEntry {
  id: string;            // UUID
  title: string;         // タイトル（本文の最初の行）
  body: string;          // メモ本文
  entryType: "memo" | "schedule";  // メモ or スケジュール
  tags: string[];        // タグ
  pinned: boolean;       // ピン留め
  startDate?: string;    // 開始日（YYYY-MM-DD）
  endDate?: string;      // 終了日（YYYY-MM-DD）
  startTime?: string;    // 開始時刻（HH:mm）
  endTime?: string;      // 終了時刻（HH:mm）
  createdAt: string;     // ISO日時
  updatedAt: string;     // ISO日時
}
```

DynamoDBのソートキー（`sk`）に日付を含めることで、`today` や `week` のクエリを効率的に行えます。`BETWEEN` オペレーションで日付範囲を一発取得です。

## ハマったポイント

### Discordのed25519署名検証（地獄）

Discord BotとAPI Gateway/Lambdaを組み合わせる際、最大の難関は**リクエストの署名検証**でした。

DiscordはSlash Commandのリクエストが本当にDiscordから来たものか検証するため、**ed25519署名**を使います。公開鍵はDiscord Developer Portalで取得でき、32バイトの生バイト列です。

ここでハマりました。Node.jsの標準 `crypto.verify()` に生の32バイト鍵をそのまま渡しても動かないのです。

```typescript
// 動かないコード（32バイト生鍵をそのまま渡す）
const publicKey = Buffer.from(DISCORD_PUBLIC_KEY, "hex"); // 32 bytes
const isValid = crypto.verify(
  "ed25519",
  Buffer.from(timestamp + signature),
  publicKey,          // ❌ 生バイトはダメ
  Buffer.from(signature, "hex")
);
```

**原因**: Node.jsの `crypto.verify()` は公開鍵を **SPKI (SubjectPublicKeyInfo) 形式**で受け取る必要があります。Discordが提供する32バイトの生鍵は生のed25519公開鍵であり、そのままではSPKIとして認識されません。

**解決策**: 32バイトの生鍵にSPKIヘッダーを付与してDERエンコードし、`createPublicKey()` で読み込ませます。

```typescript
// 動くコード
const publicKeyBytes = Buffer.from(DISCORD_PUBLIC_KEY, "hex");

// SPKIヘッダー（ed25519用）
// 302a 3005 0603 2b65 7003 2100 + 32バイト鍵
const spkiHeader = Buffer.from([
  0x30, 0x2a, 0x30, 0x05, 0x06, 0x03, 0x2b, 0x65,
  0x70, 0x03, 0x21, 0x00,
]);
const spkiDer = Buffer.concat([spkiHeader, publicKeyBytes]);

const publicKey = crypto.createPublicKey({
  key: spkiDer,
  format: "der",
  type: "spki",
});

const isValid = crypto.verify(
  null,  // ← アルゴリズムはnull（鍵から自動判別）。"ed25519" を指定すると動かない
  Buffer.from(signatureTimestamp + jsonBody),
  publicKey,
  Buffer.from(signature, "hex"),
);
```

2つ目の落とし穴が **アルゴリズム指定** です。`crypto.verify()` の第1引数に `"ed25519"` を渡すと「アルゴリズムが見つからない」とエラーになります。正解は `null`（鍵オブジェクトから自動判別）です。これはNode.jsのED25519対応がまだ比較的新しく、ドキュメントと実際の挙動に乖離がある部分でした。

### LambdaとAPI Gatewayのレスポンス形式

DiscordはInteractionに対して `{ type: 1 }` のようなJSONを期待しますが、API Gateway Lambda統合の場合、Lambdaは `{ statusCode: 200, body: "..." }` の形式でレスポンスを返す必要があります。

このミスマッチを吸収するラッパー関数を作りました。

```typescript
// Discord Interaction Lambda ハンドラ
export async function handler(
  event: { body?: string; headers?: Record<string, string> },
): Promise<{ statusCode: number; headers: Record<string, string>; body: string }> {
  // Discordからのリクエストの署名検証
  const publicKey = process.env.DISCORD_PUBLIC_KEY;
  if (publicKey) {
    const headers = event.headers ?? {};
    const signature = headers["x-signature-ed25519"] ?? "";
    const timestamp = headers["x-signature-timestamp"] ?? "";
    const body = event.body ?? "";

    const isValid = verifyDiscordRequest(publicKey, signature, timestamp, body);
    if (!isValid) {
      return { statusCode: 401, headers: { "Content-Type": "application/json" }, body: JSON.stringify({ error: "Invalid signature" }) };
    }
  }

  if (!event.body) {
    return { statusCode: 400, headers: { "Content-Type": "application/json" }, body: JSON.stringify({ error: "Missing body" }) };
  }

  const interaction = JSON.parse(event.body);

  // TYPE 1: PING（疎通確認）
  if (interaction.type === 1) {
    return {
      statusCode: 200,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ type: 1 }),  // PONG
    };
  }

  // TYPE 2: Slash Command の処理...
  const result = await handleCommand(interaction);
  return {
    statusCode: 200,
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(result),
  };
}
```

このラッパー関数で、CLIとDiscordの両方からのリクエストを同じLambdaハンドラで処理できるようになりました。

## まとめ

### 作ったもの: mycow2

名前は「My Cow...じゃなくて、My Co-Worker」の略です。2は2作目という意味で、最初はPython + ローカルSQLiteで作っていましたが、フルクラウド版への移行に伴ってバージョンが上がりました。

リポジトリはprivateで、argonpt2プロジェクトのサブディレクトリとして管理しています。CDKのスタック定義、Lambdaのコード、CLI、Discord Botのコードがすべて同じリポジトリに入っています。

### 今後の課題

- **Discordモーダルでの編集UI**: 現在はコマンドベースで編集していますが、モーダルUIにすればスマホでの操作性が格段に向上します
- **リマインダー機能**: EventBridge Scheduler + SNS で、指定日時にDiscordに通知を飛ばす仕組みを検討中
- **ファイル添付**: Discordの添付ファイルをS3に保存して、メモにリンクを残す機能

### コードは全部AIエージェントに書かせた

最後に補足ですが、このツールのコードはほぼすべて**12人のBuild Pipelineエージェント**に書かせています。code-researcherが既存コードを調査し、story-writerが仕様を整理し、backend-builderがLambdaのコードを実装し、test-verifierがテストを書く、という流れです。

人間である私は「日付パーサーはこんな仕様で」「Discord署名検証の落とし穴はここ」といった方向性の指示と、コードレビューだけを行いました。ed25519の署名検証のハマりポイントも、実はtest-verifierが「署名検証テストが通らない」と差し戻しをかけてくれたおかげで発覚したものです。

AIエージェントにコードを書かせる時代、人間の役割は「設計判断」と「品質保証の最終チェック」にシフトしている実感があります。

---

**関連リンク**

- このブログのリポジトリ: <https://github.com/tomoln/argonpt2>
- AWS CDK: <https://docs.aws.amazon.com/cdk/v2/guide/home.html>
- Discord Interactions: <https://discord.com/developers/docs/interactions/receiving-and-responding>
- Agent Arc（12エージェントパイプライン）の紹介記事: </posts/agent-arc-first-complete-run/>
