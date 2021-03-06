---
title: "Log Analytics アラートから Azure Automation Runbook を呼び出す | Microsoft Docs"
description: "この記事では、Microsoft OMS Log Analytics アラートから Automation Runbook を呼び出す方法の概要を示します。"
services: automation
documentationcenter: 
author: eslesar
manager: jwhit
editor: 
ms.assetid: 
ms.service: automation
ms.workload: tbd
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: get-started-article
ms.date: 01/31/2017
ms.author: magoedte
ms.openlocfilehash: 10b445f8fcaa80182119e47f37ffb11240a46869
ms.sourcegitcommit: 5d3e99478a5f26e92d1e7f3cec6b0ff5fbd7cedf
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 12/06/2017
---
# <a name="calling-an-azure-automation-runbook-from-an-oms-log-analytics-alert"></a>OMS Log Analytics アラートから Azure Automation Runbook を呼び出す

結果が条件 (プロセッサ使用率の長期的なスパイクなど) に一致した場合やビジネス アプリケーションの機能にとって重要なアプリケーション プロセスが失敗し、対応するイベントを Windows イベント ログに書き込んだ場合にアラート レコードを作成するように Log Analytics でアラートを構成すると、そのアラートは、問題を自動修復しようとして Automation Runbook を自動的に実行できます。  

アラートの構成で Runbook を呼び出す方法は 2 つあります。

1. webhook を使用する。
   * これは、OMS ワークスペースが Automation アカウントにリンクされていない場合に使用できる唯一の方法です。
   * Automation アカウントを既に OMS ワークスペースにリンクしている場合も引き続きこのオプションを使用できます。  

2. 直接 Runbook を選択する。
   * このオプションは、OMS ワークスペースが Automation アカウントにリンクされている場合のみ使用できます。

## <a name="calling-a-runbook-using-a-webhook"></a>webhook を使用して Runbook を呼び出す

webhook を使用することにより、単一の HTTP 要求を通して Azure Automation で特定の Runbook を開始することができます。 アラート アクションとして webhook を使用して Runbook を呼び出すように [Log Analytics アラート](../log-analytics/log-analytics-alerts.md#alert-rules)を構成する前に、まず、この方法を使用して呼び出される Runbook 用に webhook を作成する必要があります。 [webhook の作成](automation-webhooks.md#creating-a-webhook)に関する記事の手順を実行し、アラート ルールの構成時に参照できるように webhook URL を忘れずに記録しておいてください。   

## <a name="calling-a-runbook-directly"></a>直接 Runbook を呼び出す

OMS ワークスペースで Automation & Control サービスをインストールして構成してある場合は、アラートの Runbook アクションのオプションを構成するときに、**[Runbook の選択]** ボックスの一覧ですべての Runbook を表示し、そのアラートに対応して実行する特定の Runbook を選択できます。 選択した Runbook は、Azure クラウド内のワークスペース、または Hybrid Runbook Worker で実行できます。 Runbook オプションを使用してアラートを作成すると、その Runbook 用に webhook が作成されます。 その webhook は、Automation アカウントに移動し、選択した Runbook の webhook ウィンドウに移動すると表示されます。 アラートを削除しても webhook は削除されませんが、ユーザーは手動で webhook を削除できます。 webhook が削除されない場合、これは問題ではありません。単に、構成された Automation アカウントを管理するために最終的に削除する必要がある孤立した項目にすぎません。  

## <a name="characteristics-of-a-runbook-for-both-options"></a>Runbook の特性 (両方のオプションに該当)

Log Analytics アラートから Runbook を呼び出す 2 つの方法には、アラート ルールを構成する前に理解しておくべき特性があります。

* **Object** 型の **WebhookData** という Runbook 入力パラメーターが必要です。 これには必須と任意があります。 アラートは、この入力パラメーターを使用して検索結果を Runbook に渡します。

    ```powershell
    param  
        (  
        [Parameter (Mandatory=$true)]  
        [object] $WebhookData  
        )
    ```
*  WebhookData を PowerShell オブジェクトに変換するためのコードが必要です。

    ```powershell
    $SearchResult = (ConvertFrom-Json $WebhookData.RequestBody).SearchResult.value
    ```

    *$SearchResult* はオブジェクトの配列になります。各オブジェクトには、1 つの検索結果からの値を含むフィールドが含まれています。

## <a name="example-walkthrough"></a>例のチュートリアル

ここでは、次のグラフィカルな Runbook の例を使用してこのプロセスのしくみを説明します。この Runbook は Windows サービスを開始します。<br><br> ![Windows サービスのグラフィカルな Runbook の開始](media/automation-invoke-runbook-from-omsla-alert/automation-runbook-restartservice.png)<br>

Runbook には、**WebhookData** という **Object** 型の入力パラメーターが 1 つあり、\*.SearchResult\* を含むアラートから渡された webhook データが含まれています。<br><br> ![Runbook の入力パラメーター](media/automation-invoke-runbook-from-omsla-alert/automation-runbook-restartservice-inputparameter.png)<br>

この例では、Log Analytics で、2 つのカスタム フィールド (\*SvcDisplayName\_CF\* と \*SvcState\_CF\*) を作成し、システム イベント ログに書き込まれたイベントからサービスの表示名と状態 (つまり、実行中や停止済み) を抽出します。 その後、Windows システムで印刷スプーラー サービスが停止したときに検出できるように `Type=Event SvcDisplayName_CF="Print Spooler" SvcState_CF="stopped"` 検索クエリを含むアラート ルールを作成します。 これは関心のあるサービスならどれでもかまいませんが、この例では、Windows OS に付属している既存のサービスの 1 つを参照しています。 アラート アクションは、この例で使用されている Runbook を実行し、Hybrid Runbook Worker で動作するように構成されています。これは、ターゲット システムで有効になっています。   

Runbook コード アクティビティ **Get Service Name from LA** は、JSON 形式の文字列をオブジェクトの種類に変換し、項目 \*SvcDisplayName\_CF\* をフィルター処理して Windows サービスの表示名を抽出し、この値を次のアクティビティに渡します。次のアクティビティは、このサービスが停止していることを確認してから、サービスの再開を試みます。 *SvcDisplayName_CF* は、この例を説明するために Log Analytics に作成された[カスタム フィールド](../log-analytics/log-analytics-custom-fields.md)です。

```powershell
$SearchResult = (ConvertFrom-Json $WebhookData.RequestBody).SearchResult.value
$SearchResult.SvcDisplayName_CF  
```

サービスが停止すると、Log Analytics のアラート ルールは、一致を検出して Runbook をトリガーし、アラートのコンテキストを Runbook に送信します。 Runbook は、サービスが停止していることを確認するためのアクションを実行します。停止している場合は、サービスを再開し、正しく開始されたことを確認して、結果を出力します。     

また、Automation アカウントを OMS ワークスペースにリンクしていない場合は、Runbook をトリガーする webhook アクションでアラート ルールを構成し、前に説明したガイダンスに従って JSON 形式の文字列を変換して \*.SearchResult\* をフィルター処理するよう Runbook を構成します。    

## <a name="next-steps"></a>次のステップ

* Log Analytics のアラートの詳細とアラートの作成方法については、「[Log Analytics のアラート](../log-analytics/log-analytics-alerts.md)」を参照してください。

* webhook を使用して Runbook をトリガーする方法を理解するには、「[Azure Automation Webhook](automation-webhooks.md)」を参照してください。
