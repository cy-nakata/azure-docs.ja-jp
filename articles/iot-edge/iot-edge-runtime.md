---
title: "Azure IoT Edge ランタイムについて | Microsoft Docs"
description: "Azure IoT Edge ランタイムの概要、およびそのエッジ デバイスの使用方法について説明します。"
services: iot-edge
keywords: 
author: kgremban
manager: timlt
ms.author: kgremban
ms.date: 10/05/2017
ms.topic: article
ms.service: iot-edge
ms.openlocfilehash: 7b37f9e103644d2492f69f4a4cc80d3fd57d4aa4
ms.sourcegitcommit: 9a61faf3463003375a53279e3adce241b5700879
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 11/15/2017
---
# <a name="understand-the-azure-iot-edge-runtime-and-its-architecture---preview"></a>Azure IoT Edge ランタイムとそのアーキテクチャについて - プレビュー

IoT Edge ランタイムは、プログラムの集合体で、IoT Edge デバイスと認識されるためには、デバイスにインストールする必要があります。 これらの IoT Edge ランタイムで構成されるコンポーネントを使用することにより、IoT Edge デバイスは、エッジで実行するコードを受信し、結果を通信できます。 

IoT Edge ランタイムは、IoT Edge デバイスで次の機能を実行します。

* デバイスにワークロードをインストールし、更新する。
* デバイス上の Azure IoT Edge のセキュリティ標準を維持する。
* [IoT Edge モジュール] [lnk モジュール]の実行を継続する。
* モジュールの正常性をクラウドに報告してリモート監視を可能にする。
* ダウンストリームのリーフ デバイスと IoT Edge デバイス間の通信を円滑化する。
* IoT Edge デバイス上のモジュール間の通信を円滑化する。
* IoT Edge デバイスとクラウドとの間の通信を円滑化する。

![IoT Edge ランタイムによる洞察とモジュールの正常性の IoT Hub への通信][1]

IoT Edge ランタイムには、モジュールの管理と通信の 2 つのカテゴリの役割があります。 これら 2 つの役割は、IoT Edge ランタイムを構成する 2 つのコンポーネントによって実行されます。 IoT Edge ハブは通信を実行し、IoT Edge エージェントは、モジュールの展開と監視を実行します。 

Edge ハブと Edge エージェントは、いずれも、IoT Edge デバイス上で実行される他のモジュールと同様のモジュールです。 モジュールのしくみの詳細については、[lnk モジュール]を参照してください。 

## <a name="iot-edge-hub"></a>IoT Edge ハブ

Edge ハブでは、Azure IoT Edge ランタイムを構成する 2 つのモジュールの 1 つです。 Edge ハブは、IoT Hub と同じプロトコル エンドポイントを公開することで IoT Hub のためのローカル プロキシとして動作します。 この整合性により、クライアント (デバイスまたはモジュール) は、IoT Hub と同じように IoT Edge ランタイムに接続できます。 

>[!NOTE]
> パブリック プレビューでは、Edge Hub は MQTT を使用して接続するクライアントのみサポートします。

Edge ハブは、ローカルで実行される完全バージョンの IoT Hub ではありません。 Edge ハブは、いくつかの機能を IoT Hub をサイレントにデリゲートします。 たとえば、Edge ハブは、デバイスがはじめて接続を試みたとき、IoT Hub に認証要求を送信します。 初めて接続を確立した後は、セキュリティ情報は Edge ハブによってローカルでキャッシュされます。 そのデバイスからのその後の接続は、クラウドへの認証なしに許可されます。 

>[!NOTE]
> パブリック プレビューでは、デバイスを認証するたびに、ランタイムに接続する必要があります。

IoT Edge ソリューションが使用する帯域幅を減らすために、Edge ハブは、クラウドへの実際の接続数を最適化します。 Edge ハブは、モジュールまたはリーフ デバイスなどのクライアントからの論理接続を取得し、クラウドへの 1 つの物理接続に統合します。 ソリューションの他の部分は、このプロセスの詳細を認識する必要がありません。 クライアントは、すべて、共通の接続を使って接続しているにもかかわらず、クラウドにそれぞれ独自に接続していると認識します。 

![Edge ハブは複数の物理デバイスとクラウド間のゲートウェイとして機能します。][2]

Edge ハブでは、IoT Hub に接続されているかどうかを判断できます。 接続が切断された場合は、Edge ハブがメッセージを保存するか、ツインがローカルで更新します。 接続が再確立されると、すべてのデータが同期されます。 この一時キャッシュに使用される場所は、Edge ハブのモジュール ツインのプロパティによって規定されます。 キャッシュのサイズには上限がなく、デバイスにストレージ容量がある限り拡張されます。 

>[!NOTE]
>追加のキャッシュを管理するために、一般に提供される前に製品にパラメータが追加されます。

### <a name="module-communication"></a>モジュール通信

Edge Hub を使用することで、モジュール間の通信が容易になります。 メッセージ ブローカーとして Edge Hub を使用することで、モジュールの相互独立性を維持できます。 モジュールは、メッセージを受信する入力と、メッセージを書き出す出力を指定するだけです。 ソリューション開発者は、そのソリューションに固有の順序でデータを処理できるよう、これらの入力と出力を統合します。 

