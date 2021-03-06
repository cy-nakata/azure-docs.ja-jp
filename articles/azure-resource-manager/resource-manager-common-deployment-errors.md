---
title: "Azure へのデプロイで発生する一般的なエラーのトラブルシューティング | Microsoft Docs"
description: "Azure Resource Manager を使用した Azure へのリソースのデプロイ時に発生する一般的なエラーの解決方法について説明します。"
services: azure-resource-manager
documentationcenter: 
tags: top-support-issue
author: tfitzmac
manager: timlt
editor: tysonn
keywords: "デプロイのエラー、Azure へのデプロイ、Azure へのデプロイ"
ms.service: azure-resource-manager
ms.devlang: na
ms.topic: support-article
ms.tgt_pltfrm: na
ms.workload: na
ms.date: 11/29/2017
ms.author: tomfitz
ms.openlocfilehash: db7561c31c0748ae5c1500ba8c39dfa79274901e
ms.sourcegitcommit: 5a6e943718a8d2bc5babea3cd624c0557ab67bd5
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 12/01/2017
---
# <a name="troubleshoot-common-azure-deployment-errors-with-azure-resource-manager"></a>Azure Resource Manager を使用した Azure へのデプロイで発生する一般的なエラーのトラブルシューティング

この記事では、Azure へのデプロイで発生する可能性がある一般的なエラーについて説明し、そのエラーを解決するための情報を提供します。 デプロイ エラーのエラー コードを見つけることができない場合は、「[エラー コードを見つける](#find-error-code)」を参照してください。

## <a name="error-codes"></a>エラー コード

| エラー コード | 対応策 | 詳細情報 |
| ---------- | ---------- | ---------------- |
| AccountNameInvalid | ストレージ アカウントの名前付けの制限に従ってください。 | [ストレージ アカウント名の解決](resource-manager-storage-account-name-errors.md) |
| AccountPropertyCannotBeSet | 使用可能なストレージ アカウント プロパティを確認してください。 | [storageAccounts](/azure/templates/microsoft.storage/storageaccounts) |
| AllocationFailed | クラスターまたはリージョンに使用可能なリソースがないか、要求された VM サイズをサポートできません。 後で要求を再試行するか、別の VM サイズを要求します。 | [Linux のプロビジョニングと割り当ての問題](../virtual-machines/linux/troubleshoot-deployment-new-vm.md) および [Windows のプロビジョニングと割り当ての問題](../virtual-machines/windows/troubleshoot-deployment-new-vm.md) |
| AnotherOperationInProgress | 同時実行操作の完了を待ちます。 | |
| AuthorizationFailed | お客様のアカウントまたはサービス プリンシパルには、デプロイを完了するために十分なアクセス権がありません。 自分のアカウントが属するロールと、デプロイの範囲に対するアクセス権を確認してください。 | [Azure のロールベースのアクセス制御](../active-directory/role-based-access-control-configure.md) |
| BadRequest | Resource Manager で予期される値と一致しないデプロイ値を送信しました。 トラブルシューティングの方法については、内部ステータス メッセージを確認してください。 | [テンプレート リファレンス](/azure/templates/)と[サポートされている場所](resource-manager-template-location.md) |
| 競合 | リソースの現在の状態では許可されていない操作を要求しています。 たとえば、ディスクのサイズ変更が許可されているのは、VM の作成時と VM の割り当て解除時のみです。 | |
| DeploymentActive | このリソース グループへの同時実行デプロイが完了するまで待ちます。 | |
| DnsRecordInUse | DNS レコード名は、一意の名前にする必要があります。 別の名前を指定するか、既存のレコードを変更してください。 | |
| ImageNotFound | VM イメージの設定を確認してください。 | [Linux イメージのトラブルシューティング](../virtual-machines/linux/troubleshoot-deployment-new-vm.md)と [Windows イメージのトラブルシューティング](../virtual-machines/windows/troubleshoot-deployment-new-vm.md) |
| InUseSubnetCannotBeDeleted | リソースを更新しようとするときにこのエラーが発生することがありますが、リソースを削除して作成すると、要求が処理されます。 変更されていないすべての値を指定してください。 | [リソースを更新する](/azure/architecture/building-blocks/extending-templates/update-resource) |
| InvalidAuthenticationTokenTenant | 該当するテナントのアクセス トークンを取得します。 トークンは、自分のアカウントが属するテナントからのみ取得できます。 | |
| InvalidContentLink | これは、入れ子になった無効なテンプレートにリンクしようとしたことが原因と考えられます。 入れ子になったテンプレートに指定した URI を十分に確認してください。 ストレージ アカウントにテンプレートがある場合は、URI がアクセス可能であることを確認してください。 場合によっては、SAS トークンを渡す必要があります。 | [リンク済みテンプレート](resource-group-linked-templates.md) |
| InvalidParameter | リソースに対して指定したいずれかの値が、予期される値に一致しません。 このエラーはさまざまな状況が原因となって発生する可能性があります。 たとえば、パスワードが十分でない場合や、BLOB 名が正しくない場合があります。 エラー メッセージを確認して、どの値を修正する必要があるかを判断してください。 | |
| InvalidRequestContent | デプロイの値に予期されない値が含まれているか、必要な値が不足しています。 リソースの種類の値を確認してください。 | [テンプレート リファレンス](/azure/templates/) |
| InvalidRequestFormat | デプロイを実行するときにデバッグ ログを有効にし、要求の内容を確認してください。 | [デバッグ ログ](resource-manager-troubleshoot-tips.md#enable-debug-logging) |
| InvalidResourceNamespace | **type** プロパティに指定したリソース名前空間を確認してください。 | [テンプレート リファレンス](/azure/templates/) |
| InvalidResourceReference | リソースがまだ存在しないか、正しく参照されていません。 依存関係を追加する必要があるかどうかを確認してください。 シナリオに必要なパラメーターを含めて **reference** 関数を使用しているかどうかを確認してください。 | [依存関係を解決する](resource-manager-not-found-errors.md) |
| InvalidResourceType | **type** プロパティに指定したリソースの種類を確認してください。 | [テンプレート リファレンス](/azure/templates/) |
| InvalidTemplate | テンプレートの構文にエラーがないか確認してください。 | [無効なテンプレートを解決する](resource-manager-invalid-template-errors.md) |
| LinkedAuthorizationFailed | デプロイ先のリソース グループと同じテナントに自分のアカウントが属しているかどうかを確認してください。 | |
| LinkedInvalidPropertyId | リソースのリソース ID が正しく解決されていません。 リソース ID に必要なすべての値 (サブスクリプション ID、リソース グループ名、リソースの種類、親リソース名 (必要な場合)、リソース名など) を指定しているかどうかを確認してください。 | |
| LocationRequired | リソースの場所を指定します。 | [場所を設定する](resource-manager-template-location.md) |
| MissingRegistrationForLocation | リソース プロバイダーの登録状態、およびサポートされている場所を確認してください。 | [登録を解決する](resource-manager-register-provider-errors.md) |
| MissingSubscriptionRegistration | リソース プロバイダーにサブスクリプションを登録してください。 | [登録を解決する](resource-manager-register-provider-errors.md) |
| NoRegisteredProviderFound | リソース プロバイダーの登録状態を確認してください。 | [登録を解決する](resource-manager-register-provider-errors.md) |
| NotFound | 依存するリソースを、親リソースと並列にデプロイしようとしています。 依存関係を追加する必要があるかどうかを確認してください。 | [依存関係を解決する](resource-manager-not-found-errors.md) |
| OperationNotAllowed | デプロイで、サブスクリプション、リソース グループ、またはリージョンのクォータを超過する操作が試みられています。 可能であれば、クォータ内に収まるようにデプロイを修正してください。 修正できない場合は、クォータの変更を要求することを検討してください。 | [クォータを解決する](resource-manager-quota-errors.md) |
| ParentResourceNotFound | 子リソースを作成する前に親リソースが存在することを確認してください。 | [親リソースを解決する](resource-manager-parent-resource-errors.md) |
| PrivateIPAddressInReservedRange | 指定した IP アドレスは、Azure で必要なアドレス範囲に含まれます。 予約済みの範囲を避けるように IP アドレスを変更してください。 | [IP アドレス](../virtual-network/virtual-network-ip-addresses-overview-arm.md) |
| PrivateIPAddressNotInSubnet | 指定した IP アドレスがサブネットの範囲外です。 サブネットの範囲に収まるように IP アドレスを変更してください。 | [IP アドレス](../virtual-network/virtual-network-ip-addresses-overview-arm.md) |
| PropertyChangeNotAllowed | デプロイ済みのリソースで、一部のプロパティを変更できません。 リソースを更新する際は、許可されているプロパティに変更を制限してください。 | [リソースを更新する](/azure/architecture/building-blocks/extending-templates/update-resource) |
| RequestDisallowedByPolicy | デプロイ時に実行しようとしているアクションを禁止するリソース ポリシーがサブスクリプションに含まれます。 アクションをブロックしているポリシーを見つけてください。 可能であれば、ポリシーの制限を満たすようにデプロイを変更してください。 | [ポリシーを解決する](resource-manager-policy-requestdisallowedbypolicy-error.md) |
| ReservedResourceName | 予約された名前が含まれていないリソース名を指定します。 | [予約されたリソース名](resource-manager-reserved-resource-name.md) |
| ResourceGroupBeingDeleted | 削除が完了するまで待ちます。 | |
| ResourceGroupNotFound | デプロイのターゲット リソース グループの名前を確認してください。 サブスクリプションにそのリソース グループが既に存在している必要があります。 サブスクリプションのコンテキストを確認してください。 | [Azure CLI](/cli/azure/account?#az_account_set)、[PowerShell](/powershell/module/azurerm.profile/set-azurermcontext) |
| ResourceNotFound | 解決できないリソースをデプロイで参照しています。 **reference** 関数に、シナリオに必要なパラメーターを含まれていることを確認してください。 | [参照を解決する](resource-manager-not-found-errors.md) |
| ResourceQuotaExceeded | デプロイで、サブスクリプション、リソース グループ、またはリージョンのクォータを超過するリソースの作成が試みられています。 可能であれば、クォータ内に収まるようにインフラストラクチャを変更してください。 修正できない場合は、クォータの変更を要求することを検討してください。 | [クォータを解決する](resource-manager-quota-errors.md) |
| SkuNotAvailable | 選択した場所で利用可能な SKU (VM サイズなど) を選択します。 | [SKU を解決する](resource-manager-sku-not-available-errors.md) |
| StorageAccountAlreadyExists | ストレージ アカウントに一意の名前を指定してください。 | [ストレージ アカウント名の解決](resource-manager-storage-account-name-errors.md)  |
| StorageAccountAlreadyTaken | ストレージ アカウントに一意の名前を指定してください。 | [ストレージ アカウント名の解決](resource-manager-storage-account-name-errors.md) |
| StorageAccountNotFound | サブスクリプション、リソース グループ、および使用するストレージ アカウントの名前を確認してください。 | |
| SubnetsNotInSameVnet | 仮想マシンでは、仮想ネットワークを 1 つのみを持つことができます。 複数の NIC をデプロイするときは、それらが同じ仮想ネットワークに属していることを確認してください。 | [複数の NIC](../virtual-machines/windows/multiple-nics.md) |

## <a name="find-error-code"></a>エラー コードを見つける

デプロイ中にエラーが発生した場合、Resource Manager がエラー コードを返します。 エラー メッセージは、ポータル、PowerShell、または Azure CLI を介して確認できます。 外部エラー メッセージは、一般的すぎてトラブルシューティングに向かない可能性があります。 エラーに関する詳細情報が記載された内部メッセージを探してください。 詳細については、「[エラー コードの判断](resource-manager-troubleshoot-tips.md#determine-error-code)」を参照してください。

## <a name="next-steps"></a>次のステップ
* 監査アクションについては、「 [リソース マネージャーの監査操作](resource-group-audit.md)」をご覧ください。
* デプロイ時にエラーが発生した場合の対応については、 [デプロイ操作の確認](resource-manager-deployment-operations.md)に関するページを参照してください。
