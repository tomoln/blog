---
title: "Oracle Cloud無料枠でOpenCode環境を作る"
slug: "oracle-cloud-free-tier-opencode"
date: 2026-06-16
draft: true
categories: ["tech"]
tags: ["OracleCloud", "FreeTier", "OpenCode", "クラウドVM", "体験記", "Hetzner"]
summary: "Oracle Cloud Free Tierの2コア/12GB無料枠に魅了され、OpenCode＋AIエージェント用のVM構築に挑戦した記録。OCI CLIの壁、Python SDKでの手動構築、そして「Out of host capacity」の悲劇までを時系列で綴ります。"
---

## 背景

私は現在、OpenCode（旧Continue）を使ったAIエージェント開発を趣味で行っています。ローカルのWindowsマシン（i5-1335U / 16GB RAM）でも十分動作するのですが、以下のような理由から**常時稼働のクラウドVM**が欲しくなりました。

- 長期間のバッチ処理的なエージェント実行を任せたい
- 外出先（iPadなど）からSSHで接続して作業を続けたい
- ローカルマシンのリソースを圧迫せずに済ませたい

そこで「安くてそこそこのスペックがあるVPS」を探し始めました。さくらのVPS、Hetzner、AWS Lightsail…と見ているうちに、ある噂を耳にします。

**「Oracle Cloud Free Tierが最強」** と。

## Oracle Cloud Free Tier との出会い

調べてみると、Oracle Cloud Free Tierは以下のスペックが**完全無料**で使えるとのこと。

| リソース | 内容 |
|----------|------|
| AMD VM | 1 OCPU / 1GB RAM × 2 インスタンス |
| ARM VM（Ampere A1） | **最大 2 OCPU / 12GB RAM** |
| ブロックストレージ | 合計 200GB |
| ロードバランサー | 10Mbps |
| オブジェクトストレージ | 合計20GB |

特に注目すべきは **Ampere A1.Flex** です。ARMアーキテクチャのCPUながら、2コア・12GB RAM（1,500 OCPU時間/月相当）が無料。同スペックを他社で借りると月2,000円〜3,000円することを考えると、破格の条件です。

「これだ」と思いました。すぐにUS Ashburnリージョンでアカウントを作成し、VM構築に取り掛かりました。

## アカウント作成とSSH鍵の生成

サインアップ自体はスムーズでした。クレジットカード情報の入力（本人確認のため）が必要ですが、実際に課金はされません。リージョンはUS East (Ashburn) を選択。日本の大阪リージョンもあるのですが、ARMインスタンスの空きが多いという噂を信じてAshburnにしました。

まずはSSH鍵を生成します。

```bash
ssh-keygen -t ed25519 -C "opencode-vm" -f ~/.ssh/oracle_opencode
```

ed25519を選んだのは、セキュリティ強度が高く鍵長が短いため。OCIコンソールに公開鍵を登録する準備も整いました。

ここまでは順調でした。

## OCI CLI の壁

VM作成にはOCI CLI（Oracle Cloud Infrastructure Command Line Interface）を使うのが効率的です。しかし、ここで最初の壁にぶつかります。

### winget でのインストール（失敗）

Windowsなので、まずはwingetでインストールを試みました。

```powershell
winget install Oracle.OracleCloudInfrastructureCLI
```

インストールは成功したように見えたのですが、**コマンドプロンプトを再起動しても `oci` コマンドが認識されません**。

調べてみると、OCI CLIのWindowsインストーラーはPATHの設定が正しく行われない場合があるようです。手動でPATHに追加しようとしましたが、どこにインストールされたのか特定するのに手間取りました。

### pip でのインストール（失敗）

次に、Pythonのpipでインストールを試みました。

```powershell
pip install oci-cli
```

すると、今度は **「パスが長すぎます」** エラー。WindowsのMAX_PATH制限（260文字）に引っかかりました。Pythonの仮想環境を使えば回避できたかもしれませんが、この時点で「OCI CLI、Windowsでは相性が悪いな」と感じ始めました。

**「ならばPythonのOCI SDKだけで全てやろう」** — そう決断しました。

## Python OCI SDK でVM作成に挑戦

OCI CLIを諦め、Pythonの `oci` SDK（Software Development Kit）だけを使ってVMを作成することにしました。

SDKのインストールはpip一発で完了しました。

```powershell
pip install oci
```

ここからが本番です。VMを作成するために必要なネットワークリソースを**すべて手動でコードに書いて作成**します。具体的には以下の6つが必要です。

1. **VCN**（Virtual Cloud Network / 仮想クラウドネットワーク）
2. **Internet Gateway**（インターネットへの出入り口）
3. **Route Table**（ルーティングの設定表）
4. **Security List**（ファイアウォールのルール）
5. **Subnet**（サブネット）
6. **Compute Instance**（VM本体）

それぞれに対してOCI SDKのAPIを呼び出していきます。コードの一部を紹介します。

