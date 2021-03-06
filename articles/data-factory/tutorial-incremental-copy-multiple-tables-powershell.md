---
title: "Azure Data Factory を使って複数のテーブルを増分コピーする | Microsoft Docs"
description: "このチュートリアルでは、オンプレミスの SQL Server データベースにある複数のテーブルから Azure SQL Database に差分データを増分コピーする Azure Data Factory パイプラインを作成します。 "
services: data-factory
documentationcenter: 
author: linda33wj
manager: jhubbard
editor: spelluru
ms.service: data-factory
ms.workload: data-services
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: get-started-article
ms.date: 12/01/2017
ms.author: jingwang
ms.openlocfilehash: 5472c41af9b0c524b931ed3e6b149270e17a633a
ms.sourcegitcommit: 5d3e99478a5f26e92d1e7f3cec6b0ff5fbd7cedf
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 12/06/2017
---
# <a name="incrementally-load-data-from-multiple-tables-in-sql-server-to-azure-sql-database"></a>SQL Server にある複数のテーブルから Azure SQL Database にデータを増分読み込みする
このチュートリアルでは、オンプレミスの SQL Server にある複数のテーブルから Azure SQL Database に差分データを読み込むパイプラインを使用して Azure Data Factory を作成します。    

このチュートリアルでは、以下の手順を実行します。

> [!div class="checklist"]
> * ソース データ ストアとターゲット データ ストアを準備します。
> * データ ファクトリを作成します。
> * 自己ホスト型統合ランタイム (IR) を作成します。
> * 統合ランタイムのインストール 
> * リンクされたサービスを作成します。 
> * ソース データセット、シンク データセット、基準値データセットを作成します。
> * パイプラインを作成、実行、監視します。
> * 結果の確認
> * ソース テーブルのデータを追加または更新します。
> * パイプラインを再実行して監視します。
> * 最終結果を確認します。 

> [!NOTE]
> この記事は、現在プレビュー段階にある Data Factory のバージョン 2 に適用されます。 一般公開 (GA) されている Data Factory サービスのバージョン 1 を使用している場合は、[Data Factory バージョン 1 のドキュメント](v1/data-factory-copy-data-from-azure-blob-storage-to-sql-database.md)を参照してください。

## <a name="overview"></a>概要
このソリューションを作成するにあたっての重要な手順を次に示します。 

1. **基準値列を選択する**。
    ソース データ ストアのテーブルごとに、いずれか 1 つの列を選択します。この列は、実行ごとに新しいレコードまたは更新されたレコードを特定する目的で使用されます。 通常、行が作成または更新されたときに常にデータが増える列を選択します (last_modify_time、ID など)。 この列の最大値が基準値として使用されます。
2. **基準値を格納するためのデータ ストアを準備する**。   
    このチュートリアルでは、Azure SQL データベースに基準値を格納します。
3. **次のアクティビティを含んだパイプラインを作成する**。 
    
    1. パイプラインにパラメーターとして渡された一連のソース テーブル名を反復処理する **ForEach** アクティビティを作成する。 このアクティビティが、ソース テーブルごとに次のアクティビティを呼び出して各テーブルの差分読み込みを実行します。 
    2. 2 つの**ルックアップ** アクティビティを作成する。 1 つ目のルックアップ アクティビティは、前回の基準値を取得するために使用します。 2 つ目のルックアップ アクティビティは、新しい基準値を取得するために使用します。 これらの基準値は、コピー アクティビティに渡されます。 
    3. **コピー アクティビティ**を作成する。これは基準値列の値がかつての基準値より大きく、かつ新しい基準値よりも小さい行をソース データ ストアからコピーするアクティビティです。 その差分データが、ソース データ ストアから新しいファイルとして BLOB ストレージにコピーされます。 
    4. **ストアド プロシージャ アクティビティ**を作成する。これはパイプラインの次回実行に備えて基準値を更新するためのアクティビティです。 

        次に示したのは、このソリューションの概略図です。 

        ![データの増分読み込み](media\tutorial-incremental-copy-multiple-tables-powershell\high-level-solution-diagram.png)


