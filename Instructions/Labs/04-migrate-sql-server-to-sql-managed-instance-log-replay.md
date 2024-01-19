---
lab:
  title: 使用日志重播服务将 SQL Server 数据库迁移到 Azure SQL 托管实例
---

# 使用日志重播服务将 SQL Server 数据库迁移到 Azure SQL 托管实例

在本练习中，你将了解如何使用日志重播服务将 SQL Server 数据库迁移到 Azure SQL 托管实例。 

首先部署 Azure SQL 托管实例。 然后，你将使用日志重播服务执行 SQL Server 数据库到 Azure SQL 托管实例的联机迁移。 你还将了解如何在 PowerShell 中监视迁移过程。

本练习大约需要 **45** 分钟。

> **注意：** 要完成此练习，您需要访问 Azure 订阅以创建 Azure 资源。 如果没有 Azure 订阅，请在开始之前创建一个[免费](https://azure.microsoft.com/free/?azure-portal=true)帐户。

## 开始之前

要运行此练习，需要：

| 项 | 说明 |
| --- | --- |
| **目标服务器** | Azure SQL 托管实例。 我们将在本练习中创建它。|
| **源服务器** | 安装在服务器上的 SQL Server 2019 或[更新版本](https://www.microsoft.com/en-us/sql-server/sql-server-downloads)的实例。 |
| **源数据库** | 要在 SQL Server 实例上还原的轻量级 [AdventureWorks](https://learn.microsoft.com/sql/samples/adventureworks-install-configure) 数据库。 |
| **Azure Data Studio** | 在源数据库所在的同一服务器上安装 [Azure Data Studio](https://learn.microsoft.com/sql/azure-data-studio/download-azure-data-studio)。 如果已经安装，请进行更新，确保使用的是最新版本。 |

## 还原 SQL Server 数据库

让我们在 SQL Server 实例中还原 *AdventureWorksLT* 数据库。 此数据库可充当本实验室练习的源数据库。 如果数据库已经还原，则可以跳过这些步骤。

1. 选择 Windows 开始按钮，然后键入 SSMS。 从列表中选择**Microsoft SQL Server Management Studio 18**。  

1. 当 SSMS 打开时，请注意，“**连接到服务器**”对话框将使用默认实例名称预填充。 选择“连接” 。

1. 选择**数据库**文件夹，然后选择**新建查询**

1. 在新建查询窗口中，复制并粘贴以下 T-SQL。 执行查询以还原数据库。

    ```sql
    RESTORE DATABASE AdventureWorksLT
    FROM DISK = 'C:\LabFiles\AdventureWorksLT2019.bak'
    WITH RECOVERY,
          MOVE 'AdventureWorksLT2019_Data' 
            TO 'C:\LabFiles\AdventureWorksLT2019.mdf',
          MOVE 'AdventureWorksLT2019_Log'
            TO 'C:\LabFiles\AdventureWorksLT2019.ldf';
    ```

    > **注意**：确保上例中的数据库备份文件名称和路径与实际备份文件一致。 否则，命令可能会失败。

1. 还原完成后，应会看到一条成功消息。

## 部署 Azure SQL 托管实例

通过执行以下步骤来创建 Azure SQL 托管实例：

1. 登录到 [Azure 门户](https://portal.azure.com/learn.docs.microsoft.com?azure-portal=true)，然后在左上角选择“创建资源”。
1. 搜索“托管实例”，选择“Azure SQL 托管实例”，然后选择“创建”。
1. 使用下表中的信息填写 SQL 托管实例表单：

    |  | 建议的值 |
    |---|---|
    | **订阅** | 你的订阅。 |
    | 托管实例名称 | 任何有效的名称。 |
    | 托管实例管理员登录名 | 任何有效的用户名。 不要使用“serveradmin”，因为这是保留的服务器级角色。 |
    | **密码** | 任何长度超过 16 个字符且满足复杂性要求的密码。 |
    | **时区** | 托管实例要观察的时区。 |
    | **排序规则** | 要用于托管实例的排序规则。 如果从 SQL Server 迁移数据库，请使用 SELECT SERVERPROPERTY(N'Collation') 查看源排序规则，然后使用该值。 |
    | **位置** | 要在其中创建托管实例的 Azure 区域。 |
    | **虚拟网络** | 选择“创建新虚拟网络”或有效的虚拟网络和子网。 |
    | **启用公共终结点** | 选中此选项可启用公共终结点，然后帮助 Azure 外部的客户端访问数据库。 |
    | **允许的访问来源** | 从 Azure 服务、Internet 或无访问权限中选择。 |
    | **连接类型** | 选择“代理”或“重定向”连接类型。 |
    | **资源组** | 新的或现有的资源组。 |

1. 选择“定价层”，以调整计算和存储资源的大小并查看定价层选项。 
1. 完成后，选择“应用”以保存你的选择，然后选择“创建”以部署托管实例。
1. 选择“通知”图标以查看部署状态。
1. 选择“正在进行的部署”以打开“托管实例”窗口，进一步监视部署进度****。

## 创建 Azure Blob 存储帐户和容器

在 Azure SQL 托管实例所在同一区域中创建 Azure Blob 存储帐户。 这是用于存储数据库备份以进行迁移的位置。

1. 转到 [Azure 门户](https://portal.azure.com)并使用自己的帐户凭据登录。
1. 在左侧菜单中，选择“**所有服务**”并搜索 *“存储帐户”*。 选择“**存储帐户**”以打开“存储帐户”页。
1. 在“存储帐户”页上，选择“**+ 添加** ”以创建新的存储帐户。
1. 在“**创建存储帐户** ”页的“**基本信息**”选项卡中，选择要用于存储帐户的订阅。 然后选择包含 Azure SQL 托管实例的资源组。
1. 为存储帐户输入唯一名称。 
    
    > **注意：** 该名称长度必须为 3 到 24 个字符，并且只能包含小写字母和数字。

1. 选择 Azure SQL 托管实例所在的位置（区域）。
1. 选择存储帐户的性能层。
1. 选择“**BlobStorage**”作为存储帐户的帐户类型。 
1. 选择“**本地冗余存储 (LRS)** 作为存储帐户的复制选项。
1. 查看并选择“**查看 + 创建**”以创建存储帐户。
1. 创建存储帐户后，转到存储帐户页，并在左侧菜单中选择“**容器**”选项。 然后，选择“**+ 容器**”来新建容器。 输入容器名称并选择公共访问级别。 
1. 选择“**创建**”按钮以创建容器。

完成这些步骤后，将会在 Azure SQL 托管实例所在的同一区域中创建 Azure Blob 存储帐户，以及可以存储数据库备份以进行迁移的容器。

## 备份 SQL Server 数据库

我们将在 SQL Server 实例上创建 *AdventureWorksLT* 数据库的完整备份，然后创建启用了 `CHECKSUM` 的差异备份和日志备份。 

1. 选择 Windows 开始按钮，然后键入 SSMS。 从列表中选择**Microsoft SQL Server Management Studio 18**。  
1. 当 SSMS 打开时，请注意，“**连接到服务器**”对话框将使用默认实例名称预填充。 选择“连接”。
1. 选择**数据库**文件夹，然后选择**新建查询**
1. 在新建查询窗口中，将以下 T-SQL 复制并粘贴到其中。 执行查询以还原数据库。

    ```sql
    BACKUP DATABASE AdventureWorksLT
    TO DISK = 'C:\LabFiles\AdventureWorksLT_full.bak'
    WITH CHECKSUM;

    BACKUP DATABASE AdventureWorksLT
    TO DISK = 'C:\LabFiles\AdventureWorksLT_diff.dif'
    WITH DIFFERENTIAL, CHECKSUM;

    BACKUP LOG AdventureWorksLT
    TO DISK = 'C:\LabFiles\AdventureWorksLT_log.trn'
    WITH CHECKSUM;
    ```

    > **注意**：确保上述示例中的文件路径与实际文件路径匹配。 否则，命令可能会失败。

1. 还原完成后，应会看到一条成功消息。
1. 如果运行的是某个版本的 SQL Server （从 SQL Server 2012 SP1 CU2 和 SQL Server 2014 开始），则可以使用本机 SQL Server `BACKUP TO URL` 选项直接从 SQL Server 备份到 Blob 存储帐户。 

    ```sql
    CREATE CREDENTIAL [https://<mystorageaccountname>.blob.core.windows.net/<containername>] 
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE',  
    SECRET = '<SAS_TOKEN>';  
    GO
    
    -- Take a full database backup to a URL
    BACKUP DATABASE [AdventureWorksLT]
    TO URL = 'https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>/AdventureWorksLT_full.bak'
    WITH INIT, COMPRESSION, CHECKSUM
    GO
    
    -- Take a differential database backup to a URL
    BACKUP DATABASE [AdventureWorksLT]
    TO URL = 'https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>/AdventureWorksLT_diff.bak'  
    WITH DIFFERENTIAL, COMPRESSION, CHECKSUM
    GO
    
    -- Take a transactional log backup to a URL
    BACKUP LOG [AdventureWorksLT]
    TO URL = 'https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>/AdventureWorksLT_log.trn'  
    WITH COMPRESSION, CHECKSUM
    ```

    > **注意：** 如果决定使用此选项，则可以跳过接下来的“**将备份文件复制到 Azure 存储帐户**”部分。

## 将备份文件复制到 Azure 存储帐户

现在，我们将备份文件复制到之前创建的 Azure Blob 存储帐户。

1. 转到 [Azure 门户](https://portal.azure.com)并使用自己的帐户凭据登录。
1. 在左侧菜单中，选择“**存储帐户**”，然后选择之前创建的存储帐户。
1. 在存储帐户概述页中，向下滚动到“**Blob 服务**”部分并选择“**容器**”。 选择前面创建的容器。
1. 选择容器页顶部的“**上传**”。 在“**上传 Blob**”页中，选择“**文件夹**”以选择包含备份文件的文件夹，或选择“**文件**”以选择单个备份文件。 选择文件后，选择“**上传**”以启动上传过程。

## 验证访问权限

请务必验证 SQL Server 和 SQL 托管实例是否可以成功访问 Blob 存储帐户。 为此，请运行示例测试查询，以确定托管实例是否可以访问容器中的备份。

1. 通过 SSMS 在 SQL 托管实例上连接。
1. 打开新的查询编辑器，并运行该命令。

```sql
CREATE CREDENTIAL [https://<mystorageaccountname>.blob.core.windows.net/databases] 
WITH IDENTITY = 'SHARED ACCESS SIGNATURE' 
, SECRET = '<sastoken>' 

RESTORE HEADERONLY 
FROM URL = 'https://<mystorageaccountname>.blob.core.windows.net/<containername>/<backup_file_name>.bak'
```
1. 重复在 SQL Server 实例上连接的此过程。

## 使用日志重播服务还原备份文件

你将使用日志重播服务 (LRS) 将备份文件从 Azure Blob 存储还原到 Azure SQL 托管实例。 LRS 是基于 SQL Server 日志传送技术的免费服务。

1. 在存储帐户概述页中，向下滚动到“**Blob 服务**”部分并选择“**容器**”。 选择存储备份文件的容器。
1. 选择容器页顶部的“**生成 SAS**”。 在“**生成共享访问签名**”页中，选择要授予的权限，设置 SAS 令牌的开始时间和到期时间，然后选择“**生成 SAS 和连接字符串**”。 SAS 令牌将显示在“**SAS 令牌**”字段中，然后复制它。
1. 通过运行 `Connect-AzAccount` cmdlet 来使用 PowerShell 连接到 Azure 帐户。

    ```powershell
    Login-AzAccount
    Select-AzSubscription -SubscriptionId <subscription ID>
    ```

1. 使用 `Start-AzSqlInstanceDatabaseLogReplay` cmdlet 为要还原的数据库启动日志重播服务。 需要提供之前复制的资源组名称、实例名称、数据库名称、存储容器 URI 和 SAS 令牌。

```PowerShell
Import-Module Az.Sql

Start-AzSqlInstanceDatabaseLogReplay -ResourceGroupName "YourResourceGroupName" -InstanceName "YourInstanceName" -Name "YourDatabaseName" -StorageContainerUri "https://yourstorageaccount.blob.core.windows.net/yourcontainer" -StorageContainerSasToken "YourSasToken"
```

## 监视迁移进度

可以使用 `Get-AzSqlInstanceDatabaseLogReplay` cmdlet 监视日志重播服务的进度。 此 cmdlet 会返回有关服务当前状态的信息，包括已还原的最后一个日志备份文件。

1. 运行以下 PowerShell 代码。

```powershell
# Import the Az.Sql module
Import-Module Az.Sql

# Set the resource group name, instance name, and database name
$resourceGroupName = "YourResourceGroupName"
$instanceName = "YourInstanceName"
$databaseName = "YourDatabaseName"

# Get the log replay status
$logReplayStatus = Get-AzSqlInstanceDatabaseLogReplay -ResourceGroupName $resourceGroupName -InstanceName $instanceName -Name $databaseName

# Display the log replay status
$logReplayStatus | Format-List
```

## 执行迁移交接

在 Azure SQL 数据库托管实例的目标实例上还原完整数据库备份之后，可以使用该数据库执行迁移交接。

1. 如果已准备好完成联机数据库迁移，请选择“开始交接”。
1. 停止所有传入源数据库的流量。
1. 执行结尾日志备份，使备份文件在 SMB 网络共享中可用，然后等到还原这最后一个事务日志备份。
1. 此时，会看到“挂起的更改”设置为 0。
1. 依次选择“确认”、“应用” 。

    ![迁移交接屏幕](../media/3-migration-cutover-screen.png)

1. 当数据库迁移状态显示“已完成”**** 时，将应用程序连接到 Azure SQL 数据库托管实例的新目标实例。