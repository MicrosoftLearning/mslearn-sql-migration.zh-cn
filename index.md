---
title: Azure OpenAI 练习
permalink: index.html
layout: home
---

# SQL Server 迁移练习

以下练习旨在支持 Microsoft Learn 上 [将 SQL Server 工作负载迁移到 Azure SQL](https://learn.microsoft.com/training/paths/migrate-sql-workloads-azure/)学习路径中的模块。

{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions/Labs'" %} {% for activity in labs  %}
- [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) {% endfor %}