---
title: How To Develop A D365 Number Sequence
comments: true
date: 2025-07-11 20:31:12
tags:
 - NumberSeq
categories:
 - x++
description: How To Develop A D365 Number Sequence
---

In Dynamics 365 Finance and Operations, number sequences are essential for generating unique identifiers for business records, such as orders, transactions, or custom entities. In this blog, we’ll walk through how to develop a custom number sequence module step by step.

## Step 1: Create a Custom Number Sequence Class

The first step is to create a class that extends `NumberSeqApplicationModule`. This class defines the number sequence logic and configuration.

Here’s an example:

```c#
/// <summary>
/// HM_SUN-034204_EXT-00189 SPIFF workspace
/// Willie Yao - 07/07/2025
/// HMSpiffsIdNumSeq
/// </summary>
internal final class HMSpiffsIdNumSeq extends NumberSeqApplicationModule
{
    /// <summary>
    /// Subscribe to global number sequence module registration
    /// </summary>
    /// <param name="numberSeqModuleNamesMap">numberSeqModuleNamesMap</param>
    [SubscribesTo(classstr(NumberSeqGlobal), delegatestr(NumberSeqGlobal, buildModulesMapDelegate))]
    static void buildModulesMapSubsciber(Map numberSeqModuleNamesMap)
    {
        NumberSeqGlobal::addModuleToMap(classnum(HMSpiffsIdNumSeq), numberSeqModuleNamesMap);
    }

    /// <summary>
    /// Return the number sequence module
    /// </summary>
    /// <returns>NumberSeqModule</returns>
    public NumberSeqModule numberSeqModule()
    {
        return NumberSeqModule::CRM;
    }

    /// <summary>
    /// Load number sequence datatype configuration
    /// </summary>
    protected void loadModule()
    {
        NumberSeqDatatype datatype = NumberSeqDatatype::construct();

        datatype.parmDatatypeId(extendedTypeNum(HMSpiffsId));
        datatype.parmReferenceHelp(literalStr("@HM:HMTheSPIFFIdentificationReference"));
        datatype.parmWizardIsContinuous(false);
        datatype.parmWizardIsManual(NoYes::No);
        datatype.parmWizardFetchAheadQty(10);
        datatype.parmWizardIsChangeDownAllowed(NoYes::No);
        datatype.parmWizardIsChangeUpAllowed(NoYes::No);
        datatype.parmSortField(1);
        datatype.addParameterType(NumberSeqParameterType::DataArea, true, false);

        this.create(datatype);
    }
}
```

**Key Points:**

- The class must extend `NumberSeqApplicationModule`.

- Use the `buildModulesMapSubsciber` method to register your module.

- Implement `loadModule()` to define the configuration.

- Choose an appropriate `NumberSeqModule` (e.g., `Sales`, `Inventory`).

## Step 2: Load the Number Sequence from a Parameter Form

Once your number sequence class is defined, you can call it in a form (typically in a `loadModule()` method for parameter setup):

```c#
HMSpiffsIdNumSeq spiffsIdNumSeq = new HMSpiffsIdNumSeq();
spiffsIdNumSeq.load();
```

This code is often placed in the `loadModule()` method of a parameter setup form class (e.g., `MyModuleParameters`). This ensures that when number sequences are initialized, your custom identifier logic is registered.

With these two simple steps, you’ve successfully extended D365FO with a custom number sequence. This is especially useful for custom business scenarios where the out-of-box sequences don't meet your needs.

If you need to integrate this with forms, workflows, or auto-numbering during record creation, you can take it a step further by using the `NumberSeqReference` framework.