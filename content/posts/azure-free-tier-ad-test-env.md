---
title: "Azure無料枠(2026年)でAD検証環境を3VM構築する方法"
date: 2026-06-15
draft: true
tags: ["Azure", "ActiveDirectory", "WEF", "HyperV", "検証環境"]
categories: ["infrastructure"]
summary: "2026年にアップデートされたAzure無料枠を使い、Active Directoryの検証環境（AD/DNSサーバ、WEC、クライアント端末の3VM構成）をほぼ無料で構築する方法を解説します。過去に課題だったRunCommand問題やVM台数制限も現在は解決されています。"
---

## 背景

Active Directory（AD）とWindows Event Forwarding（WEF）の検証環境を構築するには、最低3台のVMが必要です。

| VM | 役割 | 必要なソフトウェア |
|----|------|-------------------|
| VM1 | AD/DNSサーバ | Windows Server + AD DS ロール |
| VM2 | WECサーバ | Windows Server + Windows Event Collector ロール |
| VM3 | クライアント端末 | Windows 10/11 Pro（AD参加してイベントを発生） |

本番環境の前段として手軽に検証したいが、過去には2つの壁がありました。

1. **Azure無料枠ではVMが実質2台までしか作れなかった** → 3台目から課金が発生していた
2. **Azure RunCommandが安定して動作しない** → リモートからVMをセットアップする手段が限られる

2026年現在、これらの課題はどう変わったのか、調査した結果を報告します。

## 2026年のAzure無料枠——「3VM構成」の現実

2026年にAzure無料枠が大幅にアップデートされました。無料枠のVM提供は以下の2タイプが**公式のFree Servicesページで確認**できます。

| VMタイプ | アーキテクチャ | vCPU | RAM | 無料枠 |
|---------|--------------|------|-----|-------|
| B2pts v2 | Arm（Ampere） | 2 | 1GB | 750時間/月 |
| B2ats v2 | AMD | 2 | 1GB | 750時間/月 |

> **補足**: Windows VMの料金ページにはB2ts v2（Intel）も含めた3タイプの記載がありますが、Free Servicesページでは2タイプのみ確認できます。実際に無料で使えるかはサインアップ後のポータルでご確認ください。

> **RAMについて**: これらのVMは1GB RAMです。AD DSの最小要件は512MBなので動作はしますが、余裕を持たせたい場合は無料枠外のB2ls_v2（4GB/約¥2,400/月）の利用も検討してください。

各VMタイプに**750時間/月の独立した無料枠**が割り当てられます（「750 hours **each**」）。月30時間（平日1時間/日）の利用であれば、各VM枠の**4％**しか消費しません。

### 唯一の課金ポイント：Managed Disk

ただし注意点もあります。無料枠に含まれるManaged Diskは**2枚まで**（64GB P6 SSD）です。3台目のVMのOSディスクは有料となり、月額約700円（リージョンにより変動）がかかります。

## RunCommand問題——使わなくてもいい

従来のセットアップではAzure RunCommand（アクション実行）をよく使っていました。しかし、このRunCommandには以下の7つの制約があります。

1. 出力が最大4KB（超えると消失）
2. 最小実行時間が約20秒（短いスクリプトでも待たされる）
3. 同時実行不可（直列実行のみ）
4. 90分タイムアウト
5. キャンセル不可
6. VM AgentがNOT READYだと動作しない
7. Azure Public IPへの443番outboundが必須

これらの制約により、特に**6番目の「VM Agent NOT READY」**が頻発し、RunCommandがそもそも使えないケースが多発していました。

### 解決策：RunCommandを使わない

結論としては、RunCommandに頼らずとも以下の方法でリモートコマンドを実行できます。

- **同一VNet内** → WinRM over Private IP（ポート5985）
- **Hyper-V環境** → PowerShell Direct（ネットワーク不要）

Azure VM同士が同一VNetにいる場合、Private IP経由のWinRMが安定して動作します。初回接続のために以下のコマンドをVMのローカル管理者として実行しておきます。

```powershell
# WinRMを有効化
Enable-PSRemoting -Force

# TrustedHostsにすべてのホストを許可（検証環境限定）
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force

# ネットワークプロファイルをPrivateに変更
Set-NetConnectionProfile -NetworkCategory Private
```

## 3つの構成案比較

2026年現在、AD検証環境を構築する方法は主に3パターンあります。

