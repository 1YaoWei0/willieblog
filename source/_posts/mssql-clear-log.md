---
title: The Way for Clearing the Log Disk
date: 2024-12-23 16:33:47
tags:
 - x++
 - t-sql
categories:
 - d365
comments: true
description: This article shows you the sql scripts that cleared the sql log disk.​
---

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