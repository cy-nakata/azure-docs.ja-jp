---
title: "Azure Site Recovery によるセカンダリ サイトへのレプリケーションのサポート マトリックス | Microsoft Docs"
description: "Azure Site Recovery でサポートされているオペレーティング システムとコンポーネントの概要について説明します"
services: site-recovery
documentationcenter: 
author: rayne-wiselman
manager: carmonm
editor: 
ms.assetid: 
ms.service: site-recovery
ms.devlang: na
ms.topic: article
ms.tgt_pltfrm: na
ms.workload: storage-backup-recovery
ms.date: 10/30/2017
ms.author: raynew
ms.openlocfilehash: da120d8e325867eaf9eb8b9be1ae8d9152db54c4
ms.sourcegitcommit: e38120a5575ed35ebe7dccd4daf8d5673534626c
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 11/13/2017
---
# <a name="support-matrix-for-replication-to-a-secondary-site-with-azure-site-recovery"></a>Azure Site Recovery によるセカンダリ サイトへのレプリケーションのサポート マトリックス

この記事では、[Azure Site Recovery](site-recovery-overview.md) サービスを使用してセカンダリのオンプレミス サイトにレプリケートする際のサポートについて説明します。

## <a name="supported-scenarios"></a>サポートされるシナリオ

**デプロイ** | **詳細** 
--- | ---
**VMware から VMware** | オンプレミスの VMware VM からセカンダリ VMware サイトへのディザスター リカバリー。<br/><br/> [InMage Scout ユーザー ガイド](https://aka.ms/asr-scout-user-guide) をダウンロードする
**Hyper-V から Hyper-V** | VMM クラウドのオンプレミスの Hyper-V VM のセカンダリ VMM クラウドへのディザスター リカバリー。<br></br> VMM なしではサポートされていません。



  

## <a name="host-servers"></a>ホスト サーバー

**デプロイ** | **サポート**
--- | ---
**VMware VM/物理サーバー** | vCenter 5.5、6.0、および 6.5 (5.5 の機能のみサポート)
**Hyper-V (VMM あり)** | Windows Server 2016、最新の更新プログラムが適用された Windows Server 2012 R2。<br/><br/> Windows Server 2016 ホストは VMM 2016 によって管理する必要があります。<br/><br/> Windows Server 2016 ホストと 2012 R2 ホストが混在する VMM 2016 クラウドは、現在サポートされていません。<br/><br/> 既存の VMM 2012 R2 から System Center 2016 へのアップグレードを含むデプロイは、現在サポートされていません。


## <a name="support-for-replicated-machine-os-versions"></a>レプリケートされるマシンの OS バージョンのサポート

次の表は、Site Recovery 使用時にレプリケートされるマシンのオペレーティング システムのサポートをまとめたものです。 すべてのワークロードは、サポートされるオペレーティング システム上で実行できます。

**VMware/物理サーバー** | **Hyper-V (VMM あり)**
--- | ---
64 ビット Windows Server 2016、Windows Server 2012 R2、Windows Server 2012、Windows Server 2008 R2 SP1 以降<br/><br/> Red Hat Enterprise Linux 6.7、6.8、6.9、7.1、7.2 <br/><br/> Centos 6.5、6.6、6.7、6.8、6.9、7.0、7.1、7.2 <br/><br/> Red Hat 互換カーネルまたは Unbreakable Enterprise Kernel リリース 3 (UEK3) を実行している Oracle Enterprise Linux 6.4、6.5、6.8 <br/><br/> SUSE Linux Enterprise Server 11 SP3、11 SP4  | [Hyper-V でサポートされている](https://technet.microsoft.com/library/mt126277.aspx)任意のゲスト オペレーティング システム

## <a name="linux-machine-storage"></a>Linux マシンのストレージ

次のストレージを使用する Linux マシンのみでリプリケート可能です。

- ファイル システム (EXT3、ETX4、ReiserFS、XFS)。
- マルチパス ソフトウェア: デバイス マッパー。
- ボリューム マネージャー (LVM2)。
- HP CCISS コントローラー ストレージを使用する物理サーバーはサポートされていません。
- ReiserFS ファイル システムは、SUSE Linux Enterprise Server 11 SP3 でのみサポートされています。

## <a name="network-configuration"></a>ネットワーク構成

### <a name="hosts"></a>ホスト

**構成** | **VMware/物理サーバー** | **Hyper-V (VMM あり)**
--- | --- | ---
NIC チーミング | あり | あり
VLAN | あり | あり
IPv4 | あり | あり
IPv6 | なし | なし

### <a name="guest-vms"></a>ゲスト VM

**構成** | **VMware/物理サーバー** | **Hyper-V (VMM あり)**
--- | --- | ---
NIC チーミング | なし | なし
IPv4 | あり | あり
IPv6 | なし | なし
静的 IP (Windows) | あり | あり
静的 IP (Linux) | あり | あり
マルチ NIC | あり | あり


## <a name="storage"></a>Storage

### <a name="host-storage"></a>ホスト ストレージ

**ストレージ (ホスト)** | **VMware/物理サーバー** | **Hyper-V (VMM あり)**
--- | --- | ---
NFS | あり | 該当なし
SMB 3.0 | 該当なし | あり
SAN (ISCSI) | あり | はい
マルチパス (MPIO) | あり | あり

### <a name="guest-or-physical-server-storage"></a>ゲストまたは物理サーバーのストレージ

**構成** | **VMware/物理サーバー** | **Hyper-V (VMM あり)**
--- | --- | ---
VMDK | あり | 該当なし
VHD/VHDX | 該当なし | あり (最大 16 個のディスク)
第 2 世代 VM | 該当なし | あり
共有クラスター ディスク | あり  | なし
暗号化されたディスク | なし | なし
UEFI| あり | 該当なし
NFS | なし | なし
SMB 3.0 | なし | なし
RDM | あり | 該当なし
1 TB より大きいディスク | あり | あり
ストライピングされたディスクのボリューム > 1 TB<br/><br/> LVM | あり | あり
記憶域 | なし | あり
ディスクのホット アド/削除 | あり | なし
ディスクの除外 | あり | あり
マルチパス (MPIO) | 該当なし | あり

## <a name="vaults"></a>資格情報コンテナー

**アクション** | **VMware/物理サーバー** | **Hyper-V (VMM あり)**
--- | --- | ---
リソース グループ間 (サブスクリプション内またはサブスクリプション間) での資格情報コンテナーの移動 | なし | なし
リソース グループ間 (サブスクリプション内またはサブスクリプション間) でのストレージ、ネットワーク、Azure VM の移動 | なし | なし

## <a name="provider-and-agent"></a>プロバイダーとエージェント

**名前** | **説明** | **最新バージョン** | **詳細**
--- | --- | --- | --- | ---
**Azure Site Recovery プロバイダー** | オンプレミスのサーバーと Azure の間の通信を調整します <br/><br/> オンプレミス VMM サーバー、または Hyper-V サーバー (VMM サーバーがない場合) にインストールされます | 5.1.19 ([ポータルから使用可能](http://aka.ms/downloaddra)) | [最新の機能と修正](https://support.microsoft.com/kb/3155002)
**モビリティ サービス** | オンプレミスの VMware サーバー/物理サーバーとセカンダリ サイトの間のレプリケーションを調整します<br/><br/> レプリケートする VMware VM または物理サーバーにインストールされます  | 該当なし (ポータルから使用可能) | 該当なし


## <a name="next-steps"></a>次のステップ

- [VMM クラウドの Hyper-V VM をセカンダリ サイトにレプリケートする](tutorial-vmm-to-vmm.md)
- [VMware VM と物理サーバーをセカンダリ サイトにレプリケートする](tutorial-vmware-to-vmware.md)