### 案A：ローカルHyper-V完全構築（0円/月）

Windows Pro/EnterpriseエディションにはHyper-Vが標準搭載されています。Windows Serverの評価版（180日間無料）を使用すれば、ライセンス費用ゼロで3VMをすべてローカルに構築できます。

- **メリット**: 完全無料、レイテンシゼロ、オフラインでも作業可能
- **デメリット**: ローカルPCのリソース消費（RAM最低16GB、推奨32GB）
- **向いている人**: 開発用PCに32GB以上のRAMがある人

### 案B：Azure無料枠2VM構成＋ローカルVM（約0円〜700円/月）

Azure無料枠で2台（B2pts v2 + B2ats v2）、残り1台はローカルHyper-Vで運用する構成です。Azure側のManaged Disk2枚は無料枠に収まるため、実質0円で運用できます。どうしても3台ともAzureに置きたい場合は、2台は無料枠、3台目はB2ts v2（有料・約¥2,200/月）または同じく無料枠VMを時間をずらして使い回すなどの工夫が必要です。

- **メリット**: ローカルPC負荷を軽減、どこからでもアクセス可能
- **デメリット**: 無料枠だけでは3台同時運用が難しい
- **向いている人**: ノートPCなどリソースが限られている環境

### 案C：ハイブリッド構成（Hyper-V 2台 + Azure 1台）← ベスト

**結論から言うと、この構成が最もバランスが取れています。**

| VM | 配置 | 費用 |
|----|------|------|
| VM1（AD/DNS） | ローカルHyper-V | 0円 |
| VM2（WEC） | ローカルHyper-V | 0円 |
| VM3（クライアント） | Azure無料枠（B2pts v2 / B2ats v2） | 0円（ディスク代も無料枠2枚以内） |

3台目のManaged Disk代もかからないため、**月額0円**で3VM構成を実現できます。

VM3（クライアント）だけをクラウドに置くことで、社外からのドメイン参加テストや、異なるネットワーク経由のイベント転送テストも可能になります。

## リモートコマンド実行のベストプラクティス

各VMへのコマンド実行方法を整理します。

### Hyper-V VM → PowerShell Direct（最速・最安定）

Hyper-VのVMに対してはPowerShell Directが最も高速かつ安定しています。ネットワークスタックを経由しないため、ファイアウォールやDNSの設定が不完全でも動作します。

```powershell
# VMにセッション接続
Enter-PSSession -VMName "AD-Server" -Credential (Get-Credential)

# スクリプトブロックを実行
Invoke-Command -VMName "AD-Server" -ScriptBlock {
    Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
}
```

### Azure VM → WinRM over Private IP

Azure VMの場合は同一VNet内のPrivate IP経由でWinRM接続します。Public IPを必要としないためセキュリティリスクも低減できます。

```powershell
$secPass = ConvertTo-SecureString "YourPassword" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential ("Administrator", $secPass)

Invoke-Command -ComputerName "10.0.0.5" -Credential $cred -ScriptBlock {
    # ADドメインに参加
    Add-Computer -DomainName "contoso.local" -Credential $cred -Restart
}
```

### どうしてもRunCommandが必要な場合

既存のスクリプトとの互換性など、どうしてもRunCommandを使う必要がある場合は、**Managed RunCommand**（改善版）の使用を推奨します。従来のAction RunCommandに比べ、出力サイズ制限やタイムアウト周りが改善されています。

```azurecli
az vm run-command create \
  --resource-group myRG \
  --vm-name myVM \
  --name myRunCmd \
  --script "echo hello" \
  --location japaneast
```

## まとめ

2026年のAzure無料枠アップデートにより、AD検証環境の構築は以前より格段に容易になりました。

| 課題 | 2026年の状況 |
|------|------------|
| VM台数制限 | 2VMは無料枠でOK。残り1台はHyper-V or 有料VM |
| RunCommand問題 | WinRM / PowerShell Directで完全回避可能 |
| コスト | ハイブリッド構成なら0円/月〜700円/月 |

**最適解は案C（Hyper-V 2台 + Azure無料枠1台）**です。ローカルPCのリソースに余裕があれば完全ローカル構築（案A）も有力な選択肢です。

この記事で紹介したPowerShellのワンライナーを活用すれば、GUI操作をほとんど行わずに3VM構成をセットアップできます。検証環境の構築にお役立てください。
