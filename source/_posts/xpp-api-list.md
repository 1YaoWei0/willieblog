---
title: X++ api list
date: 2024-12-23 15:41:59
categories:
 - x++
comments: true
description: X++ Api List
---
![X++ api list technical flow diagram](xpp-api-list/cover.png)

<!--
AI Image Prompt:
Create a clean, minimal, professional technical diagram for Microsoft Dynamics 365 Finance & Operations (D365 F&O). Topic: X++ api list. Show key components, data flow arrows, extension points, transaction boundaries, and where X++ logic executes. Use simple boxes, labels, and directional connectors on a white background. Style should look like an enterprise architecture blueprint, no decorative art, no characters, no 3D effects.
-->

### FileIOPermission

`FileIOPermission`类用于设置文件的访问权限。`assert()`方法用于验证`FileIOPermission`类设置权限是否成功，如果错误，便会抛出错误，效果类似`try catch`程序块。

### RecordInsertList

用于批量插入的工具类，示例代码如下：

{% asset_img "xpp-api-list-recordinsertlist.png" "RecordInsertList example" %}