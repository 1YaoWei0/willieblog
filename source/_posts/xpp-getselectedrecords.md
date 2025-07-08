---
title: Get selected records in a grid on a form - X++
comments: true
date: 2025-07-08 20:23:17
tags:
categories:
 - x++
description: Get selected records in a grid on a form - X++
---

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