---
title: "M4Lデバイス開発をAIに委ねる: MCP Server構築記"
date: 2026-06-21
draft: false
tags: ["opencode", "M4L", "Max for Live", "MCP", "TypeScript", "Zod"]
categories: ["技術"]
description: "opencodeのMCP (Model Context Protocol) サポートを活用し、Max for Liveデバイス開発をAIに委ねるMCP ServerをTypeScript + Zodで構築した過程。53テスト全PASS、ループエンジニアリングの実践から得た知見。"
---

## はじめに

Max for Live（M4L）はAbleton Live上で動作するプラグイン形式で、Max/MSPのパッチをLiveのセッション内で実行できる。通常、M4Lデバイスの開発はMaxアプリケーション上でビジュアルプログラミングにより行うが、これを**AIエージェントに委ねる**ことはできないか。

本記事では、opencode（AIコーディングエージェント）のMCP（Model Context Protocol）サポートを活用し、M4LデバイスをAIが自律的に生成・検証・デバッグできるパイプラインを構築した過程を記録する。

## 発見: .maxpat はJSONだった

最初のブレイクスルーは、M4Lデバイスのソース形式である `.maxpat` ファイルが **JSON形式** であるという発見だ。

```typescript
// .maxpat の型定義（抜粋）
interface MaxPatcher {
  patcher: {
    boxes: MaxBox[];     // オブジェクト一覧
    lines: MaxLine[];    // パッチケーブル接続
    appversion: string;
    classnamespace: string;
  };
}

interface MaxBox {
  box: {
    id: string;
    maxclass: string;   // "noteout" | "transport" | "live.dial" など
    text?: string;      // オブジェクトの引数
    rect: [number, number, number, number]; // [x, y, w, h]
    varname?: string;
    parameter_initial?: number;
  };
}
```

つまり、Node.jsからプログラムとして生成できる。これにより、「AIにデバイスを設計させる → JSONを生成する → `.maxpat` として保存する」という経路が現実的になった。

## アーキテクチャ選定: CLIではなくMCP Serverに

当初はCLIツールとして実装するつもりだったが、調査中に **opencodeがMCP（Model Context Protocol）をネイティブサポート**していることを発見した。

MCPはAnthropicが策定したAI→ツール連携の標準プロトコルで、opencodeはMCPホストとして動作し、MCPサーバーが公開するツールを**LLMが自動認識して呼び出せる**。

CLIツールではなくMCP Serverとして実装することで：
- AIが会話の文脈を保ったままツールを呼び出せる
- Zodスキーマがそのままツールパラメータの型定義として機能
- ファイル入出力を介さずにJSONを直接やり取りできる

## 実装した6つのMCPツール

TypeScript + `@modelcontextprotocol/sdk` + Zod の組み合わせで、以下の6ツールを実装した：

| ツール名 | 機能 | 主なパラメータ |
|---------|------|--------------|
| `m4l_generate_device` | 仕様から `.maxpat` を生成 | name, noteDivision, noteNumber, velocity, channel, swing, noteLength |
| `m4l_validate_device` | 構造検証（必須オブジェクト・接続・パラメータ範囲） | maxpatJson |
| `m4l_simulate_device` | BPM/Note Divisionに基づくMIDIタイミングシミュレーション | bpm, division, durationBars, swing... |
| `m4l_inspect_device` | 内部構造解析（summary/fullモード） | maxpatJson, detail |
| `m4l_package_device` | `.amxd` 化の手順をMarkdownで提供 | name |
| `m4l_test_pipeline` | 生成→検証→シミュレーションを一括実行 | name, bpm, noteDivision... |

### ツール定義の実装パターン

各ツールはZodスキーマで入力をバリデーションし、薄いHandler層がService層に委譲するパターンで統一した：

```typescript
// src/tools/generateDevice.ts
import { z } from "zod";

export const GenerateDeviceInputSchema = z.object({
  name: z.string().min(1, "Device name is required").max(100),
  noteDivision: z.number().refine(v => [4, 8, 16, 32].includes(v)).default(4),
  noteNumber: z.number().min(0).max(127).default(60),
  velocity: z.number().min(1).max(127).default(100),
  channel: z.number().min(1).max(16).default(1),
  swing: z.number().min(0).max(1).default(0.5),
  noteLength: z.number().min(0).max(1).default(0.5),
});

export async function handleGenerateDevice(input) {
  const patcher = buildDeviceFromTemplate({
    name: input.name,
    tempoSync: {
      noteDivision: input.noteDivision,
      noteNumber: input.noteNumber,
      // ...
    },
  });
  return {
    content: [{ type: "text", text: JSON.stringify(patcher, null, 2) }],
  };
}
```