```python
import oci

config = oci.config.from_file("~/.oci/config")
compute_client = oci.core.ComputeClient(config)
network_client = oci.core.VirtualNetworkClient(config)

# VCNの作成
vcn = network_client.create_vcn(
    oci.core.models.CreateVcnDetails(
        compartment_id=COMPARTMENT_ID,
        display_name="opencode-vcn",
        cidr_block="10.0.0.0/16"
    )
).data

# （Internet Gateway, Route Table, Security List, Subnetも同様に作成）
```

OSイメージは **Ubuntu 24.04 LTS (ARM64)** を指定。シェイプは **VM.Standard.A1.Flex** で **2 OCPU / 12GB RAM** に設定しました。

いよいよインスタンス作成のAPIを叩きます。

## 「Out of host capacity」の悲劇

APIのレスポンスは一瞬で返ってきました。しかし、その内容は期待とは程遠いものでした。

```
oci.exceptions.ServiceError: {
    'status': 500,
    'code': 'OutOfHostCapacity',
    'message': 'The host capacity is currently unavailable for the requested resources in the specified availability domain.'
}
```

**「Out of host capacity」** — つまり空きサーバーがない、というエラーです。

US Ashburnリージョンには3つのAvailability Domain（可用性ドメイン / 物理的に分離されたデータセンター）があります。すべてのADに対して順番に試しましたが、**3つとも空きなし**。

「スペックを下げれば空くかもしれない」と考え、2 OCPU / 12GB → 1 OCPU / 6GB に落として再試行。それでもダメでした。

ここで噂の真実を実感します。**Oracle Cloud Free TierのARMインスタンスは、人気リージョンでは常に取り合いの状態** なのです。特にUS Ashburnは需要が集中しており、無料枠でVMを確保するのは至難の業とのこと。

## PAYGアップグレードの壁（断念）

ネットで調べると「PAYG（Pay As You Go / 従量課金）にアップグレードすると空きが出やすくなる」という情報がありました。有料アカウントになると、無料ユーザーより優先的にリソースが割り当てられるようです。

コンソールのアップグレードボタンを探すと、確かにありました。しかし **灰色でクリックできない**。

要件を確認すると、PAYGアップグレードには**支払い方法の追加登録（クレジットカードの再検証）** が必要なケースがあるとのこと。サインアップ時に登録したカード情報だけでは不十分で、追加の手続きが必要な場合があるようです。

ここで私は「無料にこだわるのはやめよう」と冷静になりました。

「ただより高いものはない」とはよく言ったものです。無料を追い求めて数時間を溶かすよりも、確実に動く環境を月額1,000円程度で借りたほうが、トータルコスト（時間的にも金銭的にも）は安いのです。

## 学び・補足

### Oracle Cloud Free Tier の現実

- **スペックは確かに最強**：2コア/12GB RAM（1,500 OCPU時間/月相当）が無料なのは他に類を見ません
- **空きが慢性的に不足**：特にUS Ashburn・US Phoenixなどの人気リージョンでは、ARMインスタンスの空きは宝くじのようなもの
- **リージョン選びが最重要**：日本リージョン（大阪）のほうが空きがあるという情報もあります。ただし試していないので確証はありません
- **PAYGアップグレードは手間**：有料アカウントにすれば空きが出やすくなりますが、支払い方法の追加登録などが必要

### 最終的に選んだ選択肢：Hetzner

結局私は **Hetzner**（ドイツのVPSプロバイダ）に注目しました。スペックは以下の通り。

| 項目 | 内容 |
|------|------|
| プラン | CAX11（ARM / 2 vCPU / 4GB RAM） |
| 料金 | €6.49/月（約1,040円、税別） |
| OS | Ubuntu 24.04 |
| 回線 | 1Gbps |

OpenCodeを動かすには十分すぎるスペックです。セットアップはSSH接続してから **わずか15分** で完了します。

```bash
# HetznerのUbuntu VMでやること（全部）
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs git
npm install -g opencode
```

OCI CLIとの格闘に数時間を費やした後なら、このシンプルさに感動すら覚えるでしょう。

### まとめ

| 方法 | コスト | 確実性 | セットアップ時間 | 総合評価 |
|------|--------|--------|-----------------|---------|
| Oracle Cloud Free Tier | 無料 | ★☆☆☆☆ | 数時間〜（空き次第） | △ |
| Oracle Cloud PAYG | 使いすぎ注意 | ★★★★☆ | 1〜2時間 | ○ |
| Hetzner | €6.49/月 | ★★★★★ | 15分 | ◎ |

**Oracle Cloud Free Tierは最強の無料サービスですが、ARMインスタンスの空きの問題は深刻です。**

もし「絶対に無料で通したい」という強い意志があるなら、以下を試す価値はあります。

1. 日本リージョン（大阪）でアカウントを作り直す
2. 深夜など空きが出やすい時間帯を狙う
3. スクリプトで空きを監視して自動作成する（OCI SDKなら可能）

ただし私のように「確実に動く環境が欲しいだけ」なら、HetznerやさくらのVPSなどの **格安有料VPS** をおすすめします。月1,000円程度で、面倒な空き待ちから解放されるなら安いものです。

「ただより高いものはない」—— 今回の経験は、この言葉を身を以て理解する良い機会になりました。
