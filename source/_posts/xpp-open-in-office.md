---
title: Open in Microsoft Office
comments: true
date: 2025-01-18 22:15:28
tags: 
categories:
 - d365
 - x++
description: Open in Microsoft Office
---

## Operation Step

The first step is to implement the **OfficeIMenuCustomizer** and **OfficeIGenerateWorkbookCustomExporter**.

{% asset_img "xpp-open-in-office.png" "Class Header" %}

The example code is as follows:

{% asset_img "xpp-open-in-office2.png" "Class Header" %}

{% asset_img "xpp-open-in-office3.png" "Class Header" %}

## Filter Setting

Do not use the Recid of the primary table as the filter condition. This will inevitably require the Recid of the primary table to be released, but this will cause an insertion problem for the data entity: since RecId is the primary key of the primary table, it will make an insertion error regardless of any field entered, and will not be able to skip validation.

In this case, you can only use other unique keys instead, but many unique indexes contain multiple fields, so you need to create a computed column in the data entity and then construct a computed column field from the multiple fields of the unique index.