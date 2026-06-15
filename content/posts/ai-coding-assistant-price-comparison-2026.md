---
title: "opencode go が便利という話"
date: 2026-06-15
draft: true
categories: ["tech"]
tags: ["opencode", "opencode-go", "ai-coding-assistant", "terminal"]
---

しばらく `opencode` を使い続けてみて「これはかなり便利だな」と感じたので、実際に使って良かったポイントをいくつか紹介します。ちなみに `opencode` 自体はCLIツールで、その上の有料サービスとして **OpenCode Go（月額$10固定）** があります。この記事では両方含めて「opencode」と書いていきます。

## セッションがフリーズしても安心

opencode を使っていると、たまに挙動がフリーズすることがあります。そんなとき「セッションIDを控えてなかった……！」と焦る必要はありません。

```bash
# セッション一覧を表示
opencode session list
```

これを実行すると、過去の全セッションが **ID・タイトル・更新日時** 付きで一覧表示されます。

```
Session ID                      Title                      Updated
──────────────────────────────────────────────────────────────────
ses_13712a3e5ffe...  opencodeフリーズ時の過去セッション復元方法  10:38
ses_137524b2effe...  opencodeのセッション履歴表示         9:28
ses_13fbfbaa4ffe...  mycow2 メモ/スケジュール管理ツール      23:21
...
```

タイトルで判断できるので、IDを覚えていなくても目的のセッションを探せます。

続行するときは：

```bash
# 最後のセッションを続行（フリーズ後はこれ一発）
opencode -c

# 特定のセッションを指定
opencode -s ses_13712a3e5ffe...
```

## 知っておくと便利なコマンド

セッション関連以外にも、日常使いで役立つコマンドが揃っています。

```bash
# 使用統計とコストを表示（直近7日間）
opencode stats --days 7

# 使える全モデルを一覧
opencode models

# 特定プロバイダのモデルだけ表示
opencode models google

# モデル情報を詳細表示（コスト含む）
opencode models --verbose

# セッションをJSONでエクスポート（バックアップ用）
opencode export <sessionID> --sanitize

# ノンインタラクティブ実行（スクリプト連携に便利）
opencode run "このコードのバグを探して"

# ファイルを添付して実行
opencode run -f src/main.ts "型エラーを直して"

# デバッグ情報
opencode debug paths
opencode debug config
```

特に `opencode stats` は「今月いくら使ったか」がすぐ分かるので、予算管理に便利です。

## 気になるお値段

opencode で使えるモデルはいくつかの層に分かれています：

- **無料モデル**（`opencode/north-mini-code-free` など）→ 料金発生なし
- **OpenCode Go モデル**（`opencode-go/deepseek-v4-flash` など）→ **月額$10固定**で使い放題
- **外部プロバイダモデル**（Google / Groq など）→ 各プロバイダの従量課金

このうち OpenCode Go は **月額$10の固定料金** で、専用のモデル群が使い放題になります。実際に7日間使ってみた統計（外部プロバイダ含む全体）がこちら：

```
Sessions: 57 | Messages: 2,720
Total Cost: $7.98 (7日間)
```

57セッション、2,720メッセージで **$7.98**。無料モデルと OpenCode Go の $10固定をうまく組み合わせれば、かなり安く抑えられます。

## 他のサービスと比べても安い

同じAIコーディングアシスタントの価格をざっと並べてみると：

| サービス | 月額（ヘビーユース想定） |
|---------|----------------------|
| **OpenCode Go** | **$10/月 固定** |
| GitHub Copilot Pro | $10〜（クレジット超過時は追加） |
| GitHub Copilot Pro+ | $39 |
| Cursor Pro | $20 |
| Claude Code Pro | $17〜$20 |
| Claude Code Max | $100〜$200 |
| Devin Pro | $20 |

OpenCode Go は GitHub Copilot Pro と並んで最も安い部類です。ただ Copilot Pro は $15/月分のクレジット制で、フロンティアモデルをがっつり使うとすぐ追加料金が発生します。その点 OpenCode Go は **月額$10の固定** なので、使いすぎを気にせず色んなモデルを試せます。

## 色んなLLMを気軽に試せる

opencode の一番の魅力は、**50種類以上のモデル** を自由に切り替えながら使えることです。

```bash
# モデルを指定して起動
opencode -m google/gemini-2.5-flash-lite
opencode -m opencode-go/deepseek-v4-flash
opencode -m opencode/north-mini-code-free
```

「このタスクは安いモデルで十分」「難しい設計判断は高性能モデルで」という使い分けが、同じツール内で完結します。新しいモデルが出たら `opencode models --refresh` で一発反映。Google、Anthropic、OpenAI、Grok……各社の最新モデルを、追加のサブスクリプションなしで試せるのはかなり便利です。

他のサービス（Cursor や Claude Code など）は使えるモデルが限られているか、高額プランにしないとモデルを選べないケースが多いです。いろんなLLMを検証したい人ほど、OpenCode Go のモデル自由度は大きなアドバンテージになります。

## まとめ

opencode を実際に使ってみて感じたのは「必要十分な機能が揃っていて、しかも安い」の一言です。

- セッション管理がしっかりしていて、フリーズしても安心
- 便利なコマンドが揃っている
- OpenCode Go は月額$10固定で専用モデル使い放題
- 無料モデルも含めると50種類以上のモデルを自由に使える
- 他のサービスと比べても価格面で十分競争力がある

ターミナルTUIというと「難しそう」と思われるかもしれませんが、慣れるとGUIより速い場面も多く、一度使い始めると手放せなくなります。AIコーディングアシスタントを探している人は、一度試してみる価値があると思います。
