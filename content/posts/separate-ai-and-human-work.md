---
title: "AI作業と人間作業を分離する"
slug: "separate-ai-and-human-work"
date: 2026-06-13
description: "opencodeエージェントをローカル端末から分離して動かすため、無料VMサービス4種を調査しGitHub Codespacesで試した記録"
tags: ["opencode", "Codespaces", "VM", "DevContainer"]
draft: false
tldr: "opencodeと人間作業の干渉を避けるため無料VM4種を比較。CodespacesのDevContainerを構築したがopencodeは動作せず。原因は調査中。次はOracle Cloud ARM VMを検討する。"
---

## この記事でわかること

- opencode（AIコーディングエージェント）を同じ端末で人間作業と同時に行うと何が起きるか
- 無料で使えるVM環境4種の比較
- GitHub CodespacesでDevContainer環境を構築した結果

---

## 問題の発生

筆者はopencode（AIによる自律コーディングエージェント）を使って開発を進めている。パイプラインに沿って12人のAIエージェントがコード調査→実装→テスト→ブログ公開までを自動実行する仕組みだ。

しかし、同じWindows端末でopencodeを稼働させながら別の作業をしようとすると、以下の問題が発生した。

### 確認できた問題

**文字化け**:
opencodeがPowerShellを介してコマンドを実行した後、コンソールの出力が文字化けする。具体的には、Shift-JISとUTF-8の混在により `�����ꏊ` のような読めない文字列がセッションログに数十箇所記録された。これはopencodeのbash実行がPowerShellのエンコーディング設定を破壊することが原因と見られる。

**タイムアウト**:
`npm install -g netlify-cli` のような重い処理が60秒のタイムアウトに達した。AIが裏で処理を実行中に人間が別の操作をすると、リソース競合で応答が遅延する。

**git操作の競合リスク**:
opencode（site-publisher）が `git push` を実行しようとしたタイミングで、人間も別のgit操作を行うと履歴が競合するリスクがある。opencodeのエージェント定義には「git pushは人間承認必要」と明記されているが、これは競合を避けるための設計判断である。

### なぜ問題になるか

筆者の端末スペックは以下の通り：

| 項目 | 値 |
|------|------|
| CPU | Intel Core i5-8500 (6コア) |
| RAM | **8GB** |
| 空きディスク | 367GB |

8GB RAMではopencode + Node.js + ブラウザ + 人間の作業を同時に動かすにはメモリが不足する。さらにopencodeはVSCodeのターミナルを操作するため、人間のキー入力とAIのコマンド実行が衝突する可能性がある。

**結論：AIエージェントは別のマシンで動かすべき** という判断に至った。

---

## 無料VM4種の比較調査

「費用をかけず、安全に」を条件に、以下の4つの無料VMサービスを調査した。

### GitHub Codespaces

| 項目 | 内容 |
|------|------|
| **無料枠** | 月120時間 / 15GBストレージ |
| **超過料金** | $0.18/時間（2コア） |
| **VMスペック** | 2コア / 8GB RAM / 32GB〜64GBストレージ |
| **セキュリティ** | GitHub認証、HTTPS通信、企業グレード |
| **VSCode統合** | ネイティブ（Remote Explorer） |

メリット：すでにGitHub CLIで認証済みのため追加設定不要。
デメリット：月120時間の無料枠は24時間稼働には不足する。

### Google Cloud Free Tier

| 項目 | 内容 |
|------|------|
| **無料枠** | e2-micro VM 1台（永久無料）+ 30GBストレージ |
| **VMスペック** | 0.25 vCPU / 1GB RAM |
| **セキュリティ** | IAM認証、SSH鍵 |

1GB RAMではopencode + Node.jsを動かすには非力と判断した。

### AWS Free Tier

| 項目 | 内容 |
|------|------|
| **無料枠** | t2.micro / t3.micro 750時間/月（12ヶ月限定）+ $200クレジット |
| **VMスペック** | 1 vCPU / 1GB RAM |
| **セキュリティ** | IAM認証、Security Groups |

12ヶ月の期限があり長期運用には不向き。また1GB RAMではCodespaces同様に非力。

### Oracle Cloud Free Tier

| 項目 | 内容 |
|------|------|
| **無料枠** | Ampere A1 (ARM) 最大4コア / 24GB RAM（永久無料） |
| **VMスペック** | 選べる（最大4コア/24GB） |
| **セキュリティ** | IAM認証、SSH鍵、VPCファイアウォール |
| **デメリット** | ARMアーキテクチャ、日本リージョンの在庫問題、クレカ必須 |

スペックは群を抜いて高い。4コア24GB RAMが永久無料。ただしARMアーキテクチャのためNode.jsは動くが一部パッケージで問題が出る可能性がある。

### 比較結果

