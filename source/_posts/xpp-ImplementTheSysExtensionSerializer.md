---
title: Implement the SysExtensionSerializer
comments: true
date: 2025-07-14 19:40:52
tags:
 - Table
 - Extend
categories:
 - x++
description: This content is from the MB-500 training material. How should we handle when the extended fields exceed 10?
---

You can add fields to a table by using an extension. If you're adding more than 10 fields to a table extension, the compiler throws the best practice error. We recommend that you use the alternative way of creating a new table and adding a foreign key relation to the standard table. Maps are available to help you improve a developer’s processes. The SysExtensionSerializerMap and SysExtensionSerializerExtensionMap maps make the create, read, update, and delete (CRUD) operations automated to the custom table.
You can create a new table for your new fields, as the following image depicts.

{% asset_img "Screenshot of the My Cust Table showing the Sys Extension Serializer Map_.png" "Screenshot of the My Cust Table showing the Sys Extension Serializer Map" %}

Consider the following details about implementing the SysExtensionSerializer extension：

- The CustTable field is an int64 (RecId).
- It creates relation to the CustTable standard table and points to RecId on CustTable, as the following screenshot shows.

{% asset_img "Screenshot of the Properties form showing that the Replacement Key is set to the Alternate key_.png" "Screenshot of the Properties form showing that the Replacement Key is set to the Alternate key" %}

- It creates mapping to map the SysExtensionSerializerExtensionMap field BaseRecId to a new table called MyCustTable and the CustTable field。
- It creates an alternate key (Index) on the table for the CustTable field in the MyCustTable table. For the table property, the Replacement Key is set to the Alternate key, as the following screenshot shows.

{% asset_img "Screenshot of the Properties form showing the relation to the standard table_.png" "Screenshot of the Properties form showing the relation to the standard table" %}

The SysExtensionSerializerMap contains methods for automatically inserting, updating, and performing many other functions. The standard CustTable table includes calls to the SysExtensionSerializer framework, which means that whenever a record in CustTable is updated, your table is also updated, as the following screenshot shows.

{% asset_img "Screenshot showing the Cust Table_.png" "Screenshot showing the Cust Table" %}

Using the SysExtensionSerializer framework fields helps you add a related table by following best practices and enabling CRUD functionality.