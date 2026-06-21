---
title: "WSL2 + Ubuntu 26.04 に opencode と VNC デスクトップを構築する"
date: 2026-06-21
draft: true
tags: ["WSL2", "Ubuntu", "opencode", "x11vnc", "noVNC", "AI", "DeepSeek"]
categories: ["Tech"]
description: "Windows上でWSL2+Ubuntuを使ってAIコーディングエージェントopencodeを動かし、x11vnc+noVNCでブラウザからGUIデスクトップにアクセスするまでのセットアップ記録"
---

## はじめに

最近、AI コーディングエージェント「opencode」が注目を集めています。
opencode は Anomaly 社が開発するオープンソースの AI エージェントで、MIT ライセンスのもとで公開されています（GitHub 177K Stars）。

特徴は：
- ターミナルベースの対話型 TUI
- 任意の LLM プロバイダに対応（OpenAI, Anthropic, DeepSeek など）
- LSP によるコードインテリジェンス
- MCP による機能拡張

今回は「ローカルの Windows 端末を汚さずに」これを動かす環境として、**WSL2 + Ubuntu** を選択しました。
結果的に GUI デスクトップまで構築することになったので、その試行錯誤も含めて記録します。

## 環境

| 項目 | 値 |
|------|-----|
| Windows | 10 Home (Insider Build 26200) |
| WSL2 | Ubuntu 26.04 LTS (Resolute Raccoon) |
| opencode | v1.17.9 |
| Node.js | v22.14.0 |
| デスクトップ | xfce4 |
| VNC | x11vnc + noVNC |

## WSL2 のセットアップ

### 発生した問題

最初に `wsl --install` を実行したところ、以下のエラーで失敗しました：

```
Linux カーネルがインストールされていません。
wsl --install を実行してください。
```

Windows の機能自体は有効になっていたものの、WSL2 の Linux カーネルパッケージが不足している状態でした。
手動で MSI をダウンロードしてインストールしようとしましたが、管理権限の昇格がうまくいかず失敗。

### 解決策

最終的に **winget** を使うことで解決しました：

```powershell
winget install Microsoft.WSL
winget install Canonical.Ubuntu.2404
```

これで WSL2 + Ubuntu 26.04 LTS が正常にインストールされ、`wsl -l -v` で稼働を確認できました。

## opencode のインストール

### Node.js のセットアップ

opencode は npm パッケージとしても配布されています。
Ubuntu 26.04 では apt のパッケージ名が異なっていたため、Node.js バイナリを直接ダウンロードしてインストールしました。

```bash
# Node.js v22.14.0 をインストール
wget https://nodejs.org/dist/v22.14.0/node-v22.14.0-linux-x64.tar.xz
tar -xf node-v22.14.0-linux-x64.tar.xz
sudo cp node-v22.14.0-linux-x64/bin/* /usr/local/bin/
```

### opencode のインストール

opencode の公式インストールスクリプトを使います：

```bash
curl -fsSL https://opencode.ai/install | bash
```

このスクリプトは `~/.opencode/bin/opencode` にバイナリを配置し、パスを通してくれます。

### 動作確認

```bash
export PATH="$HOME/.opencode/bin:$PATH"
opencode --version
# => 1.17.9
```

### LLM モデルの選択

opencode は多様な LLM プロバイダに対応しています。
OpenCode Zen ではテスト済みのモデルが提供されており、Free モデルもあります：

| モデル | 入力（100万トークン） | 出力（100万トークン） | 備考 |
|-------|---------------------|---------------------|------|
| DeepSeek V4 Flash Free (OpenCode Zen) | **無料** | **無料** | OpenCode Zen の提供。期間限定 |
| DeepSeek V4 Flash | $0.14 | $0.28 | 直接契約の場合の参考料金 |
| Claude Sonnet 4 | $3.00 | $15.00 | Anthropic 直接契約の場合 |