### 3層アーキテクチャ

```
[Handler層] src/tools/*.ts     — MCPリクエスト/レスポンス形式への変換
[Service層] src/lib/*.ts       — ビジネスロジック（生成・検証・シミュレーション）
[Domain層] src/types/maxpat.ts — .maxpatの型定義
```

各層を分離することで、独立したテストが可能になっている。Service層はすべて純粋関数として実装し、副作用を排除した。

### .maxpat生成エンジン

テンプレートJSONに `{{variable}}` 形式のプレースホルダーを埋め込み、実行時に文字列置換する方式を採用した：

```typescript
// src/lib/maxpatBuilder.ts （生テキスト置換 → JSONパース）
const raw = readFileSync(templatePath, "utf-8");
let processed = raw
  .replace(/\{\{noteDivision\}\}/g, String(4))
  .replace(/\{\{noteNumber\}\}/g, String(60))
  .replace(/\{\{velocity\}\}/g, String(100));
const patcher = JSON.parse(processed) as MaxPatcher;
```

置換後にbox IDをUUIDに差し替え、lines（パッチコード）のsource/destinationも追随して更新する。これによりテンプレートの再利用とIDの一意性を両立している。

## opencode統合

opencodeの設定ファイルにMCPサーバーの起動コマンドを追加するだけで統合完了する：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "default_agent": "m4l-engineer",
  "mcp": {
    "m4l": {
      "type": "local",
      "command": ["npx", "tsx", "src/index.ts"],
      "enabled": true
    }
  }
}
```

さらに、`.opencode/agents/` にカスタムエージェント定義をYAML frontmatter形式で配置する：

```yaml
---
description: M4Lデバイス開発エンジニア。6つのMCPツールを駆使してM4Lデバイスを自律開発する
mode: primary
temperature: 0.2
permission:
  read: allow
  edit: allow
  bash: allow
  task:
    "*": allow
---
あなたはm4l-engineerです。6つのMCPツールを駆使してM4Lデバイスを開発します...
```

## VERIFY: 53テスト全PASS

3つのテストファイルにわたる53のテストケースを実装し、全て通過した：

```bash
 ✓ tests/tempo-calc.test.ts      (30 tests)
 ✓ tests/maxpat-builder.test.ts  (11 tests)
 ✓ tests/mcp-server.test.ts      (12 tests)

 Test Files  3 passed (3)
      Tests  53 passed (53)
```

テストの内訳：
- **tempo-calc.test.ts** (30 tests): タイミング計算、clamp、`clampDivision` のロジック検証
- **maxpat-builder.test.ts** (11 tests): `.maxpat` 生成、必須オブジェクト存在確認、自己検証
- **mcp-server.test.ts** (12 tests): 全6ツールの正常系・異常系の結合テスト

## ループエンジニアリングの軌道修正

このプロジェクトで特筆すべきは、**ゴールが一度誤って解釈され、ループ内で修正された** ことだ。

初期の解釈：「M4Lデバイスを作成する」
↓ DISCOVERフェーズで以下の発見
- .maxpatはJSON → Node.jsで生成可能
- opencodeはMCPをネイティブサポート
- `@modelcontextprotocol/sdk` が存在する
↓ **ゴール修正**
「AIがM4LデバイスをCLI/MCPベースで生成・検証・デバッグできるパイプラインを構築する」
↓ さらに **MCP Serverとして実装する判断**

## まとめと展望

今回の取り組みで得られた知見：

1. **MCPはCLIの進化形**: 従来のCLIツールはAIとの相性が良くない。MCPのようにコンテキストを共有できるプロトコルが、AI時代の標準インターフェースになる。

2. **型安全が安心感を生む**: TypeScript + Zodの組み合わせで、特にLLMが生成するパラメータのバリデーションを実行時に行える安心感がある。

3. **ループの軌道修正を恐れない**: 初回の計画が間違っていても、DISCOVER→EVALUATEのループを高速に回せばより良い解に収束する。プロンプトを「打つ」のではなくループを「設計する」というマインドセットが重要。

今回のアプローチはM4Lに限らず、**任意の外部ツールやAPIをAIエージェントと統合する汎用的なパターン**として応用できる。皆さんも自分のドメインでMCP Serverを試してみてほしい。
