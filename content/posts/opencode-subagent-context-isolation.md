---
title: "subagentのコンテキスト分離を理解する"
slug: "opencode-subagent-context-isolation"
date: 2026-06-26
draft: true
categories: ["技術解説"]
tags: ["OpenCode", "subagent", "マルチエージェント", "コンテキスト管理", "LLM"]
description: "OpenCodeのsubagent（子エージェント）は呼び出すたびに完全に独立したChild Sessionとして動作する。db/sessionテーブルのparent_id構造・opencode exportの出力・実測データ量の3方向からコンテキスト分離の実態を検証し、Orchestratorパターンにおける設計上の示唆を導く。"
---

## 背景

OpenCodeでは `opencode.json` または `.opencode/agents/*.md` でエージェントを定義し、`task` ツールを使ってsubagentを呼び出せる。筆者は14のsubagentを持つOrchestrator構成（argonpt2loop / Agent Arc）を運用しており、この構成で「subagentがどのようなコンテキストで動作するのか」を正確に理解する必要があった。

特に、ループを重ねるごとに親エージェントのコンテキストが肥大化する現象に悩まされ、「subagentのコンテキストは親から完全に独立しているのか」「結果はどのように親に戻るのか」を突き止めるため、以下の4方向から調査を実施した。

## 調査内容

### 検証方法

1. **公式ドキュメント調査**: OpenCode Agentsページでsubagentの仕様を確認
2. **動作検証**: orchestratorからcode-researcherを `task` 呼び出しし、Child Sessionが作成されることを確認
3. **データベース解析**: `~/.local/share/opencode/opencode.db`（SQLite）の `session` テーブルを直接クエリ
4. **/export の挙動確認**: 親セッションと子セッションそれぞれを `opencode export` し、含まれる情報を比較

## 原因 / 仕組み

### subagent呼び出し = 新規Child Session

`task` ツールでsubagentを呼ぶたびに、**完全に独立したChild Session**が作成される。子のsystem prompt・会話履歴・権限は親と完全に分離されている。

```
親セッション（orchestrator）:
┌─────────────────────────────────────┐
│ システムプロンプト（orchestrator用）  │
│ ユーザーとの対話履歴（66 messages）   │
│ subagent呼び出しの結果（累積）        │
│   → code-researcher の戻り値         │
│   → backend-builder の戻り値         │
│   → test-verifier の戻り値          │
│ コンテキスト使用率: 26.2KB           │
└─────────────────────────────────────┘
          │ task 呼び出し
          ▼

子セッション（code-researcher）:
┌─────────────────────────────────────┐
│ システムプロンプト（researcher用）    │
│ タスク指示（親から渡された内容のみ）   │
│ 会話履歴（6 messages）               │
│ コンテキスト使用率: 2.4KB            │
└─────────────────────────────────────┘
```

### 親→子は一方通行

子セッションは `parent_id` カラムで親を知っている。しかし親のexport出力には `childIDs: []` と出力され、**子の詳細情報は含まれない**。つまり親から子の会話内容を参照することはできない。

### データベース構造

OpenCodeはセッション情報をSQLiteで永続化している。`session` テーブルの構造は以下の通り。

```sql
CREATE TABLE session (
  id text PRIMARY KEY,
  parent_id text,        -- 親セッションのID（ルートはNULL）
  agent text,            -- 使用エージェント名
  title text,            -- セッションタイトル
  created_at text,
  updated_at text
  -- ※ 他にもカラムがあるが、主要なもののみ記載
);

CREATE INDEX session_parent_idx ON session (parent_id);
```

`parent_id` にインデックスが張られていることから、親子関係の検索が頻繁に行われることが分かる。

### TUIのコンテキスト表示の注意点

OpenCode TUIの右上に表示されるコンテキスト使用率（例: `95.2K(10%)`）は、**親セッションのみの値**である。子セッションのコンテキスト使用率は、子セッションに移動（Tab切替）しないと確認できない。

この仕様を見落とし、「コンテキスト使用率が低いから問題ない」と判断してしまうと、実際にコンテキストが肥大化していることに気づかないままループを続けることになる。

### 実測データ：あるセッションの例

検証で取得した実際のセッション情報が以下である。

| 項目 | 親セッション (orchestrator) | 子セッション (code-researcher) |
|------|---------------------------|-------------------------------|
| メッセージ数 | 66 | 6 |
| データ量 | 26.2KB | 2.4KB |
| 役割 | ユーザー対話 + ループ制御 | コードベース調査 |

別のセッション（7回のsubagent呼び出しを行ったケース）ではさらに顕著な差が出た。

| 項目 | 親 | 子7セッションの合計 |
|------|-----|-------------------|
| メッセージ数 | 85 | 28 |
| データ量 | 36.4KB | 11.3KB |

親が子の結果を全て積算していることが数値から明らかである。子は独立したコンテキストで素早くタスクを完了するが、その結果は必ず親のコンテキストに蓄積される。

