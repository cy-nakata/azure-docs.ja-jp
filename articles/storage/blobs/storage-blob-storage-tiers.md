---
title: "BLOB の Azure ホット、クール、およびアーカイブ ストレージ | Microsoft Docs"
description: "Azure BLOB ストレージ アカウントのためのホット、クール、およびアーカイブ ストレージ。"
services: storage
documentationcenter: 
author: michaelhauss
manager: vamshik
editor: tysonn
ms.assetid: eb33ed4f-1b17-4fd6-82e2-8d5372800eef
ms.service: storage
ms.workload: storage
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: get-started-article
ms.date: 06/05/2017
ms.author: mihauss
ms.openlocfilehash: 501fc59efb8bacf58fea2825752d3a33c6ea5963
ms.sourcegitcommit: b854df4fc66c73ba1dd141740a2b348de3e1e028
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 12/04/2017
---
# <a name="azure-blob-storage-hot-cool-and-archive-preview-storage-tiers"></a>Azure Blob Storage: ホット、クール、およびアーカイブ (プレビュー) ストレージ層

## <a name="overview"></a>概要

Azure Storage では、BLOB オブジェクト ストレージ用に 3 つのストレージ層が提供されています。そのため、使い方しだいで、コスト効率の高い方法でデータを格納することができます。 Azure **ホット ストレージ層**は、頻繁にアクセスされるデータの格納に適しています。 Azure **クール ストレージ層**は、アクセスされる頻度は低いものの、少なくとも 30 日以上保管されるデータの格納に適しています。 Azure **アーカイブ ストレージ層** (プレビュー) は、ほとんどアクセスされず、少なくとも 180 日以上保管され、待ち時間の要件が柔軟 (数時間単位) であるデータの格納に適しています。 アーカイブ ストレージ層は、BLOB レベルのみで使用することができ、ストレージ アカウント レベルで使用することはできません。 クール ストレージ層に格納されるデータについては、可用性が若干低くても許容できますが、高い持続性は必要で、アクセスにかかる時間とスループット特性もホット データと同程度である必要があります。 ホット データと比較して可用性の SLA が若干低く、アクセス コストが高めのクール データは、ストレージ コストが低ければ許容できます。 アーカイブ ストレージはオフラインであり、ストレージ コストは最も低くなりますが、アクセス コストが最も高くなります。

最近では、クラウドに格納されるデータが急激に増加しています。 ストレージのニーズが拡大する中でコストを管理するには、アクセスの頻度や予定保有期間などの属性に基づいてデータを整理し、コストを最適化する方法が効果的です。 クラウドに格納されるデータは、有効期間を通じてどのように生成、処理、アクセスされるかという点で異なる場合があります。 有効期間を通じて活発にアクセスおよび変更されるデータもあれば、 有効期間の初期に頻繁にアクセスされ、古くなるにつれて大幅にアクセスが減るデータもあります。 また、クラウド内でアイドル状態のままとなり、格納されてからはほとんどアクセスされないデータもあります。

各データ アクセス シナリオは、特定のアクセス パターン用に最適化された異なるストレージ層の利点を利用しています。 Azure Blob Storage は、ホット、クール、およびアーカイブの各ストレージ層によって、分化されたストレージ層のニーズに異なる価格モデルで対応しています。

## <a name="blob-storage-accounts"></a>BLOB ストレージ アカウント

**BLOB ストレージ アカウント**とは、Azure Storage に BLOB (オブジェクト) として非構造化データを格納するための特殊なストレージ アカウントです。 BLOB ストレージ アカウントでは、アクセス パターンに基づいて、アカウント レベルでホット ストレージ層とクール ストレージ層を選択でき、BLOB レベルでホット層、クール層、およびアーカイブ層を選択できるようになりました。 ほとんどアクセスしないデータ、アクセス頻度の低いデータ、頻繁にアクセスするデータのそれぞれを、アーカイブ、クール、ホットの各ストレージ層に格納してコストを最適化します。 BLOB ストレージ アカウントは、既存の汎用ストレージ アカウントと同様に、現在使用されているすべての優れた耐久性、可用性、スケーラビリティ、およびパフォーマンス機能を共有します。たとえば、ブロック BLOB と追加 BLOB の 100% の API 整合性などです。

> [!NOTE]
> BLOB ストレージ アカウントは、ブロック BLOB と追加 BLOB のみをサポートします。ページ BLOB はサポートしません。

BLOB ストレージ アカウントは、**[アクセス レベル]** 属性をアカウント レベルで公開します。この属性では、既定のストレージ アカウント層を**ホット**または**クール**として指定できます。 既定のストレージ アカウント層は、BLOB レベルで層が明示的に設定されていない BLOB に適用されます。 データの使用パターンが変化した場合は、いつでもこれらのストレージ層を切り替えることができます。 **アーカイブ層** (プレビュー) は、BLOB レベルでのみ適用できます。

