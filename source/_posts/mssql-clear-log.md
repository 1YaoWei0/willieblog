---
title: The Way for Clearing the Log Disk
date: 2024-12-23 16:33:47
categories:
 - t-sql
comments: true
description: This article shows you the sql scripts that cleared the sql log disk.​
---
![The Way for Clearing the Log Disk technical flow diagram](mssql-clear-log/cover.png)

<!--
AI Image Prompt:
Create a clean, minimal, professional technical diagram for Microsoft Dynamics 365 Finance & Operations (D365 F&O). Topic: The Way for Clearing the Log Disk. Show key components, data flow arrows, extension points, transaction boundaries, and where X++ logic executes. Use simple boxes, labels, and directional connectors on a white background. Style should look like an enterprise architecture blueprint, no decorative art, no characters, no 3D effects.
-->

{% asset_img "mssql-clear-log.png" "clear mssql clear log example" %}

The example SQL script code as shown below:

```sql
use [AxDB]

SELECT name FROM sys.database_files

USE [master]
GO
ALTER DATABASE [AxDB] SET RECOVERY SIMPLE WITH NO_WAIT
GO

USE [AxDB]
GO
DBCC SHRINKFILE (AxDB_Restore_log, 1024)
GO

USE [master]
GO
ALTER DATABASE [AxDB] SET RECOVERY FULL WITH NO_WAIT
GO
```