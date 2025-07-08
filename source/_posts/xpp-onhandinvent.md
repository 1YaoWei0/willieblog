---
title: Query On-Hand inventory using specific inventory-dimensions
comments: true
date: 2025-07-08 20:10:18
tags:
categories:
 - x++
description: Query On-Hand inventory using specific inventory-dimensions
---

## InventDimOnhand

Using the class InventDimOnhand, you can determine the on-hand inventory of articles and/or specific inventory-dimensions. The following job determines the inventory for a specific item and a specific inventlocation. This is grouped per color.

```c#
static void getInventOnhandExample(Args _args)
{
    ItemId itemId;
    InventDimOnHand inventDimOnHand;
    InventDimParm inventDimParmOnHandLevel;
    InventDimOnHandIterator inventDimOnHandIterator;
    InventDimOnHandMember inventDimOnHandMember;
    InventDim inventDim;
    InventDim inventDimCriteria;
    InventDimParm inventDimParmCriteria;

    // Item: Query specific item
    itemId = "DMO003";
    
    // inventDimCriteria: Apply ranges
    inventDimCriteria.wmsLocationId             = "12-1";
    inventDimCriteria.InventBatchId             = "DMOBatch001";

    // inventDimParmCriteria: should values from inventDimCriteria be used?
    inventDimParmCriteria.ItemIdFlag            = false;
    inventDimParmCriteria.InventSiteIdFlag      = false;
    inventDimParmCriteria.InventLocationIdFlag  = false;
    inventDimParmCriteria.wmsLocationIdFlag     = true;     // wmsLocationId from inventDimCriteria will be used
    inventDimParmCriteria.wmsPalletIdFlag       = false;
    inventDimParmCriteria.InventBatchIdFlag     = false;    // inventBatchId from inventDimCriteria will not be used
    inventDimParmCriteria.InventSerialIdFlag    = false;
    inventDimParmCriteria.ConfigIdFlag          = false;
    inventDimParmCriteria.InventSizeIdFlag      = false;
    inventDimParmCriteria.InventColorIdFlag     = false;
    inventDimParmCriteria.InventStyleIdFlag     = false;

    // inventDimParmOnHandLevel: Which dimensions should be used to group for?
    // inventDimParmOnHandLevel necessary for inventDimOnHandLevel::DimParm
    inventDimParmOnHandLevel.ItemIdFlag            = true;      // necessary
    inventDimParmOnHandLevel.InventSiteIdFlag      = false;
    inventDimParmOnHandLevel.InventLocationIdFlag  = false;
    inventDimParmOnHandLevel.wmsLocationIdFlag     = false;
    inventDimParmOnHandLevel.wmsPalletIdFlag       = false;
    inventDimParmOnHandLevel.InventBatchIdFlag     = false;
    inventDimParmOnHandLevel.InventSerialIdFlag    = false;
    inventDimParmOnHandLevel.ConfigIdFlag          = false;
    inventDimParmOnHandLevel.InventSizeIdFlag      = false;
    inventDimParmOnHandLevel.InventColorIdFlag     = true;      // group by color
    inventDimParmOnHandLevel.InventStyleIdFlag     = false;

    inventDimOnHand = InventDimOnHand::newAvailPhysical(itemId,
                                                        inventDimCriteria,
                                                        inventDimParmCriteria,
                                                        InventDimOnHandLevel::DimParm,
                                                        inventDimParmOnHandLevel);

    inventDimOnHandIterator = inventDimOnHand.onHandIterator();
    while (inventDimOnHandIterator.more())
    {
        inventDimOnHandMember = inventDimOnHandIterator.value();

        inventDim = InventDim::find(inventDimOnHandMember.parmInventDimId());

        info(con2Str([  inventDimOnHandMember.parmItemId(),
                        inventDim.InventSiteId,
                        inventDim.InventLocationId,
                        inventDim.wmsLocationId,
                        inventDim.wmsPalletId,
                        inventDim.InventBatchId,
                        inventDim.InventSerialId,
                        inventDim.ConfigId,
                        inventDim.InventSizeId,
                        inventDim.InventColorId,
                        inventDim.InventStyleId,
                        inventDimOnHandMember.parmInventQty()]));

        inventDimOnHandIterator.next();
    }
}
```

## InventOnHand

Below function returns onhand qty of item. You can further filter on hand qty of the item for a location by passing invent location id as its optional parameter.

```c#
display InventQtyOnHand qtyOnHand(InventLocationId _inventLocationId = '')
{
    InventOnHand inventQtyOnHand;

    inventQtyOnHand = InventOnHand::newItemId(this.ItemId);

    if (_inventLocationId)
        inventQtyOnHand.parmInventLocationId(_inventLocationId);

    return inventQtyOnHand.availPhysical();
}
```