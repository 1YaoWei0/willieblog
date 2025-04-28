---
title: Merge the Ledger Account
categories:
 - x++
comments: true
date: 2024-12-23 15:56:06
description: Merge the Ledger Account
---


The example code as shown below:

```c#
public static LedgerDimensionAccount generateDefaultDimension(container _conName, container _conValue)
{
    DimensionAttributeValueSetStorage   valueSetStorage = new DimensionAttributeValueSetStorage();
    int                                 i;
    DimensionAttribute                  dimensionAttribute;
    DimensionAttributeValue             dimensionAttributeValue;
    container                           conAttr  = _conName;
    container                           conValue = _conValue;
    str                                 dimValue;

    for (i = 1; i <= conLen(conAttr); i++)
    {
        dimensionAttribute = dimensionAttribute::findByName(conPeek(conAttr,i));

        if (dimensionAttribute.RecId == 0)
        {
            continue;
        }

        dimValue = conPeek(conValue,i);

        if (dimValue != "")
        {
            // The last parameter is "true". A dimensionAttributeValue record will be created if not found.
            dimensionAttributeValue = dimensionAttributeValue::findByDimensionAttributeAndValue(dimensionAttribute, dimValue, false, true);

            // Add the dimensionAttibuteValue to the default dimension
            valueSetStorage.addItem(dimensionAttributeValue);
        }
    }
    return valueSetStorage.save();
}

public static LedgerDimensionAccount replaceDefaultDimensionValues(
    DimensionDefault    _invoiceDefaultDimension,
    DimensionDefault    _purchDefaultDimension,
    container           _conName)
{
    DimensionAttributeValueSetStorage invoiceValueSetStorage    = DimensionAttributeValueSetStorage::find(_invoiceDefaultDimension);
    DimensionAttributeValueSetStorage purchValueSetStorage      = DimensionAttributeValueSetStorage::find(_purchDefaultDimension);

    for (int i = 1; i <= conLen(_conName); i++)
    {
        var dimensionAttribute      = DimensionAttribute::findByName(conPeek(_conName, i));
        var invoiceDimensionValue   = invoiceValueSetStorage.getDisplayValueByDimensionAttribute(dimensionAttribute.RecId);
        var purchDimensionValue     = purchValueSetStorage.getDisplayValueByDimensionAttribute(dimensionAttribute.RecId);

        if (purchDimensionValue && (purchDimensionValue != invoiceDimensionValue))
        {
            var newDimensionAttributeValue = DimensionAttributeValue::findByDimensionAttributeAndValue(dimensionAttribute, purchDimensionValue, false, true);

            invoiceValueSetStorage.removeDimensionAttribute(dimensionAttribute.RecId);
            invoiceValueSetStorage.addItem(newDimensionAttributeValue);
        }
    }    
    return invoiceValueSetStorage.save();
}
```