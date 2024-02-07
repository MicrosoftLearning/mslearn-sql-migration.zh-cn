---
lab:
  title: 确定 SQL 迁移的兼容性问题
---

# 确定 SQL 迁移的兼容性问题

在我们的应用场景中，你需要评估要迁移到 Azure SQL 数据库的旧版 SQL Server 数据库的就绪情况。 你的任务是对其旧数据库进行评估，并识别任何潜在的兼容性问题或需要在迁移前执行的更改。 还需要查看数据库的架构，并确定 Azure SQL 数据库不支持的任何功能或配置。

该练习大约需要 **15** 分钟。

> **注意**：若要完成此练习，需要访问 Azure 订阅以创建 Azure 资源。 如果没有 Azure 订阅，请在开始之前创建一个[免费](https://azure.microsoft.com/free/?azure-portal=true)帐户。

## 开始之前

若要运行此练习，请确保在运行之前具备以下条件：

- 您需要 SQL Server 2019 或更高版本，以及与特定 SQL Server 实例兼容的 [**AdventureWorksLT**](https://learn.microsoft.com/sql/samples/adventureworks-install-configure#download-backup-files) 轻量级数据库。
- 下载并安装[ Azure Data Studio](https://learn.microsoft.com/sql/azure-data-studio/download-azure-data-studio)。 如果已经安装，请进行更新，确保使用的是最新版本。
- 对源数据库具有读取访问权限的 SQL 用户。

## 还原 SQL Server 数据库并运行命令

1. 选择 Windows 开始按钮，然后键入 SSMS。 从列表中选择**Microsoft SQL Server Management Studio 18**。  

1. 当 SSMS 打开时，请注意，**连接到服务器**对话框将使用默认实例名称预填充。 选择**连接** 

1. 选择**数据库**文件夹，然后选择**新建查询**

1. 在新建查询窗口中，将以下 T-SQL 复制并粘贴到其中。 确保数据库备份文件名和路径与实际备份文件匹配。 如果不匹配，命令将失败。 执行查询以还原数据库。

    ```sql
    RESTORE DATABASE AdventureWorksLT
    FROM DISK = 'C:\<FolderName>\AdventureWorksLT2019.bak'
    WITH RECOVERY,
          MOVE 'AdventureWorksLT2019_Data' 
            TO 'C:\<FolderName>\AdventureWorksLT2019.mdf',
          MOVE 'AdventureWorksLT2019_Log'
            TO 'C:\<FolderName>\AdventureWorksLT2019.ldf';
    ```

    > **注意**：在运行 T-SQL 命令之前，请确保 SQL Server 计算机上具有轻型 [AdventureWorks](https://learn.microsoft.com/sql/samples/adventureworks-install-configure#download-backup-files) 备份文件。

1. 还原完成后，应会看到一条成功消息。

1. 在 SQL Server 实例中的 **AdventureWorksLT** 数据库上运行以下命令。

```sql
ALTER TABLE [SalesLT].[Customer] ADD [Next] VARCHAR(5);
```

## 安装并启动 Azure Data Studio 的 Azure 迁移扩展

按照以下步骤安装迁移扩展。 如果已经安装了 Azure 迁移扩展，则可以跳过这些步骤。

1. 在 Azure Data Studio 中打开扩展管理器。 s

1. 搜索 ***Azure SQL 迁移***，然后安装扩展。 安装后，Azure SQL 迁移扩展就会出现在已安装扩展的列表中。

1. 选择**连接**图标，然后选择**新建连接**。 

1. 在新**连接**选项卡中，输入服务器名称。 在**加密**选项中选**择可选（假）**。

1. 选择**连接** 

1. 要启动 Azure 迁移扩展，只需右键单击源实例的名称，然后选择**管理**。 

1. 在服务器菜单中的**常规**下，选择 **Azure SQL 迁移**。 这将带你进入 Azure SQL 迁移扩展的主页。

    > **注意**：如果无法在服务器菜单中看到 **Azure SQL 迁移**选项，或者 Azure SQL 迁移页面无法加载，请重新打开 Azure Data Studio。

## 运行兼容性评估

兼容性评估有助于识别潜在的迁移问题，并在迁移过程开始前提供详细的解决指导。 这可以节省大量时间和资源。 

运行 Azure Data Studio 的 Azure 迁移扩展，运行兼容性评估，然后查看 Azure SQL 数据库目标的结果。

1. 在Azure SQL 迁移仪表板中，选择**迁移到 Azure SQL**打开迁移向导。

1. 在**步骤 1：用于评估的数据库**中，选择*AdventureWorks*数据库，然后选择**下一步**。

1. 在**步骤 2：评估结果和建议**中，等待评估完成。

## 查看评估结果

现在，您可以查看迁移扩展生成的建议。

1. 在**步骤 2：评估结果和建议中**，选择** Azure SQL 数据库**作为目标平台。

1. 在页面底部，选择**查看/选择**可查看评估结果。 

1. 选择*AdventureWorks*数据库。 花点时间查看右侧的评估结果。
    
    > **注意：** 我们可以看到，之前添加的`Next`列已被标记，因为它可能会导致 Azure SQL 数据库出错。

1. 选择**取消**，然后改为选择**Azure SQL 托管实例**作为**Azure SQL**目标平台。
    
    > **注意：** Azure SQL 托管实例不再标记该`Next`列，这是为什么呢？ 
    >
    >这意味着该`Next`列可以在 Azure SQL 托管实例上安全使用。

1. 选择**保存评估报告**，以 JSON 格式保存报告。

1. 花点时间查看 JSON 文件及其属性。

## 解决问题

1. 在*AdventureWorks*数据库上运行以下 T-SQL 命令。

    ```sql
    ALTER TABLE [SalesLT].[Customer] DROP COLUMN [Next];
    ```

1. 返回向导中的**步骤 2：评估结果和建议**页面，然后选择**刷新评估**。

1. 选择 **Azure SQL Database** 作为 **Azure SQL** 目标平台。

1. 选择**查看/选择**以查看评估结果。

    > **注意：** 问题不再标记。

你已了解如何评估要迁移到 Azure SQL 数据库的 SQL Server 数据库的就绪情况。 通过解决兼容性问题并进行必要的架构更改或报告它们，你在缓解 Azure SQL 数据库将来可能出现的潜在技术问题方面迈出了重要的一步。