Azure サブスクリプションをお持ちでない場合は、開始する前に[無料](https://azure.microsoft.com/free/)アカウントを作成してください。

## <a name="prerequisites"></a>前提条件
* **SQL Server**。 このチュートリアルでは、オンプレミスの SQL Server データベースを**ソース** データ ストアとして使用します。 
* **Azure SQL データベース**。 **シンク** データ ストアとして Azure SQL Database を使用します。 Azure SQL データベースがない場合は、[Azure SQL データベースの作成](../sql-database/sql-database-get-started-portal.md)に関する記事に書かれている手順を参照して作成してください。 

### <a name="create-source-tables-in-your-sql-server-database"></a>SQL Server データベースにソース テーブルを作成する

1. **SQL Server Management Studio** を起動し、オンプレミスの SQL Server に接続します。 
2. **サーバー エクスプローラー**で目的のデータベースを右クリックして **[新しいクエリ]** を選択します。
3. データベースに対して次の SQL コマンドを実行し、`customer_table` および `project_table` という名前のテーブルを作成します。

    ```sql
    create table customer_table
    (
        PersonID int,
        Name varchar(255),
        LastModifytime datetime
    );
    
    create table project_table
    (
        Project varchar(255),
        Creationtime datetime
    );
        
    INSERT INTO customer_table
    (PersonID, Name, LastModifytime)
    VALUES
    (1, 'John','9/1/2017 12:56:00 AM'),
    (2, 'Mike','9/2/2017 5:23:00 AM'),
    (3, 'Alice','9/3/2017 2:36:00 AM'),
    (4, 'Andy','9/4/2017 3:21:00 AM'),
    (5, 'Anny','9/5/2017 8:06:00 AM');
    
    INSERT INTO project_table
    (Project, Creationtime)
    VALUES
    ('project1','1/1/2015 0:00:00 AM'),
    ('project2','2/2/2016 1:23:00 AM'),
    ('project3','3/4/2017 5:16:00 AM');
    
    ```

### <a name="create-destination-tables-in-your-azure-sql--database"></a>Azure SQL データベースにターゲット テーブルを作成する
1. **SQL Server Management Studio** を起動し、Azure SQL Server に接続します。 
2. **サーバー エクスプローラー**で目的の**データベース**を右クリックして **[新しいクエリ]** を選択します。
3. Azure SQL データベースに対して次の SQL コマンドを実行し、`customer_table` および `project_table` という名前のテーブルを作成します。  
    
    ```sql
    create table customer_table
    (
        PersonID int,
        Name varchar(255),
        LastModifytime datetime
    );
    
    create table project_table
    (
        Project varchar(255),
        Creationtime datetime
    );

    ```

### <a name="create-another-table-in-azure-sql-database-to-store-the-high-watermark-value"></a>高基準値の格納用としてもう 1 つテーブルを Azure SQL データベースに作成する
1. Azure SQL データベースに対して次の SQL コマンドを実行し、基準値の格納先として `watermarktable` という名前のテーブルを作成します。  
    
    ```sql
    create table watermarktable
    (
    
        TableName varchar(255),
        WatermarkValue datetime,
    );
    ```
3. 両方のソース テーブルの初期基準値を基準値テーブルに挿入します。

    ```sql

    INSERT INTO watermarktable
    VALUES 
    ('customer_table','1/1/2010 12:00:00 AM'),
    ('project_table','1/1/2010 12:00:00 AM');
    
    ```

### <a name="create-a-stored-procedure-in-azure-sql-database"></a>Azure SQL データベースにストアド プロシージャを作成する 

次のコマンドを実行して、Azure SQL データベースにストアド プロシージャを作成します。 パイプラインの実行後は都度、このストアド プロシージャによって基準値が更新されます。 

```sql
CREATE PROCEDURE sp_write_watermark @LastModifiedtime datetime, @TableName varchar(50)
AS

BEGIN

    UPDATE watermarktable
    SET [WatermarkValue] = @LastModifiedtime 
WHERE [TableName] = @TableName

END

```

### <a name="create-data-types-and-additional-stored-procedures"></a>データ型と新たなストアド プロシージャの作成
次のクエリを実行して、2 つのストアド プロシージャと 2 つのデータ型を Azure SQL Database に作成します。これらは、各ソース テーブルからのデータをターゲット テーブルにマージする際に必要となります。

```sql
CREATE TYPE DataTypeforCustomerTable AS TABLE(
    PersonID int,
    Name varchar(255),
    LastModifytime datetime
);

GO

CREATE PROCEDURE sp_upsert_customer_table @customer_table DataTypeforCustomerTable READONLY
AS

BEGIN
  MERGE customer_table AS target
  USING @customer_table AS source
  ON (target.PersonID = source.PersonID)
  WHEN MATCHED THEN
      UPDATE SET Name = source.Name,LastModifytime = source.LastModifytime
  WHEN NOT MATCHED THEN
      INSERT (PersonID, Name, LastModifytime)
      VALUES (source.PersonID, source.Name, source.LastModifytime);
END

GO

CREATE TYPE DataTypeforProjectTable AS TABLE(
    Project varchar(255),
    Creationtime datetime
);

GO

CREATE PROCEDURE sp_upsert_project_table @project_table DataTypeforProjectTable READONLY
AS

BEGIN
  MERGE project_table AS target
  USING @project_table AS source
  ON (target.Project = source.Project)
  WHEN MATCHED THEN
      UPDATE SET Creationtime = source.Creationtime
  WHEN NOT MATCHED THEN
      INSERT (Project, Creationtime)
      VALUES (source.Project, source.Creationtime);
END

```

[!INCLUDE [data-factory-quickstart-prerequisites-2](../../includes/data-factory-quickstart-prerequisites-2.md)]

## <a name="create-a-data-factory"></a>Data Factory を作成する。
1. 後で PowerShell コマンドで使用できるように、リソース グループ名の変数を定義します。 次のコマンド テキストを PowerShell にコピーし、[Azure リソース グループ](../azure-resource-manager/resource-group-overview.md)の名前を二重引用符で囲んで指定し、コマンドを実行します。 (例: `"adfrg"`)。 
   
     ```powershell
    $resourceGroupName = "ADFTutorialResourceGroup";
    ```

    リソース グループが既に存在する場合、上書きしないようお勧めします。 `$resourceGroupName` 変数に別の値を割り当てて、コマンドをもう一度実行します。
2. データ ファクトリの場所の変数を定義します。 

    ```powershell
    $location = "East US"
    ```
3. Azure リソース グループを作成するには、次のコマンドを実行します。 

    ```powershell
    New-AzureRmResourceGroup $resourceGroupName $location
    ``` 
    リソース グループが既に存在する場合、上書きしないようお勧めします。 `$resourceGroupName` 変数に別の値を割り当てて、コマンドをもう一度実行します。 
3. データ ファクトリ名の変数を定義します。 

    > [!IMPORTANT]
    >  データ ファクトリ名は、グローバルに一意となるように更新してください。 たとえば、ADFIncMultiCopyTutorialFactorySP1127 とします。 

    ```powershell
    $dataFactoryName = "ADFIncMultiCopyTutorialFactory";
    ```
5. データ ファクトリを作成するには、次の **Set-AzureRmDataFactoryV2** コマンドレットを実行します。 
    
    ```powershell       
    Set-AzureRmDataFactoryV2 -ResourceGroupName $resourceGroupName -Location $location -Name $dataFactoryName 
    ```

以下の点に注意してください。

* Azure Data Factory の名前はグローバルに一意にする必要があります。 次のエラーが発生した場合は、名前を変更してからもう一度実行してください。

    ```
    The specified Data Factory name 'ADFIncMultiCopyTutorialFactory' is already in use. Data Factory names must be globally unique.
    ```
* Data Factory インスタンスを作成するには、Azure へのログインに使用するユーザー アカウントが、**共同作成者**または**所有者**ロールのメンバーであるか、Azure サブスクリプションの**管理者**である必要があります。
* 現在、Data Factory バージョン 2 でデータ ファクトリを作成できるリージョンは、米国東部、米国東部 2、および西ヨーロッパだけです。 データ ファクトリで使用するデータ ストア (Azure Storage、Azure SQL Database など) やコンピューティング (HDInsight など) は他のリージョンに配置できます。

[!INCLUDE [data-factory-create-install-integration-runtime](../../includes/data-factory-create-install-integration-runtime.md)]



## <a name="create-linked-services"></a>リンクされたサービスを作成します
データ ストアおよびコンピューティング サービスをデータ ファクトリにリンクするには、リンクされたサービスをデータ ファクトリに作成します。 このセクションでは、オンプレミスの SQL Server データベースと Azure SQL データベースに対するリンクされたサービスを作成します。 

### <a name="create-sql-server-linked-service"></a>SQL Server のリンクされているサービスを作成する
この手順では、オンプレミス SQL Server をデータ ファクトリにリンクします。

1. 次の内容を含む **SqlServerLinkedService.json** という名前の JSON ファイルを **C:\ADFTutorials\IncCopyMultiTableTutorial** フォルダーに作成します。SQL Server への接続に使用する**認証**に基づいて、右側のセクションを選択します。 ローカル フォルダーがまだ存在しない場合は新たに作成してください。 

    > [!IMPORTANT]
    > SQL Server への接続に使用する**認証**に基づいて、右側のセクションを選択します。

    **SQL 認証 (sa) を使用している場合は、次の JSON 定義をコピーします。**

    ```json
    {
        "properties": {
            "type": "SqlServer",
            "typeProperties": {
                "connectionString": {
                    "type": "SecureString",
                    "value": "Server=<servername>;Database=<databasename>;User ID=<username>;Password=<password>;Timeout=60"
                }
            },
            "connectVia": {
                "type": "integrationRuntimeReference",
                "referenceName": "<integration runtime name>"
            }
        },
        "name": "SqlServerLinkedService"
    }
   ```    
    **Windows 認証を使用している場合は、次の JSON 定義をコピーします。**

    ```json
    {
        "properties": {
            "type": "SqlServer",
            "typeProperties": {
                "connectionString": {
                    "type": "SecureString",
                    "value": "Server=<server>;Database=<database>;Integrated Security=True"
                },
                "userName": "<user> or <domain>\\<user>",
                "password": {
                    "type": "SecureString",
                    "value": "<password>"
                }
            },
            "connectVia": {
                "type": "integrationRuntimeReference",
                "referenceName": "<integration runtime name>"
            }
        },
        "name": "SqlServerLinkedService"
    }    
    ```
    > [!IMPORTANT]
    > - SQL Server への接続に使用する**認証**に基づいて、右側のセクションを選択します。
    > - **&lt;integration** **runtime** **name>** は、実際の統合ランタイムの名前に置き換えます。
    > - **&lt;servername>**、**&lt;databasename>**、**&lt;username>**、**&lt;password>** を実際の SQL Server の値に置き換えてから、ファイルを保存してください。
    > - ユーザー アカウントまたはサーバー名にスラッシュ文字 (`\`) を使用する必要がある場合は、エスケープ文字 (`\`) を使用します。 たとえば、「 `mydomain\\myuser`」のように入力します。

2. **Azure PowerShell** で **C:\ADFTutorials\IncCopyMultiTableTutorial** フォルダーに切り替えます。
3. **Set-AzureRmDataFactoryV2LinkedService** コマンドレットを実行して、リンクされたサービス **AzureStorageLinkedService** を作成します。 次の例では、**ResourceGroupName** パラメーターと **DataFactoryName** パラメーターの値を渡しています。 

    ```powershell
    Set-AzureRmDataFactoryV2LinkedService -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -Name "SqlServerLinkedService" -File ".\SqlServerLinkedService.json"
    ```

    出力例を次に示します。

    ```json
    LinkedServiceName : SqlServerLinkedService
    ResourceGroupName : ADFTutorialResourceGroup
    DataFactoryName   : ADFIncMultiCopyTutorialFactory1201
    Properties        : Microsoft.Azure.Management.DataFactory.Models.SqlServerLinkedService
    ```

### <a name="create-azure-sql-database-linked-service"></a>Azure SQL Database のリンクされたサービスを作成する
1. 以下の内容を記述した **AzureSQLDatabaseLinkedService.json** という名前の JSON ファイルを **C:\ADFTutorials\IncCopyMultiTableTutorial** フォルダー内に作成します (ADF フォルダーがない場合は作成します)。 **&lt;server&gt;、&lt;database name&gt;、&lt;user id&gt;、&lt;password&gt;** を実際の Azure SQL サーバーの名前、データベースの名前、ユーザー ID、パスワードに置き換えてからファイルを保存してください。 

    ```json
    {
        "name": "AzureSQLDatabaseLinkedService",
        "properties": {
            "type": "AzureSqlDatabase",
            "typeProperties": {
                "connectionString": {
                    "value": "Server = tcp:<server>.database.windows.net,1433;Initial Catalog=<database name>; Persist Security Info=False; User ID=<user name>; Password=<password>; MultipleActiveResultSets = False; Encrypt = True; TrustServerCertificate = False; Connection Timeout = 30;",
                    "type": "SecureString"
                }
            }
        }
    }
    ```
2. **Azure PowerShell** で **Set-AzureRmDataFactoryV2LinkedService** コマンドレットを実行して、リンクされたサービス **AzureSQLDatabaseLinkedService** を作成します。 

    ```powershell
    Set-AzureRmDataFactoryV2LinkedService -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -Name "AzureSQLDatabaseLinkedService" -File ".\AzureSQLDatabaseLinkedService.json"
    ```

    出力例を次に示します。

    ```json
    LinkedServiceName : AzureSQLDatabaseLinkedService
    ResourceGroupName : ADFTutorialResourceGroup
    DataFactoryName   : ADFIncMultiCopyTutorialFactory1201
    Properties        : Microsoft.Azure.Management.DataFactory.Models.AzureSqlDatabaseLinkedService
    ```

## <a name="create-datasets"></a>データセットを作成する
この手順では、データのコピー元とコピー先、 基準値の格納場所を表すデータセットを作成します。

### <a name="create-a-source-dataset"></a>ソース データセットを作成する

1. 以下の内容を記述した **SourceDataset.json** という名前の JSON ファイルを同じフォルダー内に作成します。 

    ```json
    {
        "name": "SourceDataset",
        "properties": {
            "type": "SqlServerTable",
            "typeProperties": {
                "tableName": "dummyName"
            },
            "linkedServiceName": {
                "referenceName": "SqlServerLinkedService",
                "type": "LinkedServiceReference"
            }
        }
    }
   
    ```

    テーブル名はダミーです。 このパイプライン内のコピー アクティビティは、テーブル全体を読み込むことはせずに、SQL クエリを使用してデータを読み込みます。 
1.  Set-AzureRmDataFactoryV2Dataset コマンドレットを実行して SourceDataset というデータセットを作成します。
    
    ```powershell
    Set-AzureRmDataFactoryV2Dataset -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -Name "SourceDataset" -File ".\SourceDataset.json"
    ```

    このコマンドレットの出力例を次に示します。
    
    ```json
    DatasetName       : SourceDataset
    ResourceGroupName : ADFTutorialResourceGroup
    DataFactoryName   : ADFIncMultiCopyTutorialFactory1201
    Structure         :
    Properties        : Microsoft.Azure.Management.DataFactory.Models.SqlServerTableDataset
    ```

### <a name="create-a-sink-dataset"></a>シンク データセットを作成する

1. 以下の内容を記述した **SinkDataset.json** という名前の JSON ファイルを同じフォルダー内に作成します。tableName は、パイプラインによって実行時に動的に設定されます。 パイプライン内の ForEach アクティビティは、一連のテーブル名を反復処理しながら、各イテレーションの中でこのデータセットにテーブル名を渡します。 

    ```json
    {
        "name": "SinkDataset",
        "properties": {
            "type": "AzureSqlTable",
            "typeProperties": {
                "tableName": {
                    "value": "@{dataset().SinkTableName}",
                    "type": "Expression"
                }
            },
            "linkedServiceName": {
                "referenceName": "AzureSQLDatabaseLinkedService",
                "type": "LinkedServiceReference"
            },
            "parameters": {
                "SinkTableName": {
                    "type": "String"
                }
            }
        }
    }
    ```

2.  Set-AzureRmDataFactoryV2Dataset コマンドレットを実行して SinkDataset というデータセットを作成します。
    
    ```powershell
    Set-AzureRmDataFactoryV2Dataset -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -Name "SinkDataset" -File ".\SinkDataset.json"
    ```

    このコマンドレットの出力例を次に示します。
    
    ```json
    DatasetName       : SinkDataset
    ResourceGroupName : ADFTutorialResourceGroup
    DataFactoryName   : ADFIncMultiCopyTutorialFactory1201
    Structure         :
    Properties        : Microsoft.Azure.Management.DataFactory.Models.AzureSqlTableDataset
    ```

### <a name="create-a-dataset-for-watermark"></a>基準値用のデータセットを作成する
この手順では、高基準値を格納するためのデータセットを作成します。 

1. 以下の内容を記述した WatermarkDataset.json という名前の JSON ファイルを同じフォルダー内に作成します。 

    ```json
    {
        "name": " WatermarkDataset ",
        "properties": {
            "type": "AzureSqlTable",
            "typeProperties": {
                "tableName": "watermarktable"
            },
            "linkedServiceName": {
                "referenceName": "AzureSQLDatabaseLinkedService",
                "type": "LinkedServiceReference"
            }
        }
    }    
    ```
2.  Set-AzureRmDataFactoryV2Dataset コマンドレットを実行して WatermarkDataset というデータセットを作成します。
    
    ```powershell
    Set-AzureRmDataFactoryV2Dataset -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -Name "WatermarkDataset" -File ".\WatermarkDataset.json"
    ```

    このコマンドレットの出力例を次に示します。
    
    ```json
    DatasetName       : WatermarkDataset
    ResourceGroupName : ADFTutorialResourceGroup
    DataFactoryName   : <data factory name>
    Structure         :
    Properties        : Microsoft.Azure.Management.DataFactory.Models.AzureSqlTableDataset    
    ```

## <a name="create-a-pipeline"></a>パイプラインを作成する。
このパイプラインは、一連のテーブル名をパラメーターとして受け取ります。 **ForEach アクティビティ**は、一連のテーブル名を反復処理しながら、次の操作を実行します。 

1. **ルックアップ アクティビティ**を使用して古い基準値 (初期値または前回のイテレーションで使用された値) を取得します。
2. **ルックアップ アクティビティ**を使用して新しい基準値 (ソース テーブルの基準値列の最大値) を取得します。
3. **コピー アクティビティ**を使用して、この 2 つの基準値の間に存在するデータをソース データベースからターゲット データベースにコピーします。 
4. **ストアド プロシージャ アクティビティ**を使用して古い基準値を更新します。この値が、次のイテレーションの最初のステップで使用されます。 

### <a name="create-the-pipeline"></a>パイプラインを作成する
1. 以下の内容を記述した IncrementalCopyPipeline.json という名前の JSON ファイルを同じフォルダー内に作成します。 

    ```json
    {
        "name": "IncrementalCopyPipeline",
        "properties": {
            "activities": [{
    
                "name": "IterateSQLTables",
                "type": "ForEach",
                "typeProperties": {
                    "isSequential": "false",
                    "items": {
                        "value": "@pipeline().parameters.tableList",
                        "type": "Expression"
                    },
    
                    "activities": [
                        {
                            "name": "LookupOldWaterMarkActivity",
                            "type": "Lookup",
                            "typeProperties": {
                                "source": {
                                    "type": "SqlSource",
                                    "sqlReaderQuery": "select * from watermarktable where TableName  =  '@{item().TABLE_NAME}'"
                                },
    
                                "dataset": {
                                    "referenceName": "WatermarkDataset",
                                    "type": "DatasetReference"
                                }
                            }
                        },
                        {
                            "name": "LookupNewWaterMarkActivity",
                            "type": "Lookup",
                            "typeProperties": {
                                "source": {
                                    "type": "SqlSource",
                                    "sqlReaderQuery": "select MAX(@{item().WaterMark_Column}) as NewWatermarkvalue from @{item().TABLE_NAME}"
                                },
    
                                "dataset": {
                                    "referenceName": "SourceDataset",
                                    "type": "DatasetReference"
                                }
                            }
                        },
    
                        {
                            "name": "IncrementalCopyActivity",
                            "type": "Copy",
                            "typeProperties": {
                                "source": {
                                    "type": "SqlSource",
                                    "sqlReaderQuery": "select * from @{item().TABLE_NAME} where @{item().WaterMark_Column} > '@{activity('LookupOldWaterMarkActivity').output.firstRow.WatermarkValue}' and @{item().WaterMark_Column} <= '@{activity('LookupNewWaterMarkActivity').output.firstRow.NewWatermarkvalue}'"
                                },
                                "sink": {
                                    "type": "SqlSink",
                                    "SqlWriterTableType": "@{item().TableType}",
                                    "SqlWriterStoredProcedureName": "@{item().StoredProcedureNameForMergeOperation}"
                                }
                            },
                            "dependsOn": [{
                                    "activity": "LookupNewWaterMarkActivity",
                                    "dependencyConditions": [
                                        "Succeeded"
                                    ]
                                },
                                {
                                    "activity": "LookupOldWaterMarkActivity",
                                    "dependencyConditions": [
                                        "Succeeded"
                                    ]
                                }
                            ],
    
                            "inputs": [{
                                "referenceName": "SourceDataset",
                                "type": "DatasetReference"
                            }],
                            "outputs": [{
                                "referenceName": "SinkDataset",
                                "type": "DatasetReference",
                                "parameters": {
                                    "SinkTableName": "@{item().TABLE_NAME}"
                                }
                            }]
                        },
    
                        {
                            "name": "StoredProceduretoWriteWatermarkActivity",
                            "type": "SqlServerStoredProcedure",
                            "typeProperties": {
    
                                "storedProcedureName": "sp_write_watermark",
                                "storedProcedureParameters": {
                                    "LastModifiedtime": {
                                        "value": "@{activity('LookupNewWaterMarkActivity').output.firstRow.NewWatermarkvalue}",
                                        "type": "datetime"
                                    },
                                    "TableName": {
                                        "value": "@{activity('LookupOldWaterMarkActivity').output.firstRow.TableName}",
                                        "type": "String"
                                    }
                                }
                            },
    
                            "linkedServiceName": {
                                "referenceName": "AzureSQLDatabaseLinkedService",
                                "type": "LinkedServiceReference"
                            },
    
                            "dependsOn": [{
                                "activity": "IncrementalCopyActivity",
                                "dependencyConditions": [
                                    "Succeeded"
                                ]
                            }]
                        }
    
                    ]
    
                }
            }],
    
            "parameters": {
                "tableList": {
                    "type": "Object"
                }
            }
        }
    }
    ```
2. Set-AzureRmDataFactoryV2Pipeline コマンドレットを実行して、パイプライン IncrementalCopyPipeline を作成します。
    
   ```powershell
   Set-AzureRmDataFactoryV2Pipeline -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -Name "IncrementalCopyPipeline" -File ".\IncrementalCopyPipeline.json"
   ``` 

   出力例を次に示します。 

   ```json
    PipelineName      : IncrementalCopyPipeline
    ResourceGroupName : ADFTutorialResourceGroup
    DataFactoryName   : ADFIncMultiCopyTutorialFactory1201
    Activities        : {IterateSQLTables}
    Parameters        : {[tableList, Microsoft.Azure.Management.DataFactory.Models.ParameterSpecification]}
   ```
 
## <a name="run-the-pipeline"></a>パイプラインを実行する

1. 次の内容を記述した **Parameters.json** というパラメーター ファイルを同じフォルダーに作成します。

    ```json
    {
        "tableList": 
        [
            {
                "TABLE_NAME": "customer_table",
                "WaterMark_Column": "LastModifytime",
                "TableType": "DataTypeforCustomerTable",
                "StoredProcedureNameForMergeOperation": "sp_upsert_customer_table"
            },
            {
                "TABLE_NAME": "project_table",
                "WaterMark_Column": "Creationtime",
                "TableType": "DataTypeforProjectTable",
                "StoredProcedureNameForMergeOperation": "sp_upsert_project_table"
            }
        ]
    }
    ```
2. **Invoke-AzureRmDataFactoryV2Pipeline** コマンドレットを使って **IncrementalCopyPipeline** パイプラインを実行します。 プレースホルダーはそれぞれ実際のリソース グループとデータ ファクトリ名に置き換えてください。

    ```powershell
    $RunId = Invoke-AzureRmDataFactoryV2Pipeline -PipelineName "IncrementalCopyPipeline" -ResourceGroup $resourceGroupName -dataFactoryName $dataFactoryName -ParameterFile ".\Parameters.json"        
    ``` 

## <a name="monitor-the-pipeline"></a>パイプラインの監視

1. [Azure Portal](https://portal.azure.com) にログインします。
2. **[その他のサービス]** をクリックし、キーワード「`data factories`」で検索して、**[データ ファクトリ]** を選択します。 

    ![[データ ファクトリ] メニュー](media\tutorial-incremental-copy-multiple-tables-powershell\monitor-data-factories-menu-1.png)
3. データ ファクトリの一覧から**目的のデータ ファクトリ**を探して選択し、[データ ファクトリ] ページを起動します。 

    ![目的のデータ ファクトリの検索](media\tutorial-incremental-copy-multiple-tables-powershell\monitor-search-data-factory-2.png)
4. [データ ファクトリ] ページの **[監視と管理]** タイルをクリックします。 

    ![Monitor & Manage tile](media\tutorial-incremental-copy-multiple-tables-powershell\monitor-monitor-manage-tile-3.png)    
5. 独立したタブで**データ統合アプリケーション**が起動します。すべての**パイプラインの実行**とその状態を確認できます。 次の例では、パイプラインの実行が、**成功**状態であることに注目してください。 **[パラメーター]** 列のリンクをクリックすると、パイプラインに渡されたパラメーターを確認できます。 エラーが発生した場合は、**[エラー]** 列にリンクが表示されます。 **[アクション]** 列のリンクをクリックします。 

    ![パイプライン実行](media\tutorial-incremental-copy-multiple-tables-powershell\monitor-pipeline-runs-4.png)    
6. **[アクション]** 列のリンクをクリックすると、以下のページが開いて、そのパイプラインに関するすべての**アクティビティの実行**が表示されます。 

    ![アクティビティの実行](media\tutorial-incremental-copy-multiple-tables-powershell\monitor-activity-runs-5.png)
7. 再度**パイプラインの実行**ビューに切り替えるには、この画像に示した **[パイプライン]** をクリックします。 

## <a name="review-the-results"></a>結果の確認
SQL Server Management Studio からターゲット Azure SQL Database に対して次のクエリを実行し、ソース テーブルからターゲット テーブルにデータがコピーされていることを確かめます。 

**クエリ** 
```sql
select * from customer_table
```

**Output**
```
===========================================
PersonID    Name    LastModifytime
===========================================
1           John    2017-09-01 00:56:00.000
2           Mike    2017-09-02 05:23:00.000
3           Alice   2017-09-03 02:36:00.000
4           Andy    2017-09-04 03:21:00.000
5           Anny    2017-09-05 08:06:00.000
```

**クエリ:**

```sql
select * from project_table
```

**Output**

```
===================================
Project     Creationtime
===================================
project1    2015-01-01 00:00:00.000
project2    2016-02-02 01:23:00.000
project3    2017-03-04 05:16:00.000
```

**クエリ**

```sql
select * from watermarktable
```

**Output**

```
======================================
TableName       WatermarkValue
======================================
customer_table  2017-09-05 08:06:00.000
project_table   2017-03-04 05:16:00.000
```

2 つのテーブルの基準値が更新されていることが確認できます。 

## <a name="add-more-data-to-the-source-tables"></a>ソース テーブルにデータを追加する

コピー元の SQL Server データベースに次のクエリを実行して、customer_table 内の既存の行を更新し、project_table に新しい行を挿入します。 

```sql
UPDATE customer_table
SET [LastModifytime] = '2017-09-08T00:00:00Z', [name]='NewName' where [PersonID] = 3

INSERT INTO project_table
(Project, Creationtime)
VALUES
('NewProject','10/1/2017 0:00:00 AM');
``` 

## <a name="rerun-the-pipeline"></a>パイプラインを再実行する

1. 今度は、次の PowerShell コマンドを実行してパイプラインを再実行します。

    ```powershell
    $RunId = Invoke-AzureRmDataFactoryV2Pipeline -PipelineName "IncrementalCopyPipeline" -ResourceGroup $resourceGroupname -dataFactoryName $dataFactoryName -ParameterFile ".\Parameters.json"
    ```
2. 「[パイプラインの監視](#monitor-the-pipeline)」セクションの手順に従ってパイプラインの実行を監視します。 パイプラインが**進行中**の状態にあるため、**[アクション]** には、パイプラインの実行をキャンセルするためのアクション リンクが別途表示されています。 

    ![パイプライン実行](media\tutorial-incremental-copy-multiple-tables-powershell\monitor-pipeline-runs-6.png)    
3. パイプラインの実行に成功するまで、**[Refresh]\(最新の情報に更新\)** をクリックして一覧を更新します。 

    ![パイプライン実行](media\tutorial-incremental-copy-multiple-tables-powershell\monitor-pipeline-runs-succeded-7.png)
4. (省略可) [アクション] の **[View Activity Runs]\(アクティビティの実行の表示\)** リンク (アイコン) をクリックすると、このパイプラインの実行に関連付けられているアクティビティの実行がすべて表示されます。 

## <a name="review-final-results"></a>最終結果を確認します。
SQL Server Management Studio からターゲット データベースに対して次のクエリを実行し、更新されたデータや新しいデータがソース テーブルからターゲット テーブルにコピーされていることを確かめます。 

**クエリ** 
```sql
select * from customer_table
```

**Output**
```
===========================================
PersonID    Name    LastModifytime
===========================================
1           John    2017-09-01 00:56:00.000
2           Mike    2017-09-02 05:23:00.000
3           NewName 2017-09-08 00:00:00.000
4           Andy    2017-09-04 03:21:00.000
5           Anny    2017-09-05 08:06:00.000
```

PersonID 3 を見ると、Name と LastModifytime が新しい値であることがわかります。 

**クエリ:**

```sql
select * from project_table
```

**Output**

```
===================================
Project     Creationtime
===================================
project1    2015-01-01 00:00:00.000
project2    2016-02-02 01:23:00.000
project3    2017-03-04 05:16:00.000
NewProject  2017-10-01 00:00:00.000
```

project_table に NewProject というエントリが追加されていることに注目してください。 

**クエリ**

```sql
select * from watermarktable
```

**Output**

```
======================================
TableName       WatermarkValue
======================================
customer_table  2017-09-08 00:00:00.000
project_table   2017-10-01 00:00:00.000
```

2 つのテーブルの基準値が更新されていることが確認できます。
     
## <a name="next-steps"></a>次のステップ
このチュートリアルでは、以下の手順を実行しました。 

> [!div class="checklist"]
> * ソース データ ストアとターゲット データ ストアを準備します。
> * データ ファクトリを作成します。
> * 自己ホスト型統合ランタイム (IR) を作成します。
> * 統合ランタイムのインストール 
> * リンクされたサービスを作成します。 
> * ソース データセット、シンク データセット、基準値データセットを作成します。
> * パイプラインを作成、実行、監視します。
> * 結果の確認
> * ソース テーブルのデータを追加または更新します。
> * パイプラインを再実行して監視します。
> * 最終結果を確認します。 

次のチュートリアルに進み、Azure 上の Spark クラスターを使ってデータを変換する方法について学習しましょう。

> [!div class="nextstepaction"]
>[Change Tracking テクノロジを使用して Azure SQL Database から Azure Blob Storage にデータを増分読み込みする](tutorial-incremental-copy-multiple-tables-powershell.md)