## 解決策 / 設計への示唆

この発見から、Orchestratorパターンを設計する際の具体的な示唆が得られる。

### 1. subagentには「結果を使い捨てる」設計

subagentは「軽量ヘルパー」として使うのが適切である。複数回の呼び出し結果を集計・比較・加工する必要がある処理は向かない。一問一答のクエリとしてsubagentを活用し、親に戻す情報は **成功/失敗 + 要約** のみに抑える。

```text
# ❌ 避けるべき設計
subagentに大量のコードを読ませ、その全文を親に戻す
→ 親のコンテキストが数KB単位で肥大化

# ✅ 望ましい設計
subagentには「問題があれば報告する」だけに留めさせる
戻り値は {success: true, summary: "...", issues: [...]} のような最小構成
```

### 2. システムプロンプトの最適化

現在のOrchestrator構成では、全エージェントの役割定義・権限設定・ルールがシステムプロンプトに常時埋め込まれている。これが親のコンテキスト肥大化の一因となっている。

OpenCodeの **Skill** はオンデマンド読み込み（必要なときにのみプロンプトに注入される）のため、システムプロンプトの常時埋め込みを避けられる。以下のように整理する。

| 項目 | システムプロンプト常駐 | Skill（オンデマンド） |
|------|---------------------|-------------------|
| エージェント定義 | 常に埋め込まれる | 呼び出し時に注入 |
| ルールファイル | 常に埋め込まれる | 必要時のみ読み込み |
| doc ファイル | 手動で添付 | 自動で読み込み |
| コンテキスト効率 | 低い（常駐分のコスト） | 高い（必要なときだけ） |

システムプロンプトは必要最小限に抑え、詳細なルールはSkillのオンデマンド読み込みに任せる設計が望ましい。

### 3. compaction 設定の活用

OpenCodeには `compaction` 設定が用意されており、コンテキストの自動圧縮が可能である。

```json
{
  "compaction": {
    "auto": true,
    "prune": false,
    "reserved": 10000
  }
}
```

| 項目 | 説明 | デフォルト |
|------|------|-----------|
| `auto` | コンテキストが上限に近づいたら自動圧縮する | `true` |
| `prune` | 古いツール出力を削除してトークンを節約する | `false` |
| `reserved` | 圧縮中用に確保するトークンバッファ | 10000 |

ループ設計時はこの設定を活用することで、親コンテキストの肥大化を抑制できる。

## 学び・補足

### コンテキスト分離の本質

OpenCodeのsubagent機構は「呼ぶたびに新規Child Session」という堅牢な設計でコンテキスト分離を実現している。しかしその結果を管理する親の設計次第で、コンテキスト効率が大きく変わる。

**独立したコンテキスト ＝ 品質向上 ＋ コスト増** というトレードオフを認識した上で、subagentに何を任せ、何を親で処理するかの線引きが重要である。

### CLIで親子関係を確認する

公式のCLIには親子関係を一覧表示するコマンドがない。確認するにはSQLiteを直接操作する必要がある。

```bash
# 特定の子セッションを探す
sqlite3 ~/.local/share/opencode/opencode.db \
  "SELECT id, agent, title FROM session WHERE parent_id = 'ses_xxx';"

# 全ての親子関係を一覧
sqlite3 ~/.local/share/opencode/opencode.db \
  "SELECT s1.id, s1.agent, s2.id AS child_id, s2.agent AS child_agent
   FROM session s1
   LEFT JOIN session s2 ON s2.parent_id = s1.id
   WHERE s1.parent_id IS NULL
   ORDER BY s1.created_at;"
```

### Orchestratorパターンの適正規模

14のsubagentを抱えるOrchestrator構成では、subagentの結果が全て親に蓄積される特性を理解した上で、設計上の工夫（結果の最小化 / compaction設定 / Skill活用）が必須となる。Subagentは「軽量ヘルパー」として設計されており、多数のsubagentを統率する場合は特に戻り値の取捨選択が重要である。

### 知見のまとめ

- subagentは呼び出すたびに完全に独立したChild Sessionを作成する
- 親は子の結果を全て積算するため、ループを重ねるごとにコンテキストが肥大化する
- TUI右上のコンテキスト表示は親セッションのみの値である
- システムプロンプトは最小限にし、詳細はSkillのオンデマンド読み込みに任せる
- compaction設定（auto/prune）を積極的に活用する
- 親に戻す情報は成功/失敗＋要約のみに留める

OpenCodeでOrchestratorパターンを実装するなら、このコンテキスト分離の特性を理解した上で設計しないと、ループのたびにコンテキスト使用率が右肩上がりになる。適切な compaction 設定と戻り値の最小化を徹底し、効率的なパイプラインを構築してほしい。

---

*本記事は、OpenCode（opencode-go/deepseek-v4-flash）上の AI エージェント群によって生成され、専用の fact-checker および privacy-reviewer による検証を経て公開されています。*
