---
title: "個人メモツールのDiscord Bot UXを改善する"
slug: "improving-discord-bot-ux-for-personal-memo-tool"
date: 2026-06-15
draft: true
categories: ["技術記事"]
tags: ["AWS", "Discord", "TypeScript", "個人開発"]
description: "mycow2のDiscord Bot UXを三段階で改善。Discord Modalによる編集UI、柔軟な日付パーサー拡張、ActionRowの5行制限対策を解説。"
tldr: "個人メモツールmycow2のDiscord Bot UXを改善した記録。①Modalで編集UIを刷新（コマンド打鍵不要に）②日付パーサーの正規表現を緩和して1桁表記や年跨ぎに対応③ActionRowの5行制限にハマりボタンを5件までに制限。TypeScriptの実装コード付き。"
---

## はじめに

[前回の記事](/posts/building-personal-memo-tool-on-cloud/)では、AWSサーバーレスで個人メモツール「mycow2」を構築し、CLIとDiscord Botの両方から操作できるところまで作りました。

アーキテクチャ自体は安定して動作しています。しかし、**スマホからの操作性**に課題を感じていました。Discord Botはスマホが主要なインターフェースなのに、編集作業でいちいちUUIDをコピーしてスラッシュコマンドを打つのは現実的ではありません。また、日付の入力パターンにも柔軟性が足りませんでした。

そこで今回、以下の3つの改善を行いました。

1. **Discord Modal による編集UI** — ボタン一発で編集モーダルを開く
2. **日付パーサーの拡張** — 1桁表記や短縮記法に対応
3. **ActionRow 5行制限への対応** — Discord APIの制約と向き合う

## 1. Discord Modal で編集UIを改善する

### 改善前の状態

前回の実装では、一覧表示に「✏️編集」「🗑️削除」ボタンをつけていました。しかし、編集ボタンを押すと「現在の内容」が表示されるだけで、実際に編集するには以下のようなコマンドを手打ちする必要がありました。

```
/edit id:550e8400-e29b-41d4-a716-446655440000 content:新しい内容をここに書く
```

スマホでUUIDをコピーして、長いコマンドを手打ちするのは非現実的です。そもそも「編集ボタンがあるのに編集できない」というUX上の問題でした。

### Discord Modal とは

Discordの**Modal**（モーダル）は、アプリケーションがユーザーにフォーム入力を促すUIコンポーネントです。テキスト入力フィールドを複数配置でき、送信時にインタラクションとして結果を受け取れます。

Modalを開くには、Interaction Response Typeとして **type 9 (MODAL)** を返します。Modalの送信は別のインタラクション（type 5: MODAL_SUBMIT）として飛んでくるので、別途ハンドリングが必要です。

```typescript
// Modalを開くレスポンス（実際のmodalResponse関数）
function modalResponse(
  title: string,
  customId: string,
  inputs: DiscordModalTextInput[],
): DiscordResponse {
  return {
    type: 9,  // InteractionResponseType.MODAL
    data: {
      title,
      custom_id: customId,
      components: inputs.map((input) => ({
        type: 1,  // ActionRow
        components: [input],
      })),
    },
  };
}

// 使用例: 3つのフィールドを持つModal
modalResponse("✏️ メモを編集", "edit_modal|{entryId}", [
  { type: 4, custom_id: "date",   label: "日付 (YYYY-MM-DD)", style: 1, value: "2026-06-16" },
  { type: 4, custom_id: "time",   label: "時刻 (HH:MM)",      style: 1, value: "14:00" },
  { type: 4, custom_id: "content", label: "内容",             style: 2, value: "現在の本文", required: true },
]);
```

### 実装のポイント

コードを確認したところ、Modalの送信処理（`handleEditModal`）自体は既に実装されていました。問題は**ボタンがModalを開いていなかった**ことです。編集ボタンのハンドラで `type: 9` のレスポンスを返すように修正するだけで、既存のロジックがそのまま使えました。