「DeepSeek V4 Flash Free」は OpenCode Zen が提供する無料枠のモデル名です。
期間限定ですが、今のところ無料で利用できます。
接続は opencode の TUI 内で `/connect` コマンドを実行し、OpenCode Zen を選択して API キーを設定するだけです。

## GUI デスクトップへの道のり

ここからは「せっかくだから Ubuntu のデスクトップも触ってみたい」という欲求から始まった試行錯誤です。

### 方式比較

以下の 4 つの方式を試しました：

| 方式 | 結果 | 原因 |
|------|------|------|
| **WSLg** | ❌ 動作せず | Insider Build で Wayland  compositor が未対応 |
| **xrdp** | ❌ 青画面 | xfce4 が Wayland 検出で正常起動せず |
| **TightVNC** | ❌ 黒画面 | 古い VNC サーバーで xfce4 がレンダリングされず |
| **TigerVNC** | ❌ 起動せず | DRI3/GPU 周りが WSL2 と非互換 |
| **x11vnc + noVNC** | ✅ **成功** | 既存 X11 セッションにアタッチ + ブラウザアクセス |

### 最終解: x11vnc + noVNC

紆余曲折の末、最も安定したのは **x11vnc + noVNC** の組み合わせでした。

**x11vnc** は既存の X11 ディスプレイにアタッチして VNC サーバーとして共有します。
**noVNC** は HTML5 ベースの VNC クライアントで、WebSocket 経由でブラウザから VNC に接続できます。

セットアップ手順：

```bash
# 1. x11vnc のインストール
sudo apt install x11vnc

# 2. xrdp の X サーバー上で動く xfce4 を x11vnc で共有
env -u WAYLAND_DISPLAY x11vnc -display :11 -forever -shared -rfbport 5900 -passwd ＜パスワード＞

# 3. noVNC で WebSocket 変換
websockify --web /usr/share/novnc 6080 localhost:5900
```

あとはブラウザで `http://localhost:6080/vnc.html` を開き、「Connect」→ パスワード入力でデスクトップが表示されます。

{{< figure src="/images/novnc-desktop.png" alt="noVNCで表示されたxfce4デスクトップ" caption="ブラウザからアクセスした xfce4 デスクトップ" >}}

## 最終的な構成

```
Windows 10 Home
└── WSL2 + Ubuntu 26.04
    ├── opencode v1.17.9 (AIエージェント)
    ├── xfce4 (デスクトップ環境)
    ├── xrdp (Xサーバー)
    ├── x11vnc (VNCサーバー、ポート5900)
    └── noVNC (ブラウザアクセス、ポート6080)
```

## 運用のポイント

### 日常の使い方

| 目的 | 方法 |
|------|------|
| opencode を使う | `wsl -d Ubuntu` → `opencode`（ターミナル） |
| GUI デスクトップ | ブラウザで `http://localhost:6080/vnc.html` |
| コード編集 | Windows の VS Code + WSL 拡張 |
| ファイルアクセス | `\\wsl.localhost\Ubuntu\home\ユーザー名\` |

### 起動用スクリプト

WSL 再起動後に毎回実行するのが面倒な場合は、以下のような起動スクリプトを用意しておくと便利です：

```bash
#!/bin/bash
# start-vnc.sh
env -u WAYLAND_DISPLAY x11vnc -display :11 -forever -shared -rfbport 5900 -passwd ＜パスワード＞ -bg
python3 -c "import subprocess, os; subprocess.Popen(['websockify', '--web', '/usr/share/novnc', '6080', 'localhost:5900'], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, preexec_fn=lambda: os.setsid())"
```

## まとめ

- **費用はすべて無料**（Windows/WSL2/Ubuntu/opencode/DeepSeek Free 全て無料）
- **WSL2 は opencode の公式推奨環境**（Windows 上の最良の体験）
- **GUI デスクトップはおまけ**（opencode はターミナル TUI で十分使える）
- **x11vnc + noVNC** が WSL2 上の GUI アクセスとしては最も安定している

AI エージェント環境をローカルに構築したい方は、ぜひ試してみてください。
