---
title: How to Copy the Add Items 
comments: true
date: 2025-07-11 20:03:12
tags:
categories:
description: How to Copy the Add Items 
---

## Summary

The D365 F&O has an out of box function named "**Add products**", If developer develops an item-base table and wants to copy this function for this table. The following codes.

## Code Example

To begin with, we need to extend the standard system class to implement a custom strategy for loading default items in the 'Add Productions' dialog. When the user selects one or multiple lines and clicks the 'Add Productions' button, the dialog should automatically prefill the item field(s) based on the selected records.

Secondly, we need to use a Chain of Command (CoC) extension on the createProducts method to implement the logic for adding items to the custom table.

```c#
/// <summary>
/// Willie Yao - 07/07/2025
/// </summary>
[ExtensionOf(classStr(RetailCreateLinesFromProductsToAdd))]
final class HMClass_RetailCreateLinesFromProductsToAdd_Extension
{
    /// <summary>
    /// Willie Yao - 07/07/2025
    /// loadLinesFromCaller
    /// </summary>
    /// <param name="_callerArgs"><Args/param>
    /// <param name="_tmpProductsToAdd">TmpRetailProductsToAdd</param>
    public void loadLinesFromCaller(Args _callerArgs, TmpRetailProductsToAdd _tmpProductsToAdd)
    {
        next loadLinesFromCaller(_callerArgs, _tmpProductsToAdd);

        TableId callerTableId = _callerArgs.dataset();

        switch (callerTableId)
        {
            case tableNum(HMTable):
                this.loadHMTable();
                break;
        }
    }

    /// <summary>
    /// Willie Yao - 07/07/2025
    /// loadHMTable
    /// </summary>
    internal void loadHMTable()
    {
        InventTable     inventTable;
        EcoResProduct   ecoResProduct;
        HMTable         MTable = callerArgs.record();

        select firstonly product from inventTable
            where inventTable.ItemId == MTable.ItemId
            join ecoResProduct
                where ecoResProduct.RecId == inventTable.Product;

        this.loadLines(ecoResProduct, 1, InventDim::findOrCreateBlank().inventDimId);
    }

    /// <summary>
    /// Willie Yao - 07/07/2025
    /// createProducts
    /// </summary>
    /// <param name="_tmpProductsToAdd">TmpRetailProductsToAdd</param>
        public void createProducts(TmpRetailProductsToAdd _tmpProductsToAdd)
    {
        next createProducts(_tmpProductsToAdd);

        switch (common.TableId)
        {
            case tableNum(HMTable):
                this.createHMTable(HMTable::findByRecId(common.RecId), _tmpProductsToAdd);
                break;
        }
    }

    /// <summary>
    /// Willie Yao - 07/07/2025
    /// createHMTable
    /// </summary>
    /// <param name="_HMTable">HMTable</param>
    /// <param name="_tmpProductsToAdd">TmpRetailProductsToAdd</param>
    internal void createHMTable(HMTable _HMTable, TmpRetailProductsToAdd _tmpProductsToAdd)
    {
        HMTable HMTable;

        ttsbegin;
        while select _tmpProductsToAdd
        {
            HMTable.clear();
            HMTable.ItemId = _tmpProductsToAdd.ItemId;
            // Others customized fields value populated logic.

            if (HMTable.validateWrite())
            {
                HMTable.insert();
            }            
        }
        ttscommit;
    }

}
```

Finally, we need to override the createCallerTableRecords method in the RetailAddItems form to trigger the logic for creating item records.

```c#
/// <summary>
/// Willie Yao - 07/07/2025
/// </summary>
[ExtensionOf(formStr(RetailAddItems))]
final class HMForm_RetailAddItems_Extension
{
    /// <summary>
    /// Willie Yao - 07/07/2025
    /// createCallerTableRecords
    /// </summary>
    /// <param name="_callerTableId">TableId</param>
    /// <param name="_tmpInventTable">TmpRetailProductsToAdd</param>
    public void createCallerTableRecords(TableId _callerTableId, TmpRetailProductsToAdd _tmpInventTable)
    {
        next createCallerTableRecords(_callerTableId, _tmpInventTable);

        switch (_callerTableId)
        {
            case tableNum(HMTable):
                this.createOrderLine(FormDataUtil::getFormDataSource(this.args().record()));
                break;
        }
    }

}
```