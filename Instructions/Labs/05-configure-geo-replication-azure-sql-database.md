---
lab:
  title: 为 Azure SQL 数据库配置异地复制
---

# 为 Azure SQL 数据库配置异地复制

在本练习中，您将学习如何为 Azure SQL 数据库启用地理复制并执行故障转移到辅助区域。 这包括创建数据库副本、为辅助数据库设置新服务器以及启动强制故障转移。 您还将学习如何检查部署状态，并了解地理辅助或地理副本在 Azure SQL 数据库管理中的作用。 最后，您将使用 Azure 门户手动将数据库故障转移到另一个区域。 本练习提供了管理和确保 Azure SQL 数据库弹性的关键方面的实践经验。

此练习大约需要 **30** 分钟。

> **注意：** 要完成此练习，您需要访问 Azure 订阅以创建 Azure 资源。 如果没有 Azure 订阅，请在开始之前创建一个[免费帐户](https://azure.microsoft.com/free/?azure-portal=true)。

## 开始之前

为完成本练习，我们将使用许多资源和工具。 让我们逐一进行详细了解：

|  | 说明 |
| --- | --- |
| **主服务器** | 我们将在本实验室中设置的 Azure SQL 数据库服务器。|
| **主数据库** | 在辅助服务器上创建的 **AdventureWorksLT** 示例数据库。|
| **辅助服务器** | 我们将在本实验室中额外设置的 Azure SQL 数据库服务器。 |
| **辅助数据库** | 这是我们在辅助服务器上的数据库副本。 |
| **SQL Server Management Studio** | 下载最新的 [SQL Server Management Studio](https://learn.microsoft.com/sql/ssms/download-sql-server-management-studio-ssms) 并将其安装到你选择的计算机中。 |

## 配置 Azure SQL 数据库资源

让我们分两步创建 Azure SQL 数据库资源。 首先，我们将建立主服务器和数据库。 然后，重复上述过程，用不同的名称建立辅助服务器。 这样就有了两个 Azure SQL 服务器，每个服务器都有自己的防火墙规则。 不过，只有主服务器有数据库。

1. 导航到 [Azure 门户](https://portal.azure.com)，使用 Azure 帐户凭据登录。

1. 选择右上角菜单栏中的 **Cloud Shell** 选项（看起来像 shell 提示符**`>_`**）。

1. 一个窗格从底部滑出，要求你选择首选的 shell 类型。 选择 **Bash**。

1. 如果这是你第一次打开 **Cloud Shell**，系统会提示你创建一个存储账户（用于跨会话持久保存数据）。 按照提示创建一个。

1. 启动外壳后，你将在 Azure 门户内看到一个命令行界面，可以在此输入脚本命令。

1. 选择**{}** 打开编辑器，然后复制并粘贴下面的脚本。 
 
    > **注意**：运行脚本前，请记住用实际值替换脚本中的占位符值。 如果需要编辑脚本，请在 **Cloud Shell** 中输入 `code` 以使用内置文本编辑器。
        
    ```powershell
    subscription="<Your subscription>"
    resourceGroup="<Your resource group>"
    location="<Your region, same as your resource group>"
    serverName="<Your SQL server name>"
    adminLogin="sqladmin"
    password="<password>"
    databaseName="AdventureWorksLT"
    
    az account set --subscription $subscription
    az sql server create --name $serverName --resource-group $resourceGroup --location $location --admin-user $adminLogin --admin-password $password
    az sql db create --resource-group $resourceGroup --server $serverName --name $databaseName --sample-name AdventureWorksLT --service-objective Basic

    ```
    此 Azure CLI 脚本会设置活动的 Azure 订阅，创建新的 Azure SQL 服务器，然后创建新的 Azure SQL 数据库，并填充 AdventureWorksLT 示例数据。

1. 右键单击编辑器页面，然后选择**保存**。

1. 提供文件名。 文件扩展名应为 **.ps1**。

1. 在 Cloud Shell 终端上键入并执行命令。

    ```bash
    chmod +x <script_name>.ps1

    ```
    
    替换 *<script_name>* 以反映您为脚本提供的名称。 该命令将更改您创建的文件的权限，使其可执行。

1. 执行该脚本。 
    
    ```powershell
    ./<script_name>.ps1

    ```

1. 该过程完成后，进入 Azure 门户并导航到 SQL 服务器页面，从而导航到新创建的 Azure SQL 服务器。 

1. 在 Azure SQL 服务器主页面，选择左侧的**网络**。

1. 在**公共访问**选项卡中，选择**选定的网络**。

1. 在**防火墙规则**部分中，选择 **+添加客户端 IPv4 地址**。 键入 IP 地址，然后选择**保存**。

    ![Azure SQL 数据库防火墙规则页面截图。](../media/5-new-firewall-rule.png)

    此时，你应该可以通过 SQL Management Studio 等客户端工具连接到主`AdventureWorksLT`数据库。

1. 现在，让我们创建一个辅助 Azure SQL 服务器。 重复前面的步骤 (6-14)，但确保使用不同的`serverName`和`location`服务器。 此外，通过注释跳过创建数据库的代码`az sql db create`命令。 这样就会在不同区域创建一个新服务器，但不包含示例数据库。

## 启用异地复制

现在，让我们为 Azure SQL 资源创建辅助副本。

1. 在 Azure 门户中，通过搜索**SQL 数据库**导航到你的数据库。

1. 选择 SQL 数据库 **AdventureWorksLT**。

1. 在 Azure SQL 数据库主页面，选择左侧**数据管理**下的**副本**。

1. 选择 **+ 创建副本**。

1. 在**创建 SQL 数据库 - 异地副本**页面的**服务器**下，选择之前创建的新辅助 SQL 服务器。

1. 依次选择**查看 + 创建**、**创建** 现在将创建辅助数据库并播种。 要检查状态，请查看 Azure 门户顶部的通知图标。 

1. 如果成功，则将从**正在进行部署**变为**部署已成功**。

1. 使用 SQL Management Studio 连接到辅助 Azure SQL 服务器。

## 将 SQL 数据库故障转移到辅助区域

想象一下主 Azure SQL 数据库由于区域中断而出现问题的场景。 为确保服务的连续性并尽量减少停机时间，您需要执行强制故障切换。

强制故障转移会切换主数据库和辅助数据库的角色。 辅助数据库接管主数据库，而原来的主数据库则成为辅助数据库。 这样，在解决原始主数据库的问题时，您的应用程序可以继续使用辅助副本运行。

让我们学习如何启动强制故障转移以应对区域中断。

1. 导航到 SQL 服务器页面，然后选择辅助服务器。

1. 在左侧的**设置**部分，选择 **SQL 数据库**。

1. 在 Azure SQL 数据库主页面，选择左侧**数据管理**下的**副本**。 地理复制链接现已建立。

1. 选择辅助服务器的 **...** 菜单，然后选择**强制故障转移**

    > **注意**：强制故障切换会将辅助数据库切换为主数据库。 在此操作期间，所有会话都会断开连接。

1. 出现警告消息提示时，选择**是**。

1. 主副本的状态将切换为**待定**，次副本的状态将切换为**故障转移**。 

    > **注意**：此操作可能需要几分钟时间。 一旦完成，角色将发生逆转：辅助服务器将成为新的主服务器，而原来的主服务器将变成辅助服务器。

考虑一下为什么要将主服务器和辅助 SQL 服务器放置在同一区域，以及何时选择不同的区域会有好处。

现在，你了解了如何为 Azure SQL 数据库启用异地副本，以及如何使用 Azure 门户手动地将其故障转移到另一个区域。

## 清理

在自己的订阅中操作时，最好在项目结束时确定是否仍需要已创建的资源。 

让资源不必要地运行可能会导致额外费用。 可以在 [Azure 门户](https://portal.azure.com?azure-portal=true)中单独删除资源或删除整套资源。

## 详细信息

有关 Azure SQL 数据库地理复制的更多信息，请参阅[活动异地复制](https://review.learn.microsoft.com/azure/azure-sql/database/active-geo-replication-overview)。