| サービス | コスト | スペック | セットアップ容易性 | 総合 |
|---------|:------:|:--------:|:-----------------:|:----:|
| **GitHub Codespaces** | ◎（無料枠あり） | ◯（2コア8GB） | **◎** | **🥇** |
| GCP Free Tier | ◎（永久無料） | △（1GB RAM） | ◯ | △ |
| AWS Free Tier | ◯（12ヶ月無料） | △（1GB RAM） | ◯ | △ |
| Oracle Cloud Free Tier | ◎（永久無料） | **◎（4コア24GB）** | △（ARM） | ◯ |

最もセットアップが容易な **GitHub Codespaces** から試すことにした。

---

## Codespaces環境の構築

以下の手順で、opencodeを動かすためのCodespaces環境を構築した。

### セットアップしたファイル

**`.devcontainer/devcontainer.json`**:
ベースイメージに `typescript-node:22`（Node.js v22）を指定。GitHub CLIとDocker-in-DockerのDevContainer Featuresを追加し、コンテナ起動後に以下の処理を自動実行するpostCreateCommandを設定した。

**`.devcontainer/post-create.sh`**:
- Hugo Extended v0.163.1 のインストール（ブログビルド用）
- opencode CLI のグローバルインストール
- blog-cli 依存関係のインストール
- Playwright + Chromium のインストール（UIテスト用）

### opencodeラッパースクリプト

グローバルインストールのPATH問題を避けるため、`/usr/local/bin/opencode` にラッパースクリプトを作成した。このスクリプトは以下の優先順位でopencodeを実行する：

1. `/usr/local/lib/node_modules/@opencode-ai/cli/dist/cli.js`（global install）
2. `npx @opencode-ai/cli`（npx経由）

どの環境でも `opencode` コマンドが使える設計にした。

### GitHub Actions自動検証

pushのたびに以下のチェックを自動実行するワークフローも作成した：

| チェック項目 | 結果 |
|------------|:----:|
| opencode CLI install | ✅ 成功 |
| blog-cli build | ✅ 成功 |
| blog-cli test（67 tests） | ✅ 全パス |
| 全ツール検証 | ✅ 成功 |

CI上ではopencodeは正常に動作することを確認している。

---

## 結果：動作しなかった

しかし、実際にCodespacesに接続して `opencode` を実行すると、以下の結果になった。

```bash
$ opencode --version
bash: opencode: command not found
```

また、`npx @opencode-ai/cli --version` も同様に動作しなかった。

### わかっていることとわかっていないこと

| 項目 | 状態 |
|------|:----:|
| postCreateCommand が実行されたか | **不明**（Codespace内に入れず確認できない） |
| opencodeのインストール自体は成功したか | CIでは成功を確認済み |
| ラッパースクリプトが作成されたか | **不明**（sudo権限の問題かも） |
| PATHの問題か | 可能性はあるが未確認 |
| CIで動いてCodespaceで動かない理由 | **不明** |

### なぜ調査できないか

CodespaceはGitHub認証が必要な環境であり、opencodeの稼働している端末から直接アクセスして調査することができない。GitHub API経由の操作にはCodespace管理用の権限（`codespace`スコープ）が必要だが、現在の認証トークンにはこの権限が付与されていない。

権限追加にはブラウザでの承認操作が必要であり、完全自動化は現時点では不可能だった。

---

## 次に試せること

今回の検証で判明した課題から、以下の選択肢が残っている。

### 1. Dockerfile方式への切り替え

postCreateCommand（コンテナ起動後に実行）ではなく、Dockerfile（イメージビルド時に実行）でopencodeをインストールする方式。これにより：
- インストールがイメージの一部になる（再現性向上）
- PATH問題が起きにくい
- sudo権限の問題が回避できる

ただし、最初のビルドに時間がかかる。

### 2. Oracle Cloud ARM VMの検証

Codespaces以外の選択肢として、Oracle CloudのAmpere A1インスタンス（4コア/24GB RAM/永久無料）がある。SSH接続＋VSCode Remote Developmentで使う想定。ARMアーキテクチャだがNode.js v22は公式にARM64をサポートしている。

### 3. GitHub Token権限の再設定

Codespacesでopencodeを動かすために必要なtoken権限（codespaceスコープ）を改めて発行する。これを実現するにはブラウザでのデバイス認証フローを経由する必要がある。

---

## まとめ

opencodeと人間作業を同じ端末で行うと、文字化けやリソース競合が発生する。その解決策として4つの無料VMサービスを調査し、Codespacesでの構築を試みた。

CI上ではopencodeの動作確認ができたが、実際のCodespaces環境では `opencode: command not found` となり動作しなかった。原因の特定にはCodespace内へのアクセスが必要だが、現時点では認証の壁があり調査できていない。

次はDockerfile方式への移行か、Oracle Cloud ARM VMの検証を進める。

### 参考リンク

- [GitHub Codespaces ドキュメント](https://docs.github.com/ja/codespaces)
- [Oracle Cloud Free Tier](https://www.oracle.com/jp/cloud/free/)
- [Google Cloud Free Tier](https://cloud.google.com/free?hl=ja)
- [AWS Free Tier](https://aws.amazon.com/jp/free/)
- [DevContainer ドキュメント](https://containers.dev/)