![Edge Hub を使用することで、モジュール間の通信が容易になります。][3]

データを Edge ハブに送信するために、モジュールは、SendEventAsync メソッドを呼び出します。 最初の引数は、どの出力にメッセージを送信するかを指定します。 次の疑似コードは、output1 にメッセージを送信します。

    DeviceClient client = new DeviceClient.CreateFromConnectionString(moduleConnectionString, settings); 
    await client.OpenAsync(); 
    await client.SendEventAsync(“output1”, message); 

メッセージを受信するには、特定の入力で受け取ったメッセージを処理するコールバックを登録します。 次の疑似コードは、input1 で受信したすべてのメッセージを処理する関数 messageProcessor を登録します。

    await client.SetEventHandlerAsync(“input1”, messageProcessor, userContext);
    
ソリューション開発者は、Edge ハブがモジュール間でメッセージを渡す方法を決定するルールを指定します。 ルーティング ルールはクラウドで定義され、そのデバイス ツインで Edge ハブにプッシュ ダウンされます。 IoT Hub ルートと同じ構文を使用して、Azure IoT Edge のモジュール間のルートが定義されます。 

<!--- For more info on how to declare routes between modules, see []. --->   

![モジュール間のルート][4]

## <a name="iot-edge-agent"></a>IoT Edge エージェント

IoT Edge エージェントは、Azure IoT Edge ランタイムを構成するもう 1 つのモジュールです。 このモジュールは、モジュールをインスタンス化し、モジュールの実行を継続し、IoT Hub にモジュールのステータスを報告します。 Edge エージェントは、他のモジュールと同じように、そのモジュール ツインを使用して、この構成データを格納します。 

Edge エージェントの実行を開始するには、azure-iot-edge-runtime-ctl.py start コマンドを実行します。 エージェントは、IoT Hub からそのモジュール ツインを取得し、モジュール ディクショナリを検査します。 モジュール ディクショナリは、起動する必要があるモジュールのコレクションです。 

モジュール ディクショナリ内の各項目には、モジュールに関する特定の情報が含まれており、モジュールのライフ サイクルを制御するために、Edge エージェントによって使用されます。 重要なプロパティを次に示します。 

* **settings.image** – Edge エージェントがモジュールを起動するために使用するコンテナー イメージ。 イメージがパスワードで保護されている場合は、コンテナー レジストリの資格情報を Edge エージェントに設定する必要があります。 Edge エージェントを設定するには、次のコマンドを使用します。`azure-iot-edge-runtime-ctl.py –configure`
* **settings.createOptions** – モジュールのコンテナーを起動したときに、Docker デーモンに直接渡される文字列。 このプロパティで Docker オプションを追加することにより、ポート転送またはモジュール コンテナーへのボリュームのマウントなどの詳細オプションを設定できます。  
* **status** – Edge エージェントがモジュールに設定する状態。 大部分のユーザーが Edge を使ってデバイス上のすべてのモジュールをただちに起動する必要があるため、この値は、通常、*実行中*に設定されます。 また、モジュールの初期状態を停止中に指定し、後で Edge エージェントを起動するよう指定することができます。 Edge エージェントは、報告されたプロパティで、クラウドに各モジュールの状態を報告します。 期待されるプロパティと報告されたプロパティの間に差がある場合は、デバイスの動作が不適切であることを示しています。 次の状態がサポートされています。
   * ダウンロード中
   * 実行中
   * 異常
   * Failed
   * 停止済み
* **restartPolicy** – Edge エージェントが、モジュールを再起動する方法です。 指定できる値は、次のとおりです。
   * Never – Edge エージェントが、モジュールを再起動することはありません。
   * onFailure - モジュールがクラッシュした場合、Edge エージェントがモジュールを再起動します。 モジュールがクリーンにシャット ダウンした場合、Edge エージェントはモジュールを再起動しません。
   * Unhealthy - モジュールがクラッシュしたか、異常と判断された場合は、Edge エージェントによって再起動されます。
   * Always - モジュールがクラッシュしたか、異常と判断された場合、あるいは方法に関係なくシャットダウンされた場合は、Edge エージェントによって再起動されます。 
   
### <a name="security"></a>セキュリティ

IoT Edge エージェントは、IoT Edge デバイスのセキュリティ上、重要な役割を果たします。 たとえば、起動前のモジュール イメージの検証などの操作を実行します。 これらの機能は、V2 機能の一般提供時に追加されます。 

<!-- For more information about the Azure IoT Edge security framework, see []. -->

## <a name="next-steps"></a>次のステップ

- [Azure IoT Edge モジュールについて][lnk モジュール]

<!-- Images -->
[1]: ./media/iot-edge-runtime/pipeline.png
[2]: ./media/iot-edge-runtime/gateway.png
[3]: ./media/iot-edge-runtime/ModuleEndpoints.png
[4]: ./media/iot-edge-runtime/ModuleEndpointsWithRoutes.png

<!-- Links -->
[lnk モジュール]: iot-edge-modules.md