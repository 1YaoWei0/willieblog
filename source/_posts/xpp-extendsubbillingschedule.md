---
title: How to bring the sub billing schedule customized field value to sales order
comments: true
date: 2025-07-11 19:54:42
tags:
 - SalesTable
 - SubBillingSchedule
 - SubBillingScheduleLine
 - SalesLine
 - Extend
categories:
 - x++
description: How to bring the sub billing schedule customized field value to sales order
---

Developer can extend `SubBillCreateSalesOrder` class to extend the customized fields value populated logic. Please see the example code:

```c#
/// <summary>
/// Willie Yao - 07/04/2025
/// The extension class for SubBillCreateSalesOrder
/// </summary>
[ExtensionOf(classStr(SubBillCreateSalesOrder))]
final class HMClass_SubBillCreateSalesOrder_Extension
{
    /// <summary>
    /// Willie Yao - 07/04/2025
    /// Coc the <c>createSalesLine</c> method for initialing the sales line's customized fields.
    /// </summary>
    /// <param name="_salesLine">SalesLine</param>
    /// <param name="_salesLineConsolidated">SubBillSalesLineConsolidated</param>
    /// <param name="_salesIdCon">_salesIdCon<SalesId/param>
    /// <param name="_allocEnabled">AllocEnabled</param>
    /// <param name="_splitByItemGroup">SplitByItemGroup</param>
    /// <param name="_isInvoiceCreator">IsInvoiceCreator</param>
    /// <param name="_includeBillingDatesToItem">IncludeBillingDatesToItem</param>
    /// <returns>SalesLine</returns>
    public static SalesLine createSalesLine(
        SalesLine _salesLine,
        SubBillSalesLineConsolidated _salesLineConsolidated,
        SalesId _salesIdCon,
        boolean _allocEnabled,
        boolean _splitByItemGroup,
        boolean _isInvoiceCreator,
        boolean _includeBillingDatesToItem)
    {
        SubBillScheduleLine     scheduleLine    = SubBillScheduleLine::find(_salesLineConsolidated.SubBillBillingScheduleNumber, _salesLineConsolidated.LineNum);
        SubBillScheduleTable    scheduleTable   = SubBillScheduleTable::find(_salesLineConsolidated.SubBillBillingScheduleNumber);
        SalesLine               salesLine       = next createSalesLine(_salesLine, _salesLineConsolidated, _salesIdCon, _allocEnabled, _splitByItemGroup, _isInvoiceCreator, _includeBillingDatesToItem);
        
        if (scheduleLine)
        {
            salesLine.HMField1 = scheduleLine.HMField1;
            salesLine.HMField2 = scheduleLine.HMField2;
        }

        return salesLine;
    }

    /// <summary>
    /// Willie Yao - 07/04/2025
    /// Coc the initSalesTable method
    /// </summary>
    /// <param name="_salesTable">SalesTable</param>
    /// <param name="_isInvoiceCreator">IsInvoiceCreator</param>
    /// <param name="_numberSeq">NumberSeq</param>
    /// <param name="_salesLineConsolidated">SalesLineConsolidated</param>
    /// <param name="_curParmId">CurParmId</param>
    /// <returns>SalesTable</returns>
    public static SalesTable initSalesTable(
        SalesTable _salesTable,
        boolean _isInvoiceCreator,
        NumberSeq _numberSeq,
        SubBillSalesLineConsolidated _salesLineConsolidated,
        ParmId _curParmId)
    {
        SubBillScheduleTable    scheduleTable   = SubBillScheduleTable::find(_salesLineConsolidated.SubBillBillingScheduleNumber);
        SalesTable              salesTable      = next initSalesTable(_salesTable, _isInvoiceCreator, _numberSeq, _salesLineConsolidated, _curParmId);

        if (scheduleTable)
        {
            salesTable.HMField1 = scheduleTable.HMField1;
            salesTable.HMField2 = scheduleTable.HMField2;
        }

        return salesTable;
    
    }

    /// <summary>
    /// Willie Yao - 07/04/2025
    /// Coc the createSalesOrder method
    /// </summary>
    /// <param name="_createSalesHeader">CreateSalesHeader</param>
    /// <param name="_salesId">SalesId</param>
    /// <param name="salesLineConsolidated">SubBillSalesLineConsolidated</param>
    /// <param name="_consolidateByCustomer">ConsolidateByCustomer</param>
    /// <param name="_splitByItemGroup">SplitByItemGroup</param>
    /// <param name="_returnValues">ReturnValues</param>
    /// <param name="_includeBillingDatesToItem">IncludeBillingDatesToItem</param>
    /// <param name="_isInvoiceCreator">IsInvoiceCreator</param>
    /// <param name="_creditAdjMethod">CreditAdjMethod</param>
    /// <param name="_terminationDate">TerminationDate</param>
    /// <param name="_prorateDaily">ProrateDaily</param>
    /// <param name="_curParmId">CurParmId</param>
    /// <param name="_consolidateAllPeriods">ConsolidateAllPeriods</param>
    /// <returns>container</returns>
    public static container createSalesOrder(
        boolean _createSalesHeader,
        SalesId _salesId,
        SubBillSalesLineConsolidated salesLineConsolidated,
        boolean _consolidateByCustomer,
        boolean _splitByItemGroup,
        container _returnValues,
        boolean _includeBillingDatesToItem,
        boolean _isInvoiceCreator,
        SubBillCreditDeferralAdjMethod _creditAdjMethod,
        SubBillTerminationDate _terminationDate,
        boolean _prorateDaily,
        ParmId _curParmId,
        boolean _consolidateAllPeriods)
    {
        SubBillScheduleTable scheduleTable = SubBillScheduleTable::find(salesLineConsolidated.SubBillBillingScheduleNumber);
        if(!_createSalesHeader)
        {            
            SalesTable salesTableForUpt;
            ttsbegin;
            update_recordset salesTableForUpt
                setting HMField1 = scheduleTable.HMField1,
                        HMField2 = scheduleTable.HMField2
                where salesTableForUpt.SalesId == _salesId;
            ttscommit;
        }

        container newReturnContainer = next createSalesOrder(_createSalesHeader, _salesId, salesLineConsolidated, _consolidateByCustomer, _splitByItemGroup, _returnValues, _includeBillingDatesToItem, _isInvoiceCreator,_creditAdjMethod, _terminationDate, _prorateDaily, _curParmId, _consolidateAllPeriods);        

        return newReturnContainer;
    }

}
```