---
title: How to add multiple lookup control to Form Grid
comments: true
date: 2025-07-11 20:22:49
tags:
 - Lookup
 - Multiple Selection Lookup
 - Form
 - Grid
categories:
 - x++
description: How to add multiple lookup control to Form Grid
---

In some scenarios, we may need to allow users to select multiple values from a lookup within a grid on a form—for example, assigning multiple categories or tags to a line-level record. Out of the box, Dynamics 365 Finance and Operations does not support multi-select lookups directly on grid controls. However, we can achieve this behavior by extending the `SysLookupMultiSelectGrid` framework.

## Step 1: Create a Custom Class Based on `SysLookupMultiSelectGrid`

To begin, we need to create a custom class that extends SysLookupMultiSelectGrid. This class will encapsulate the logic for initializing the lookup, storing selected values, and writing them back to the grid's data source.

Here are the key methods you should implement:

- init(): Register the lookup override and capture context such as the caller data source and field.

- lookup(FormStringControl _callerControl): Open the lookup form and load existing values into the selection. After the user makes a selection, update the data source field.

- getSelected(): Return the container of selected record IDs.

- setSelected(): Persist the selected values into a specific field (usually a semicolon-delimited string field).

- construct(...): A static method to initialize the custom multi-select lookup instance.

By implementing these methods, your custom control can support seamless multi-selection directly from the form grid.

```c#
/// <summary>
/// Willie Yao - 07/07/2025
/// HMLookupMultiSelectGridOrderCategory
/// </summary>
class HMLookupMultiSelectGridOrderCategory extends SysLookupMultiSelectGrid
{
    FormDataSource              callerDataSource;
    FieldId                     callerFieldId;
    #define.Spliter(';')

    /// <summary>
    /// Willie Yao - 07/07/2025
    /// getSelected
    /// </summary>
    /// <returns>container</returns>
    public container getSelected()
    {
        Array       markedRecords;
        Common      common;
        int         i;
        int         recordsMarked = 0;

        formDS      = formRun.dataSource();
        selectedId  = conNull();
        selectedStr = conNull();

        for (common = getFirstSelection(formDS); common; common = formDS.getNext())
        {
            recordsMarked++;
        }

        if (recordsMarked > 0)
        {
            markedRecords = formDS.recordsMarked();

            for (i = 1; i <= markedRecords.lastIndex(); i++)
            {
                formDS      = formRun.dataSource();
                formDS.setPosition(markedRecords.value(i));
                common      = formDS.cursor();
                selectedId  += common.RecId;
                selectedStr += common.(defaultSelectFieldId);
            }
        }

        return selectedId;
    }

    /// <summary>
    /// Willie Yao - 07/07/2025
    /// init
    /// </summary>
    public void init()
    {
        callingControlId.registerOverrideMethod(methodStr(FormStringControl, lookup), methodStr(HMLookupMultiSelectGridOrderCategory, lookup), this);
        callerDataSource    = callingControlId.formRun().dataSource(callingControlId.fieldBinding().tableName());
        callerFieldId       = callingControlId.fieldBinding().fieldId();
    }

    /// <summary>
    /// Willie Yao - 07/07/2025
    /// lookup
    /// </summary>
    /// <param name="_callerControl">FormStringControl</param>
    public void lookup(FormStringControl _callerControl)
    {
        FormDataSource  formDataSource;
        Common          callerRecord;
 
        if (callerDataSource)
        {
            selectedId      = conNull();
            selectedStr     = conNull();
            callerRecord    = callerDataSource.cursor();
 
            if (callerRecord.RecId)
            {
                HMSpiffs spiffs = callerRecord;
                container con = str2con(spiffs.OrderCategory, ';', false);
                int i;
                for (i = 1; i <= conLen(con); i++)
                {
                    sunTAFSalesCategory salesCategory = sunTAFSalesCategory::find(conPeek(con, i));

                    selectedId += salesCategory.RecId;
                    selectedStr += salesCategory.SalesCategoryID;
                }
            }
            else
            {
                if (callerRecord.validateWrite())
                {
                    callerRecord.write();
                }
                else
                {
                    return;
                }
            }
        }
 
        this.run();
 
        if (callerDataSource)
        {
            callerRecord.(callerFieldId)    = con2Str(selectedStr, #Spliter);
            formDataSource                  = callerRecord.dataSource();
            formDataSource.refresh();
        }
 
        _callerControl.modified();
    }

    /// <summary>
    /// Willie Yao - 07/07/2025
    /// setSelected
    /// </summary>
    public void setSelected()
    {
        Common          callerRecord;
 
        if (callerDataSource)
        {
            callerRecord    = callerDataSource.cursor();
 
            if (callerRecord)
            {
                HMSpiffs spiffs = callerRecord;
                spiffs.OrderCategory = con2Str(selectedStr, ';');
            }
        }
    }

    /// <summary>
    /// Willie Yao - 07/07/2025
    /// construct
    /// </summary>
    /// <param name="_ctrlId">ctrlId</param>
    /// <param name="_query">query</param>
    /// <param name="_selectField">selectField</param>
    /// <returns>HMLookupMultiSelectGridOrderCategory</returns>
    public static HMLookupMultiSelectGridOrderCategory construct(
        FormControl _ctrlId,
        Query       _query,
        container   _selectField = conNull())
    {
        HMLookupMultiSelectGridOrderCategory lookupMultiSelectControl;
 
        lookupMultiSelectControl = new HMLookupMultiSelectGridOrderCategory();
        lookupMultiSelectControl.parmCallingControl(_ctrlId);
        lookupMultiSelectControl.parmCallingControlId(_ctrlId);
        lookupMultiSelectControl.parmQuery(_query);
        lookupMultiSelectControl.parmSelectField(_selectField);
        lookupMultiSelectControl.init();
 
        return lookupMultiSelectControl;
    }

}

```

## Step 2: Use the Lookup Control in the Form Grid

Once the custom lookup class is ready, the next step is to integrate it into the form grid where multi-selection is required.

In the `FormControl` method override (usually on the lookup() method of the relevant control), you can instantiate your custom lookup using the construct() method.

For example:

```c#
HMLookupMultiSelectGridOrderCategory::construct(MainGrid_OrderCategory, query, con);
```

- `MainGrid_OrderCategory` is the name of the string control (usually bound to a field like OrderCategory) on the form grid.

- `quer`y is a standard `Query` object used to define the records shown in the lookup.

- `con` is a container specifying which field(s) to select and return from the lookup records.

This setup allows the form to display a multi-select lookup when users interact with the field in the grid, enabling them to choose multiple entries and store them in a semicolon-delimited format (or another structure as needed).