```typescript
// 編集ボタンがクリックされたときのハンドラ（実際のコード）
async function handleEditButton(entryId: string): Promise<DiscordResponse> {
  const tableName = getTableName();
  const client = getDdbClient();

  // DynamoDBからエントリを取得
  const result = await client.send(new GetCommand({
    TableName: tableName,
    Key: { PK: `ENTRY#${entryId}`, SK: "METADATA" },
  }));

  if (!result.Item) {
    return messageResponse(`ID \`${entryId}\` のエントリは見つかりませんでした。`);
  }

  const entry = dynamoItemToEntry(result.Item as never);
  const typeIcon = entry.entryType === "schedule" ? "📅" : "✏️";

  // Modalを返す（type: 9）
  return modalResponse(`${typeIcon} ${entry.entryType === "schedule" ? "予定" : "メモ"}を編集`, `edit_modal|${entryId}`, [
    {
      type: 4,
      custom_id: "date",
      label: "日付 (YYYY-MM-DD)",
      style: 1,  // Short（1行）
      value: entry.startDate ?? "",
      required: false,
    },
    {
      type: 4,
      custom_id: "time",
      label: "時刻 (HH:MM)",
      style: 1,
      value: entry.startTime ?? "",
      required: false,
    },
    {
      type: 4,
      custom_id: "content",
      label: "内容",
      style: 2,  // Paragraph（複数行）
      value: entry.body,
      required: true,
    },
  ]);
}
```

### DynamoDBのGSI更新を伴う設計判断

Modalで日付が変更された場合、DynamoDBの**GSI（DateIdx）のソートキーも更新する必要**があります。DynamoDBではGSIのキー属性を直接更新できないため、単純なUpdateCommandではGSIの再インデックスが行われません。

解決策として、**PutCommand + ConditionExpression** を使い、エントリを全置換する方式にしました。

DynamoDBのGSIはベーステーブルの更新時に自動的に更新されるわけではありません。GSIのキー属性（`GSI2PK`, `GSI2SK`）はDynamoDBのアイテムの一部として保存されており、PutCommandで丸ごと書き換えることでGSIも再インデックスされます。

```typescript
// PutCommand + ConditionExpression で安全に全置換（実際のコード）
const existing = dynamoItemToEntry(result.Item as never);
const updated: MemoEntry = {
  ...existing,
  body: content,
  title: firstLine,
  startDate: dateStr.trim() || undefined,  // 空文字はundefinedに
  startTime: timeStr.trim() || undefined,
  updatedAt: new Date().toISOString(),
};

// entryToDynamoItemがGSIキーも含めてDynamoItemを生成
const dynamoItem = entryToDynamoItem(updated);

// PutCommand + ConditionExpression で安全に全置換
await client.send(new PutCommand({
  TableName: tableName,
  Item: dynamoItem,
  ConditionExpression: "attribute_exists(PK)",  // 存在確認でupsert防止
}));
```

`entryToDynamoItem()` 関数がGSIキーを自動計算します。scheduleタイプでstartDateがある場合、`GSI2PK = DATE#{YYYY-MM}`、`GSI2SK = {startDate}#{id}` が自動設定されます。PutCommandは同じPK+SKに対して上書きになりますが、`ConditionExpression: "attribute_exists(PK")` で存在確認することで、誤った新規作成を防止しています。

### 削除ボタンにも確認ステップを追加

ついでに、削除ボタンにも改善を加えました。誤タップ防止のため、削除前に確認ボタンを表示するようにしました。

```typescript
// 削除ボタンがクリックされたとき（実際のコード）
async function handleDeleteButton(entryId: string): Promise<DiscordResponse> {
  const tableName = getTableName();
  const client = getDdbClient();

  // DynamoDBからエントリを取得（存在確認）
  const result = await client.send(new GetCommand({
    TableName: tableName,
    Key: { PK: `ENTRY#${entryId}`, SK: "METADATA" },
  }));

  if (!result.Item) {
    return updateResponse(`ID \`${entryId}\` のエントリは見つかりませんでした。`);
  }

  const entry = dynamoItemToEntry(result.Item as never);

  // type: 7 (UPDATE_MESSAGE) で元メッセージを確認表示に置き換え
  return {
    type: 7,
    data: {
      content: `本当に削除しますか？\n**${entry.title}**`,
      components: [{
        type: 1,
        components: [
          { type: 2, style: 4, label: "✅ はい、削除する", custom_id: `confirm_delete|${entryId}` },
          { type: 2, style: 2, label: "❌ キャンセル",    custom_id: `cancel_delete|${entryId}` },
        ],
      }],
    },
  };
}
```

この確認ステップを入れることで、スマホの小さな画面での誤操作を防止できます。

## 2. 日付パーサーを拡張する

### 改善前の問題点

前回の実装では、日付パターンを厳密に `\d{4}[-/]\d{2}[-/]\d{2}` と指定していました。これにより、以下のような柔軟な入力がパースできませんでした。

- `2026/7/26` — 月や日が1桁（パースエラー）
- `2026/7/26-7/27 フジロック` — 年が省略された短縮記法
- `2026/12/30-1/3 年末年始` — 年跨ぎ

### 解決策：正規表現の緩和 + 短縮展開

まず、日付パターンの桁数指定を `\d{2}` から `\d{1,2}` に変更しました。

```typescript
// 変更前
const DATE_PATTERN = /\d{4}[-/]\d{2}[-/]\d{2}/;