> [!NOTE]
> ストレージ層を変更すると、追加料金が発生することがあります。 詳細については、「[価格と課金](#pricing-and-billing)」セクションを参照してください。

### <a name="hot-access-tier"></a>ホット アクセス層

ホット ストレージは、クール ストレージとアーカイブ ストレージに比べてストレージ コストが高くなるものの、アクセス コストが最も低くなります。 ホット ストレージ層の使用シナリオの例には、次のようなものがあります。

* 活発に使用されたり、頻繁にアクセス (読み取りまたは書き込み) されると予想されるデータ。
* 処理段階にあり、最終的にはクール ストレージ層に移行されるデータ。

### <a name="cool-access-tier"></a>クール アクセス層

クール ストレージ層は、ホット ストレージに比べてストレージ コストが低くなり、アクセス コストが高くなります。 この層は、少なくとも 30 日以上クール層で保持されるデータを対象としています。 クール ストレージ層の使用シナリオの例には、次のようなものがあります。

* 短期的なバックアップおよびディザスター リカバリーのデータセット。
* 既に頻繁に参照されなくなっているものの、アクセスされたときにはすぐに利用できることが期待されている、古いメディア コンテンツ。
* 今後処理するためにさらにデータが収集されている場合に、コスト効率の高い方法で格納する必要がある、大規模なデータ セット  ("*たとえば*"、長期保存する科学データや、製造施設からの未加工のテレメトリ データ)。

### <a name="archive-access-tier-preview"></a>アーカイブ アクセス層 (プレビュー)

アーカイブ ストレージは、ストレージ コストが最も低く、ホット ストレージとクール ストレージに比べてデータ取得コストが高くなります。 この層は、数時間の取得待ち時間が許容され、少なくとも 180 日以上アーカイブ層に保持されるデータを対象としています。

BLOB はアーカイブ ストレージにありますが、オフラインになっているため、読み取り (オンラインであり利用可能なメタデータを除く)、コピー、上書き、または変更を行うことはできません。 また、アーカイブ ストレージ内の BLOB のスナップショットを作成することもできません。 ただし、既存の操作を使用して、BLOB プロパティ/メタデータの削除、一覧表示、取得を行ったり、BLOB の層を変更したりすることはできます。

#### <a name="blob-rehydration"></a>BLOB のリハイドレート
アーカイブ ストレージ内のデータを読み取るには、まず、BLOB の層をホットまたはクールに変更する必要があります。 このプロセスはリハイドレートと呼ばれ、50 GB 未満の BLOB の処理に最大で 15 時間かかることがあります。 より大きな BLOB に必要な時間は、BLOB のスループットの上限に応じて異なります。

リハイドレート中に、層が変更されたかどうかを確認するために "アーカイブ ステータス" BLOB プロパティをチェックすることができます。 ステータスは、変更先の層に応じて "rehydrate-pending-to-hot" または "rehydrate-pending-to-cool" になります。 完了すると、"アーカイブ ステータス" BLOB プロパティが削除され、"アクセス レベル" BLOB プロパティがホット層またはクール層を表します。  

アーカイブ ストレージ層の使用シナリオの例には、次のようなものがあります。

* 長期的なバックアップ、アーカイブ、およびディザスター リカバリーのデータセット。
* 最終的に使用可能な形式に処理した後も保持する必要がある、元の (未加工の) データ  ("*たとえば*"、他の形式にコード変換した後の未加工のメディア ファイル)。
* 長期間格納しておく必要があり、ほとんどアクセスされることがない、コンプライアンスおよびアーカイブ データ  ("*たとえば*"、監視カメラ映像、医療機関の古い X 線/MRI 画像、金融サービスの顧客電話の音声録音や会話記録)。

### <a name="recommendations"></a>Recommendations

ストレージ アカウントの詳細については、「 [Azure ストレージ アカウントについて](../common/storage-create-storage-account.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json) 」を参照してください。

ブロックまたは追加 Blob Storage だけを必要とするアプリケーションでは、階層化されたストレージの分化された料金モデルの利点を活用するために、BLOB ストレージ アカウントを使用することをお勧めします。 ただし、状況によってはそのようにできず、汎用ストレージ アカウントを使用するしかない場合もあります。たとえば、次のような場合です。

* テーブル、キュー、またはファイルを使用する必要があり、BLOB を同じストレージ アカウントに格納したい。 同じ共有キーを持つこと以外に、これらを同じアカウント内に格納する技術的な利点はありません。

* まだクラシック デプロイ モデルを使用する必要がある。 BLOB ストレージ アカウントは、Azure Resource Manager デプロイメント モデルだけで利用できます。

* ページ BLOB を使用する必要がある。 BLOB ストレージ アカウントはページ BLOB をサポートしていません。 一般に、ページ BLOB が必要な特別な理由がなければ、ブロック BLOB を使用することをお勧めします。

* [Storage Services REST API](https://msdn.microsoft.com/library/azure/dd894041.aspx) の 2014-02-14 より前のバージョンか、クライアント ライブラリの 4.x より前のバージョンを使用していて、アプリケーションをアップグレードできない。

> [!NOTE]
> BLOB ストレージ アカウントは、現在、すべての Azure リージョンでサポートされています。


## <a name="blob-level-tiering-feature-preview"></a>BLOB レベルの層操作機能 (プレビュー)

BLOB レベルの階層制御では、[Set Blob Tier](/rest/api/storageservices/set-blob-tier) と呼ばれる 1 つの操作を使用して、オブジェクト レベルでデータの層を変更できます。 使用パターンの変化に応じて、アカウント間でデータを移動することなく、BLOB のアクセス層をホット、クール、またはアーカイブに簡単に変更することができます。 BLOB がアーカイブからリハイドレートされている場合を除き、すべての層変更は直ちに行われます。 BLOB 層が最後に変更された時間は、BLOB プロパティの**アクセス層変更時間**属性を介して公開されます。 BLOB がアーカイブ層にある場合、BLOB は上書きされない可能性があります。したがって、このシナリオでは、同じ BLOB をアップロードすることができません。 ホット層とクール層にある BLOB は上書きすることができます。この場合、新しい BLOB は、上書きされた古い BLOB の層を継承します。

3 つすべてのストレージ層内の BLOB は、同じアカウント内で共存できます。 層が明示的に割り当てられていない BLOB では、アカウントのアクセス層の設定から層が推定されます。 アクセス層がアカウントから推定された場合、**アクセス層の推定**属性が "true" に設定され、BLOB の **アクセス層**属性がアカウント層に一致します。 Azure Portal では、アクセス層の推定プロパティが BLOB アクセス層と共に表示されます (たとえば、"ホット (推定)" または "クール (推定)")。

> [!NOTE]
> アーカイブ ストレージと BLOB レベルの階層制御では、ブロック BLOB のみがサポートされます。 また、スナップショットがあるブロック BLOB の層を変更することもできません。

### <a name="blob-level-tiering-billing"></a>BLOB レベルの階層制御の課金

BLOB をよりクールな層 (ホットからクール、ホットからアーカイブ、またはクールからアーカイブ) に移動するとき、この操作は移動先の層への書き込みとして課金され、移動先の層の書き込み操作 (10,000 件単位) およびデータ書き込み (GB 単位) の料金が適用されます。 BLOB をよりホットな層 (アーカイブからクール、アーカイブからホット、またはクールからホット) に移動するとき、この操作は移動元の層からの読み取りとして課金され、移動元の層の読み取り操作 (10,000 件単位) およびデータ取得 (GB 単位) の料金が適用されます。

プレビューでこれらの機能を使用するには、[Azure アーカイブおよび BLOB レベルの層操作に関するブログ](https://azure.microsoft.com/blog/announcing-the-public-preview-of-azure-archive-blob-storage-and-blob-level-tiering)の手順に従ってください。

以下に、BLOB レベルの層操作に対してプレビュー中に適用されるいくつかの制限事項を示します。

* プレビューの登録が成功した後、米国東部 2、米国東部、および米国西部で作成された新しい BLOB ストレージ アカウントのみが、アーカイブ ストレージをサポートします。

* プレビューの登録が成功した後、パブリック リージョンで作成された新しい BLOB ストレージ アカウントのみで、BLOB レベルの層操作がサポートされます。

* BLOB レベルの層操作とアーカイブ ストレージは、[LRS] (../common/storage-redundancy.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json#locally-redundant- storage) ストレージのみでサポートされています。 [GRS](../common/storage-redundancy.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json#geo-redundant-storage) および [RA-GRS](../common/storage-redundancy.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json#read-access-geo-redundant-storage) は、今後サポートされる予定です。

* スナップショットを備えた BLOB の層を変更することはできません。

* アーカイブ ストレージ内の BLOB をコピーしたり、そのスナップショットを作成したりすることはできません。

## <a name="comparison-of-the-storage-tiers"></a>ストレージ層の比較

次の表は、ホット ストレージ層とクール ストレージ層の比較を示しています。 アーカイブ BLOB レベル層はプレビューであるため、SLA がありません。

| | **ホット ストレージ層** | **クール ストレージ層** | **アーカイブ ストレージ層**
| ---- | ----- | ----- | ----- |
| **可用性** | 99.9% | 99% | 該当なし |
| **可用性** <br> **(RA-GRS 読み取り)**| 99.99% | 99.9% | 該当なし |
| **利用料金** | より高いストレージ コスト、より低いアクセスおよびトランザクション コスト | より低いストレージ コスト、より高いアクセスおよびトランザクション コスト | 最も低いストレージ コスト、最も高いアクセスおよびトランザクション コスト |
| **最小オブジェクト サイズ** | 該当なし | 該当なし | 該当なし |
| **最小ストレージ存続期間** | 該当なし | 該当なし | 180 日
| **待機時間** <br> **(1 バイト目にかかる時間)** | ミリ秒 | ミリ秒 | 15 時間未満
| **スケーラビリティとパフォーマンスのターゲット** | 汎用ストレージ アカウントと同じ | 汎用ストレージ アカウントと同じ | 汎用ストレージ アカウントと同じ |

> [!NOTE]
> BLOB ストレージ アカウントは、汎用ストレージ アカウントと同じパフォーマンスとスケーラビリティ ターゲットをサポートしています。 詳細については、「 [Azure Storage のスケーラビリティおよびパフォーマンスのターゲット](../common/storage-scalability-targets.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json) 」を参照してください。


## <a name="pricing-and-billing"></a>価格と課金
BLOB ストレージ アカウントでは、各 BLOB の層に基づいた BLOB ストレージの価格モデルを採用しています。 BLOB ストレージ アカウントを使用するときには、課金に関して次の点を考慮してください。

* **ストレージ コスト**: データの格納のコストは、格納されているデータの量だけでなく、ストレージ層にも左右されます。 よりクールな層になるほど、ギガバイトあたりのコストが下がります。

* **データ アクセス コスト**: データ アクセス料金は、よりクールな層になるほど高くなります。 クールおよびアーカイブ ストレージ層のデータの場合、読み取りに対して、ギガバイト単位のデータ アクセス料金が課金されます。

* **トランザクション コスト**: すべての層に対してトランザクションごとに課金されます。よりクールな層になるほどコストが高くなります。

* **geo レプリケーション データ転送コスト**: GRS と RA-GRS を含む geo レプリケーションが構成されているアカウントだけに適用されます。 geo レプリケーション データ転送には、ギガバイトあたりの料金がかかります。

* **送信データ転送コスト**: 送信データ転送 (Azure リージョン外に転送されるデータ) では、帯域幅使用量に対する課金がギガバイトあたりで発生します。これは、汎用ストレージ アカウントと同じです。

* **ストレージ層の変更**: アカウント ストレージ層をクールからホットに変更すると、ストレージ アカウントに存在するすべてのデータの読み取りと同等の課金が発生します。 ただし、アカウント ストレージ層をホットからクールに変更したときは、クール層への全データの書き込みに相当する課金が発生します。

> [!NOTE]
> BLOB ストレージ アカウントの価格モデルの詳細については、「[Azure Storage Pricing](https://azure.microsoft.com/pricing/details/storage/)」を参照してください。 送信データ転送の価格の詳細については、[データ転送の料金詳細](https://azure.microsoft.com/pricing/details/data-transfers/)に関するページを参照してください。

## <a name="quick-start"></a>クイック スタート

このセクションでは、Azure Portal を使用して、次のシナリオについて説明します。

* BLOB ストレージ アカウントの作成方法。
* BLOB ストレージ アカウントの管理方法。

以下の例では、設定がストレージ アカウント全体に適用されるため、アクセス層をアーカイブに設定することはできません。 アーカイブは、特定の BLOB に対してのみ設定できます。

### <a name="create-a-blob-storage-account-using-the-azure-portal"></a>Azure ポータルを使用した BLOB ストレージ アカウントの作成

1. [Azure ポータル](https://portal.azure.com)にサインインします。

2. ハブ メニューで、**[新規]** > **[データ + ストレージ]** > **[ストレージ アカウント]** をクリックします。

3. ストレージ アカウントの名前を入力します。

    この名前は、グローバルに一意である必要があります。この名前は、ストレージ アカウントのオブジェクトにアクセスするための URL の一部として使用されます。  

4. デプロイ モデルとして **[Resource Manager]** を選択します。

    階層型ストレージは、Resource Manager ストレージ アカウントでのみ使用できます。これは、新しいリソースに推奨されるデプロイ モデルです。 詳細については、「[Azure Resource Manager の概要](../../azure-resource-manager/resource-group-overview.md)」を参照してください。  

5. [Account Kind (アカウントの種類)] ボックスの一覧の **[Blob Storage (Blob Storage)]**を選択します。

    ここで、ストレージ アカウントの種類を選択します。 階層型ストレージは汎用ストレージでは利用できません。種類が Blob Storage のアカウントでのみ利用できます。     

    これを選択すると、パフォーマンス レベルが Standard に設定されます。 階層型ストレージは、Premium パフォーマンス レベルでは利用できません。

6. ストレージ アカウントのレプリケーション オプション (**[LRS]**、**[GRS]**、または **[RA-GRS]**) を選択します。 既定値は **[RA-GRS]**です。

    LRS はローカル冗長ストレージ、GRS は geo 冗長ストレージ (2 つのリージョン)、RA-GRS は読み取りアクセス geo 冗長ストレージ (2 つのリージョン。セカンダリに対する読み取りアクセス権が付与される) です。

    Azure Storage のレプリケーション オプションの詳細については、 [Azure Storage のレプリケーション](../common/storage-redundancy.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json)に関するページを参照してください。

7. ニーズに応じた適切なストレージ層を選択します。**[アクセス レベル]** を **[クール]** と **[ホット]** のいずれかに設定します。 既定値は **[ホット]**です。

8. 新しいストレージ アカウントを作成するサブスクリプションを選択します。

9. 新しいリソース グループを指定するか、既定のリソース グループを選択します。 リソース グループの詳細については、「[Azure Resource Manager の概要](../../azure-resource-manager/resource-group-overview.md)」をご覧ください。

10. ストレージ アカウントのリージョンを選択します。

11. **[作成]** をクリックしてストレージ アカウントを作成します。

### <a name="change-the-storage-tier-of-a-blob-storage-account-using-the-azure-portal"></a>Azure ポータルを使用した BLOB ストレージ アカウントのストレージ層の変更

1. [Azure ポータル](https://portal.azure.com)にサインインします。

2. ストレージ アカウントに移動するには、[すべてのリソース] を選択し、ストレージ アカウントを選択します。

3. [設定] ブレードで **[構成]** をクリックし、アカウント構成を表示または変更します。

4. ニーズに応じた適切なストレージ層を選択します。**[アクセス レベル]** を **[クール]** と **[ホット]** のいずれかに設定します。

5. ブレードの上部にある [保存] をクリックします。

### <a name="change-the-storage-tier-of-a-blob-using-the-azure-portal"></a>Azure Portal を使用した BLOB のストレージ層の変更

1. [Azure ポータル](https://portal.azure.com)にサインインします。

2. ストレージ アカウントの BLOB に移動するには、[すべてのリソース]、ストレージ アカウント、コンテナー、BLOB を順に選択します。

3. BLOB のプロパティ ブレードで、**[アクセス レベル]** ドロップダウン メニューをクリックし、**[ホット]**、**[クール]**、または **[アーカイブ]** ストレージ層を選択します。

5. ブレードの上部にある [保存] をクリックします。

> [!NOTE]
> ストレージ層を変更すると、追加料金が発生することがあります。 詳細については、「[価格と課金](#pricing-and-billing)」セクションを参照してください。


## <a name="evaluating-and-migrating-to-blob-storage-accounts"></a>BLOB ストレージ アカウントの評価と移行
このセクションの目的は、ユーザーが BLOB ストレージ アカウントの使用にスムーズに移行できるようにお手伝いすることです。 2 つのユーザー シナリオがあります。

* 汎用ストレージ アカウントを既に持っており、適切なストレージ層の BLOB ストレージ アカウントへの変更を評価したい。
* BLOB ストレージ アカウントの使用を決定しているか、既に所有しており、ホット ストレージ層とクール ストレージ層のどちらを使用すべきかを評価したい。

いずれの場合も、最初にすべきことは、BLOB ストレージ アカウントに格納されたデータの格納とアクセスにかかるコストを見積もり、そのコストを現在のコストと比較することです。

## <a name="evaluating-blob-storage-account-tiers"></a>BLOB ストレージ アカウント レベルの評価

BLOB ストレージ アカウントに格納されたデータの格納とアクセスにかかるコストを見積もるためには、既存の使用パターンを評価するか、予想される使用パターンを概算する必要があります。 一般に、以下のことを調べる必要があります。

* ストレージの使用量 - どのくらいの量のデータが格納され、その量は月ごとにどのように変化するか。

* ストレージ アクセス パターン - アカウントに対してどのくらいの量のデータの読み取りや書き込みがあるか (新規データも含めて)。 データ アクセスにはどのくらいのトランザクションが使用されるか。また、そのトランザクションの種類は何か。

## <a name="monitoring-existing-storage-accounts"></a>既存のストレージ アカウントの監視

既存のストレージ アカウントを監視し、そのデータを収集するには、Azure Storage Analytics を利用できます。これにより、ログ記録が実行され、ストレージ アカウントのメトリック データが得られます。 Storage Analytics では、汎用ストレージ アカウントと BLOB ストレージ アカウントの両方について、Blob Storage サービスへの要求に関して集計されたトランザクション統計情報と容量データを含むメトリックを格納できます。 このデータは、同じストレージ アカウント内の既知のテーブルに格納されます。

詳細については、「[About Storage Analytics Metrics (Storage Analytics Metrics について)](https://msdn.microsoft.com/library/azure/hh343258.aspx)」と「[Storage Analytics Metrics Table Schema (Storage Analytics Metrics のテーブル スキーマ)](https://msdn.microsoft.com/library/azure/hh343264.aspx)」を参照してください。

> [!NOTE]
> BLOB ストレージ アカウントは、そのアカウントのメトリック データの格納とアクセスのためだけに Table サービス エンドポイントを公開します。

Blob Storage サービスのストレージ使用量を監視するには、容量メトリックを有効にする必要があります。
これを有効にすると、ストレージ アカウントの Blob service に関する容量データが毎日記録されます。これは、同じストレージ アカウント内の *$MetricsCapacityBlob* テーブルに書き込まれるテーブル エントリとして記録されます。

Blob Storage サービスのデータ アクセス パターンを監視するには、API レベルで時間単位のトランザクション メトリックを有効にする必要があります。 これを有効にすると、API あたりのトランザクションが 1 時間ごとに集計され、同じストレージ アカウント内の *$MetricsHourPrimaryTransactionsBlob* テーブルに書き込まれるテーブル エントリとして記録されます。 *$MetricsHourSecondaryTransactionsBlob* テーブルには、RA-GRS ストレージ アカウントを使用している場合のセカンダリ エンドポイントに対するトランザクションが記録されます。

> [!NOTE]
> 汎用ストレージ アカウントがあり、そのアカウントに、ページ BLOB のほか、ブロック BLOB データと追加 BLOB データと共に仮想マシン ディスクを格納している場合、この見積もりプロセスは適用できません。 これは、BLOB の種類に基づいて、BLOB ストレージ アカウントに移行され得るブロック BLOB と追加 BLOB のみの容量とトランザクションのメトリックを区別できないためです。

データ使用量とアクセス パターンを正確に見積もるには、通常の使用状況を表すメトリックの保有期間を選択したうえで推定することをお勧めします。 その方法の 1 つとして、メトリック データを 7 日間保持し、そのデータを毎週収集して、月末に分析を行う方法があります。 そのほかに、過去 30 日間のメトリック データを保持し、30 日の期間の最後にそのデータを収集、分析する方法もあります。

メトリック データの有効化、収集、表示の詳細については、「[Azure のストレージ メトリックの有効化とメトリック データの表示](../common/storage-enable-and-view-metrics.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json)」を参照してください。

> [!NOTE]
> 分析データの格納、アクセス、ダウンロードについても、通常のユーザー データと同様に課金されます。

### <a name="utilizing-usage-metrics-to-estimate-costs"></a>コストを見積もるための使用状況メトリックの利用

### <a name="storage-costs"></a>ストレージ コスト

行キー *"data"* がある容量メトリック テーブル *$MetricsCapacityBlob* の最新のエントリは、ユーザー データによって使用されたストレージ容量を示します。 行キー *"analytics"* がある容量メトリック テーブル *$MetricsCapacityBlob* の最新のエントリは、分析ログによって使用されたストレージ容量を示します。

ユーザー データと分析ログ (有効な場合) の両方によって使用されたこの合計容量は、ストレージ アカウントにデータを格納するコストを見積もるために使用できます。 また、同じ方法を使用して、汎用ストレージ アカウントでのブロック BLOB と追加 BLOB のストレージ コストを見積もることができます。

### <a name="transaction-costs"></a>トランザクション コスト

*"TotalBillableRequests"*の合計は、トランザクション メトリック テーブル内の API のすべてのエントリを対象とし、その特定の API のトランザクションの総数を示すものです。 "*たとえば*"、特定期間の "*GetBlob*" トランザクションの合計は、行キー "*user;GetBlob*" を備えたすべてのエントリに対する課金対象の要求をすべて加算することによって計算できます。

BLOB ストレージ アカウントのトランザクション コストを見積もるには、トランザクションを 3 つのグループに分類する必要があります (それぞれ価格が異なるため)。

* *"PutBlob"*、*"PutBlock"*、*"PutBlockList"*、*"AppendBlock"*、*"ListBlobs"*、*"ListContainers"*、*"CreateContainer"*、*"SnapshotBlob"*、*"CopyBlob"* などの書き込みトランザクション。
* *"DeleteBlob"*、*"DeleteContainer"* などの削除トランザクション。
* その他すべてのトランザクション。

汎用ストレージ アカウントのトランザクション コストを見積もるには、操作と API に関係なく、すべてのトランザクションを集計する必要があります。

### <a name="data-access-and-geo-replication-data-transfer-costs"></a>データ アクセスと geo レプリケーション データ転送のコスト

Storage Analytics では、ストレージ アカウントに対する読み取りと書き込みのデータ量は示されませんが、トランザクション メトリック テーブルを確認することで大まかに見積もることは可能です。 トランザクション メトリック テーブル内の API の全エントリを対象とした *"TotalIngress"* の合計は、その特定の API の受信データの総量をバイトで示すものです。 同様に、 *"TotalEgress"* の合計は、送信データの総量をバイトで示します。

BLOB ストレージ アカウントのデータ アクセス コストを見積もるには、トランザクションを 2 つのグループに分類する必要があります。

* ストレージ アカウントから取得されたデータの量は、主に *"GetBlob"* 操作と *"CopyBlob"* 操作の *"TotalEgress"* の合計を確認することで見積もることができます。

* ストレージ アカウントに書き込まれたデータの量は、主に *"PutBlob"* 操作、*"PutBlock"* 操作、*"CopyBlob"* 操作、*"AppendBlock"* 操作の *"TotalIngress"* の合計を確認することで見積もることができます。

また、BLOB ストレージ アカウントの geo レプリケーション データ転送のコストは、GRS ストレージ アカウントまたは RA-GRS ストレージ アカウントを使用する場合に書き込まれるデータ量の見積もりを使用することによって計算することもできます。

> [!NOTE]
> ホット ストレージ層またはクール ストレージ層を使用する場合のコストの計算に関する詳細な例については、「*Azure Storage Pricing* 」というページにある、 ["ホットおよびクール アクセス層とはどのようなものですか? また、どちらを使用すればよいのでしょうか?"](https://azure.microsoft.com/pricing/details/storage/)にサインインします。

## <a name="migrating-existing-data"></a>既存のデータの移行

BLOB ストレージ アカウントは、ブロック BLOB と追加 BLOB の格納に特化しています。 テーブル、キュー、ファイル、ディスク、および BLOB を格納できる既存の汎用ストレージ アカウントは、BLOB ストレージ アカウントに変換することはできません。 つまり、ストレージ層を使用するには、新しい BLOB ストレージ アカウントを作成し、既存のデータを新たに作成したアカウントに移行する必要があります。

オンプレミス ストレージ デバイス、サード パーティのクラウド ストレージ プロバイダー、または Azure の既存の汎用ストレージ アカウントから、既存のデータを BLOB ストレージ アカウントに移行するには、以下の方法を使用できます。

### <a name="azcopy"></a>AzCopy

AzCopy は、Azure Storage との間で高パフォーマンスのデータ コピーを行うように設計された Windows コマンドライン ユーティリティです。 AzCopy を使用して、既存の汎用ストレージ アカウントから BLOB ストレージ アカウントにデータをコピーできます。またオンプレミス ストレージ デバイスから BLOB ストレージ アカウントにデータをアップロードすることもできます。

詳細については、「 [AzCopy コマンド ライン ユーティリティを使用してデータを転送する](../common/storage-use-azcopy.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json)」を参照してください。

### <a name="data-movement-library"></a>データ移動ライブラリ

Azure Storage Data Movement Library for .NET は、AzCopy を動作させているコア データ移動フレームワークに基づいています。 ライブラリは、AzCopy と同じように、高パフォーマンスで信頼性が高く簡単なデータ転送操作ができるように設計されています。 そのため、AzCopy によって提供される機能の利点をアプリケーションでネイティブに活用でき、AzCopy の外部インスタンスを実行したり監視したりする必要はありません。

詳細については、「 [Azure Storage Data Movement Library for .Net](https://github.com/Azure/azure-storage-net-data-movement)

### <a name="rest-api-or-client-library"></a>REST API またはクライアント ライブラリ

Azure クライアント ライブラリのいずれかまたは Azure ストレージ サービス REST API を使用して、データを BLOB ストレージ アカウントに移行するためのカスタム アプリケーションを作成することができます。 Azure Storage には、.NET、Java、C++、Node.js、PHP、Ruby、Python などの複数の言語とプラットフォーム用の豊富なクライアント ライブラリが用意されています。 クライアント ライブラリは、再試行ロジック、ログ、並列アップロードといった高度な機能を提供します。 また、REST API を直接使用して開発することもでき、HTTP/HTTPS 要求を行うどの言語からでも呼び出すことができます。

詳細については、 [Azure Blob Storage の概要](storage-dotnet-how-to-use-blobs.md)に関するページを参照してください。

> [!NOTE]
> クライアント側の暗号化を使用して暗号化された BLOB には、その BLOB と共に格納される暗号化関連メタデータが格納されます。 すべてのコピー メカニズムで、BLOB メタデータと、特に暗号化に関連するメタデータが必ず保持されることがきわめて重要です。 このメタデータなしで BLOB をコピーした場合、BLOB コンテンツを再度取得することはできません。 暗号化関連メタデータの詳細については、[Azure Storage のクライアント側の暗号化](../common/storage-client-side-encryption.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json)に関するページを参照してください。

## <a name="faq"></a>FAQ

1. **既存のストレージ アカウントを引き続き使用できますか。**

    はい。既存のストレージ アカウントはまだ使用可能であり、料金や機能も変更されていません。  それらにはストレージ層を選択する機能がなく、今後も階層化機能は導入されません。

2. **BLOB ストレージ アカウントは、いつ、どのようなときに使用し始めればよいですか。**

    BLOB ストレージ アカウントは、BLOB を格納するために特化しており、新しい BLOB 主体の機能を導入します。 今後、BLOB ストレージ アカウントは、このアカウントの種類に基づいて階層型ストレージや階層化などの新たな機能が導入されるため、BLOB を格納する方法として推奨されるようになります。 ただし、いつ移行するかは、ビジネス要件に応じてお客様が判断する必要があります。

3. **既存のストレージ アカウントを BLOB ストレージ アカウントに変換できますか。**

    
いいえ。 BLOB ストレージ アカウントは別種のストレージ アカウントであり、前に説明したように、新規に作成してデータを移行する必要があります。

4. **同じアカウントの両方のストレージ層にオブジェクトを格納することはできますか。**

    はい。 アカウント レベルで設定された "*アクセス レベル*" 属性は、明示的に層が設定されていない、そのアカウント内のすべてのオブジェクトに適用される既定の層です。 ただし、BLOB レベルの階層制御 (プレビュー) では、アカウントで設定されているアクセス層に関係なく、オブジェクト レベルでアクセス層を設定できます。 3 つのストレージ層 (ホット、クール、またはアーカイブ) のいずれの BLOB も同じアカウント内に存在することができます。

5. **自分の BLOB ストレージ アカウントのストレージ層を変更することはできますか。**

    はい。ストレージ アカウントの "*アクセス レベル*" 属性を設定して、ストレージ層を変更することができます。 ストレージ層の変更は、アカウントに格納されている、層が明示的に設定されていないすべてのオブジェクトに適用されます。 ストレージ層をホットからクールに変更すると、書き込み操作 (10,000 件単位) とデータ書き込み (GB 単位) の両方の料金が発生します (BLOB ストレージ アカウントのみ)。一方、クールからホットに変更すると、アカウント内のすべてのデータの読み取りに対し、読み取り操作 (10,000 件単位) とデータ取得 (GB 単位) の両方の料金が発生します。

6. **自分の BLOB ストレージ アカウントのストレージ層は、どの程度の頻度で変更できますか。**

    ストレージ層の変更頻度に対して制限は設けられていませんが、ストレージ層をクールからホットに変更すると、かなりの料金が発生することに注意してください。 ストレージ層を頻繁に変更することは、お勧めしません。

7. **クール ストレージ層内の BLOB の動作は、ホット ストレージ層内の BLOB とは異なりますか。**

    ホット ストレージ層内の BLOB の待機時間は、汎用ストレージ アカウントの BLOB と同じになります。 クール ストレージ層内の BLOB の待機時間は、汎用ストレージ アカウントの BLOB と類似しています (ミリ秒)。 アーカイブ ストレージ層の BLOB の待ち時間は数時間に及びます。

    クール ストレージ層内の BLOB は、ホット ストレージ層に格納された BLOB よりも可用性サービス レベル (SLA) が若干低くなります。 詳細については、「 [Storage の SLA](https://azure.microsoft.com/support/legal/sla/storage)」を参照してください。

8. **BLOB ストレージ アカウントにページ BLOB と仮想マシンのディスクを保存できますか。**

    BLOB ストレージ アカウントは、ブロック BLOB と追加 BLOB のみをサポートします。ページ BLOB はサポートしません。 Azure の仮想マシンのディスクではページ BLOB が使用されるため、BLOB ストレージ アカウントは仮想マシンのディスクの格納に使用できません。 ただし、仮想マシンのディスクのバックアップを BLOB ストレージ アカウント内にブロック BLOB として格納することはできます。

9. **BLOB ストレージ アカウントを使用するには、既存のアプリケーションを変更する必要がありますか。**

    BLOB ストレージ アカウントは、ブロック BLOB と追加 BLOB に関して、汎用ストレージ アカウントとの100% の API 整合性を備えています。 アプリケーションでブロック BLOB または追加 BLOB が使用されており、[Storage Services REST API](https://msdn.microsoft.com/library/azure/dd894041.aspx) の 2014-02-14 バージョン以降を使用している限り、アプリケーションは機能します。 プロトコルの古いバージョンを使用している場合は、新しいバージョンを使用して両方の種類のストレージ アカウントとシームレスに動作するように、アプリケーションを更新する必要があります。 一般に、どの種類のストレージ アカウントを使用するかにかかわらず、常に最新バージョンを使用することをお勧めしています。

10. **ユーザー エクスペリエンスに変更はありますか。**

    BLOB ストレージ アカウントは、ブロック BLOB と追加 BLOB の格納に関しては汎用ストレージ アカウントとよく似ており、高い持続性、可用性、スケーラビリティ、パフォーマンス、セキュリティなどの Azure Storage のすべての主要機能をサポートしています。 BLOB ストレージ アカウントに固有の機能および制限と、上に記載したストレージ層を除き、いずれも同じままです。

## <a name="next-steps"></a>次のステップ

### <a name="evaluate-blob-storage-accounts"></a>BLOB ストレージ アカウントを評価する

[リージョン別に BLOB ストレージ アカウントの可用性を確認する](https://azure.microsoft.com/regions/#services)

[Azure Storage のメトリックを有効にして現在のストレージ アカウントの使用状況を評価する](../common/storage-enable-and-view-metrics.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json)

[リージョン別に Blob Storage の価格を確認する](https://azure.microsoft.com/pricing/details/storage/)

[データ転送の価格を確認する](https://azure.microsoft.com/pricing/details/data-transfers/)

### <a name="start-using-blob-storage-accounts"></a>BLOB ストレージ アカウントの利用を開始する

[Azure Blob Storage を使用する](storage-dotnet-how-to-use-blobs.md)

[Azure Storage との間でのデータの移動](../common/storage-moving-data.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json)

[AzCopy コマンド ライン ユーティリティを使用してデータを転送する](../common/storage-use-azcopy.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json)

[自分のストレージ アカウントを調べる](http://storageexplorer.com/)
