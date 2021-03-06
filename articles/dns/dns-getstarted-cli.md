---
title: "Azure CLI 2.0 で Azure DNS の使用を開始する | Microsoft Docs"
description: "Azure DNS で、DNS ゾーンとレコードを作成する方法について説明します。 Azure CLI 2.0 を使用して最初の DNS ゾーンとレコードを作成して管理するためのステップ バイ ステップ ガイドです。"
services: dns
documentationcenter: na
author: KumuD
manager: timlt
editor: 
tags: azure-resource-manager
ms.assetid: fb0aa0a6-d096-4d6a-b2f6-eda1c64f6182
ms.service: dns
ms.devlang: azurecli
ms.topic: get-started-article
ms.tgt_pltfrm: na
ms.workload: infrastructure-services
ms.date: 03/10/2017
ms.author: kumud
ms.openlocfilehash: 76782ac1e78cd0f7da4bc1aad8eff00d79865ed7
ms.sourcegitcommit: aaba209b9cea87cb983e6f498e7a820616a77471
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 12/12/2017
---
# <a name="get-started-with-azure-dns-using-azure-cli-20"></a>Azure CLI 2.0 で Azure DNS の使用を開始する

> [!div class="op_single_selector"]
> * [Azure Portal](dns-getstarted-portal.md)
> * [PowerShell](dns-getstarted-powershell.md)
> * [Azure CLI 2.0](dns-getstarted-cli.md)

この記事では、Windows、Mac、Linux で使用できるクロスプラットフォーム Azure CLI 2.0 を使用して、最初の DNS ゾーンとレコードを作成する手順について説明します。 これらの手順は、Azure Portal または Azure PowerShell を使用して実行することもできます。

DNS ゾーンは、特定のドメインの DNS レコードをホストするために使用されます。 Azure DNS でドメインのホストを開始するには、そのドメイン名用に DNS ゾーンを作成する必要があります。 ドメインの DNS レコードはすべて、この DNS ゾーン内に作成されます。 最後に、DNS ゾーンをインターネットに公開するには、ドメインのネーム サーバーを構成する必要があります。 ここでは、その手順について説明します。

以降の手順は、Azure CLI 2.0 がインストール済みで、既にサインインしていることを前提としています。 詳細については、[Azure CLI 2.0 を使用して DNS ゾーンを管理する方法](dns-operations-dnszones-cli.md)に関するページをご覧ください。

## <a name="create-the-resource-group"></a>リソース グループの作成

DNS ゾーンを作成する前に、DNS ゾーンが含まれるリソース グループを作成します。 コマンドを次に示します。

```azurecli
az group create --name MyResourceGroup --location "West US"
```

## <a name="create-a-dns-zone"></a>DNS ゾーンの作成

DNS ゾーンは、`az network dns zone create` コマンドを使用して作成します。 このコマンドのヘルプを表示するには、「`az network dns zone create -h`」と入力します。

次の例では、*MyResourceGroup* というリソース グループに *contoso.com* という DNS ゾーンを作成します。 この例の値を実際の値に置き換えて、DNS ゾーンを作成できます。

```azurecli
az network dns zone create -g MyResourceGroup -n contoso.com
```


## <a name="create-a-dns-record"></a>DNS レコードの作成

DNS レコードを作成するには、`az network dns record-set [record type] add-record` コマンドを使用します。 A レコードなどの詳細については、「`azure network dns record-set A add-record -h`」を参照してください。

下の例では、リソース グループ "MyResourceGroup" で DNS ゾーン "contoso.com" に相対名 "www" を持つレコードを作成します。 レコード セットの完全修飾名は、"www.contoso.com" になります。 また、レコードの種類は "A"、IP アドレスは "1.2.3.4"、既定の TTL として 3,600 秒 (1 時間) が使用されています。

```azurecli
az network dns record-set a add-record -g MyResourceGroup -z contoso.com -n www -a 1.2.3.4
```

その他のレコードの種類、複数のレコードを持つレコード セット、代替 TTL 値、既存のレコードの変更については、[Azure CLI 2.0 を使用した DNS レコードおよびレコード セットの管理](dns-operations-recordsets-cli.md)に関するページをご覧ください。


## <a name="view-records"></a>レコードの表示

ゾーンで DNS レコードを表示するには、次を使用します。

```azurecli
az network dns record-set list -g MyResourceGroup -z contoso.com
```


## <a name="update-name-servers"></a>ネーム サーバーの更新

DNS ゾーンとレコードを正しく設定したら、Azure DNS ネーム サーバーを使用するようにドメイン名を構成する必要があります。 これにより、インターネット上の他のユーザーが DNS レコードを検索できるようになります。

ゾーンのネーム サーバーを指定するには、`az network dns zone show` コマンドを使用します。 ネーム サーバー名を表示するには、次の例のように、JSON 出力を使用します。

```azurecli
az network dns zone show -g MyResourceGroup -n contoso.com -o json

{
  "etag": "00000003-0000-0000-b40d-0996b97ed101",
  "id": "/subscriptions/a385a691-bd93-41b0-8084-8213ebc5bff7/resourceGroups/myresourcegroup/providers/Microsoft.Network/dnszones/contoso.com",
  "location": "global",
  "maxNumberOfRecordSets": 5000,
  "name": "contoso.com",
  "nameServers": [
    "ns1-01.azure-dns.com.",
    "ns2-01.azure-dns.net.",
    "ns3-01.azure-dns.org.",
    "ns4-01.azure-dns.info."
  ],
  "numberOfRecordSets": 3,
  "resourceGroup": "myresourcegroup",
  "tags": {},
  "type": "Microsoft.Network/dnszones"
}
```

このネーム サーバーは、ドメイン名レジストラー (ドメイン名を購入した場所) で構成する必要があります。 レジストラーにより、ドメインのネーム サーバーを設定するオプションが提供されます。 詳細については、「[Azure DNS へのドメインの委任](dns-domain-delegation.md)」を参照してください。

## <a name="delete-all-resources"></a>すべてのリソースの削除
 
この記事で作成したすべてのリソースを削除するには、次の手順を実行します。

```azurecli
az group delete --name MyResourceGroup
```

## <a name="next-steps"></a>次のステップ

Azure DNS の詳細については、「[Azure DNS の概要](dns-overview.md)」を参照してください。

Azure DNS での DNS ゾーンの管理の詳細については、[Azure CLI 2.0 を使用した Azure DNS での DNS ゾーンの管理](dns-operations-dnszones-cli.md)に関するページをご覧ください。

Azure DNS での DNS レコードの管理の詳細については、「[Azure CLI 2.0 を使用して Azure DNS のレコードおよびレコード セットを管理する](dns-operations-recordsets-cli.md)」を参照してください。