// 変更後
const DATE_PATTERN = /\d{4}[-/]\d{1,2}[-/]\d{1,2}/;
```

このたった1文字の変更（`2` → `1,2`）で、`2026/7/26` のような1桁表記がすべてパースできるようになりました。

次に、短縮日付の展開ロジックを追加しました。

```typescript
// 年なしの短縮日付（M/D）を完全日付に展開する関数（実際のコード）
export function expandShortDate(
  shortDate: string,
  referenceYear: string,
  referenceMonth: number,
): string {
  const normalized = shortDate.replace(/\//g, "-");
  const parts = normalized.split("-");
  const month = parts[0].padStart(2, "0");
  const day = parts[1].padStart(2, "0");
  let year = parseInt(referenceYear, 10);

  // 年跨ぎの検出: 開始月が12月で短縮日付の月が1月なら翌年
  if (referenceMonth === 12 && parseInt(parts[0], 10) === 1) {
    year += 1;
  }

  return `${year}-${month}-${day}`;
}
```

実際のパース処理では、まず `YYYY/M/D - M/D` のパターンに正規表現でマッチさせ、2つ目の日付をこの関数で展開します。呼び出し側では開始日の年月を参照値として渡します。

```typescript
// 日付範囲パターン（2つ目が年なし）のマッチング（実際のコード）
const DATE_RANGE_SHORT_REGEX = new RegExp(
  `^(${DATE_PATTERN.source})\\s*[-–]\\s*(${SHORT_DATE_PATTERN.source})\\s*(.*)$`
);

// マッチした場合の処理
match = firstLine.match(DATE_RANGE_SHORT_REGEX);
if (match) {
  const startDate = normalizeDate(match[1]);  // "2026-07-26"
  const startYear = startDate.slice(0, 4);    // "2026"
  const startMonth = parseInt(startDate.slice(5, 7), 10);  // 7
  const endDateFull = expandShortDate(match[2], startYear, startMonth);
  // endDateFull = "2026-07-27"
}
```

このロジックにより、以下のような入力がすべて正しくパースできます。

| 入力 | パース結果 |
|------|-----------|
| `2026/7/26-7/27 フジロック` | 2026-07-26〜2026-07-27 |
| `2026/12/30-1/3 年末年始` | 2026-12-30〜2027-01-03 |
| `2026/7/26 18:00-21:00 待機` | 2026-07-26 18:00〜21:00 |

年跨ぎの判定は「開始日の月が12月で、短縮日付の月が1月の場合、年を+1する」というシンプルなロジックです。実際にこのツールを使っていて年跨ぎが必要になるのは年末年始の予定がほとんどなので、この限定ロジックで実用上十分です。

### 波及効果

興味深いのは、正規表現を `\d{1,2}` に緩和しただけで、日付範囲パターン・時刻付きパターン・時刻範囲パターンの**すべてのパーサーが自動的に1桁表記に対応した**ことです。複数のパーサーが同じ基底パターンを参照しているため、1箇所の変更が全体に波及しました。モジュール設計の重要性を再認識した瞬間でした。

## 3. Discord ActionRow の5行制限にハマる

### 問題

`/upcoming` コマンドで「今後1週間の予定」を最大10件表示する機能を実装していたときのことです。各予定に編集ボタンと削除ボタンを付けようとして、謎のエラーに直面しました。

```
アプリケーションが時間内に応答しませんでした
```

Lambdaの実行時間は問題ないし、CloudWatch Logsにもエラーはありません。CloudWatch Metricsで確認しても **Errors（エラー数）** はゼロのままでした。**

これは、Lambdaとしては正常にレスポンスを返しているのに、そのレスポンス内のコンポーネント構成がDiscord APIの仕様に違反していたため、Discordがレスポンスを「無効」として扱っていたことを意味します。API GatewayとLambdaの間では問題なく通信できているので、エラーハンドリングの外側で起きる問題は気付きにくいという教訓を得ました。

### 原因：ActionRowは最大5行

調査の結果、Discord APIの**ActionRow（コンポーネントの行）は最大5行**までしか配置できないことが原因でした。10件の予定にそれぞれ編集・削除の2ボタンを付けると、10件 × 1ActionRow（2ボタン）= 10 ActionRow となり、制限の5行を超えてしまいます。

違反時はDiscord APIがエラーを返し、Lambdaはエラーハンドリングできずに例外がスルーされていました。これが「時間内に応答しませんでした」というエラーメッセージとして表面化していたのです。

Discordの公式ドキュメントには以下のように記載されています。

> Action Rows are non-nested components that can have a maximum of 5 Action Rows per message. ([Discord Developer Docs — Message Components](https://discord.com/developers/docs/interactions/message-components))

見落としがちな制限です。

### 解決策：表示とボタンを分離

テキストは全10件表示しつつ、ボタンは最初の5件のみ付与するという妥協案で対応しました。

```typescript
// /upcoming コマンドの実際の実装（簡略化）
async function handleUpcomingCommand(): Promise<DiscordResponse> {
  // GSI2 (DateIdx) を使って現在〜2ヶ月先を月単位でクエリ
  const allItems: MemoEntry[] = [];

  for (const month of months) {
    const result = await client.send(new QueryCommand({
      TableName: tableName,
      IndexName: "DateIdx",
      KeyConditionExpression: "GSI2PK = :pk",
      ExpressionAttributeValues: { ":pk": `DATE#${month}` },
    }));
    if (result.Items) {
      for (const item of result.Items) {
        allItems.push(dynamoItemToEntry(item as never));
      }
    }
  }

  // 今日以降のscheduleのみ抽出、日付順ソート、最大10件
  const filtered = allItems
    .filter((e) => e.entryType === "schedule" && e.startDate && e.startDate >= today)
    .sort((a, b) => (a.startDate || "").localeCompare(b.startDate || ""))
    .slice(0, 10);

  // テキストは全10件表示
  const lines = filtered.map((e, i) =>
    `${i + 1}. 📅 ${e.startDate}(${wd})${e.startTime ? " " + e.startTime : ""} ${e.title}`
  );

  // ボタンは最初の5件のみ（ActionRow制限: 最大5行）
  const buttonLimit = Math.min(filtered.length, 5);
  const buttonRows = filtered.slice(0, buttonLimit).map((e, i) => entryButtons(e, i + 1));

  return buttonsResponse(`📅 **今後の予定**\n${lines.join("\n")}`, buttonRows);
}
```

完全ではないですが、「5件以上の予定を確認したい人はリストを見てください」という運用で妥協しています。もし全件にボタンが必要なら、メッセージを分割して複数メッセージで表示する方式も検討できます。

### デバッグの教訓

この問題のデバッグで役立ったのは**CloudWatch Metrics**でした。Logsにエラーが出ていない場合でも、MetricsのDuration/Errorsグラフを見ることで、「関数が早期に異常終了している」ことがわかりました。Lambda + Discord Botのデバッグでは、以下の3つをセットで確認するのがおすすめです。

1. **CloudWatch Logs** — 例外の詳細
2. **CloudWatch Metrics** — Duration / Errors / Throttles
3. **Discord Developer Portal** — インタラクションのエンドポイントログ

## まとめ

今回の改善で、個人メモツールmycow2のDiscord Bot UXが以下のように改善されました。

| 改善項目 | 変更前 | 変更後 |
|---------|--------|--------|
| 編集UI | コマンド手打ちが必要 | Modalでフォーム入力 |
| 削除操作 | ワンクリックで削除 | 確認ステップ追加 |
| 日付入力 | `2026/07/26` のみ | `2026/7/26`, `7/26` 等もOK |
| 短期間記法 | 非対応 | `7/26-7/27` で年跨ぎもOK |
| 一覧表示 | 全件ボタン（ActionRow制限超過） | 5件までボタン、テキストは10件 |

個人的に大きかったのはDiscord Modalの改善です。コードの大部分は既に存在していたのに「ボタンがModalを開くようにする」というたった1つの修正でUXが劇的に変わりました。**「機能があること」と「機能が使えること」の差**を痛感した経験でした。

ActionRowの5行制限は、今後の設計で常に頭の片隅に置いておく必要があります。特に動的なボタン生成を行う場合は、制限値を定数として定義して、それを超えないことをテストで保証する仕組みが欲しいところです。

### 今後の予定

- **リマインダー機能**: EventBridge Scheduler で指定日時にDiscord通知
- **ファイル添付**: Discordの添付ファイルをS3に保存
- **タグ管理**: タグの追加・削除・フィルタリングUI

地道に改善を続けていきます。

---

**関連リンク**

- [Discord Message Components (公式ドキュメント)](https://discord.com/developers/docs/interactions/message-components)
- [Discord Interaction Response Types](https://discord.com/developers/docs/interactions/receiving-and-responding#interaction-response-object)
- [AWS CloudWatch Metrics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/working_with_metrics.html)
- [前回の記事: クラウドで個人メモツールを作ってみる](/posts/building-personal-memo-tool-on-cloud/)
