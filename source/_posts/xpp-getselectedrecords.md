---
title: Get selected records in a grid on a form - X++
comments: true
date: 2025-07-08 20:23:17
tags:
categories:
 - x++
description: Get selected records in a grid on a form - X++
---
![Get selected records in a grid on a form - X++ technical flow diagram](xpp-getselectedrecords/cover.png)

<!--
AI Image Prompt:
Create a clean, minimal, professional technical diagram for Microsoft Dynamics 365 Finance & Operations (D365 F&O). Topic: Get selected records in a grid on a form - X++. Show key components, data flow arrows, extension points, transaction boundaries, and where X++ logic executes. Use simple boxes, labels, and directional connectors on a white background. Style should look like an enterprise architecture blueprint, no decorative art, no characters, no 3D effects.
-->

If you can get the form data source element, then you can get the selected records. Developers usually can get the args in the source code. That is a sally port for extracting the selected records.

```c#

public static void main(Args _args)
{
    FormRun formRun = _args.caller();
    FormDataSource formDS = formRun.dataSource();
    SalesTable salesTable; // Use the SalesTable for testing.

    while (salesTable = formDS.mark(1) ? formDS.cursor() : formDS.getFirst(1);
            salesTable:
            salesTable.getNext())
    {}
}

public void modified()
{
    SalesTable salesTable; // Use the SalesTable for testing.

    while (salesTable = SalesTable_ds.mark(1) ? SalesTable_ds.cursor() : SalesTable_ds.getFirst(1);
            salesTable:
            salesTable.getNext())
    {}
}

···