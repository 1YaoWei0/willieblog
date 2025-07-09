---
title: Extend CustTrans and VendTrans
comments: true
date: 2025-07-09 19:48:50
tags:
 - CustTrans
 - VendTrans
 - Reversal
 - LedgerJournalTrans
categories:
 - x++
description: Extend CustTrans and VendTrans
---

> Just recorded how to extend the feature regards the CustTrans and VendTrans.

## Bring the customized fields of LedgerJournalTrans to CustTrans or VendTrans

If the requirement wants to create the new fields on CustTrans/VendTrans and LedgerJournalTrans, and want to sync the customized fields to Transaction record after posting the customer journal or vendor journal. You can extend the CustVoucherJournal/VendVoucherJournal as follows:

```c#
/// <summary>
/// Willie Yao - 07/08/2025
/// </summary>
[ExtensionOf(classstr(CustVoucherJournal))]
final class HMClass_CustVoucherJournal_Extension
{
    /// <summary>
    /// Willie Yao - 07/08/2025
    /// </summary>
    /// <param name = "custVendTrans">CustVendTrans</param>
    /// <param name = "_ledgerPostingJournal">LedgerVoucher</param>
    /// <param name = "_useSubLedger">UseSubLedger</param>
    protected void initCustVendTrans(
        CustVendTrans   custVendTrans, 
        LedgerVoucher   _ledgerPostingJournal, 
        boolean         _useSubLedger)
    {
        next initCustVendTrans(custVendTrans, _ledgerPostingJournal, _useSubLedger);
        
        LedgerJournalTrans ledgerJournalTrans;

        if (common.TableId == tablenum(LedgerJournalTrans))
        {
            ledgerJournalTrans = common;

            if (custVendTrans.TableId == tableNum(CustTrans))
            {
                CustTrans custTrans  = custVendTrans;
                custTrans.HMField1   = ledgerJournalTrans.HMField1;
                custTrans.HMField2   = ledgerJournalTrans.HMField2;
                custVendTrans        = custTrans;
            }

        }
    }
}

/// <summary>
/// Willie Yao - 07/08/2025
/// </summary>
[ExtensionOf(classstr(VendVoucherJournal))]
final class HMClass_VendVoucherJournal_Extension
{
    /// <summary>
    /// Willie Yao - 07/08/2025
    /// </summary>
    /// <param name = "custVendTrans">CustVendTrans</param>
    /// <param name = "_ledgerPostingJournal">LedgerVoucher</param>
    /// <param name = "_useSubLedger">UseSubLedger</param>
    protected void initCustVendTrans(
        CustVendTrans   custVendTrans, 
        LedgerVoucher   _ledgerPostingJournal, 
        boolean         _useSubLedger)
    {
        next initCustVendTrans(custVendTrans, _ledgerPostingJournal, _useSubLedger);

        LedgerJournalTrans ledgerJournalTrans;

        if (common.TableId == tablenum(LedgerJournalTrans))
        {
            ledgerJournalTrans = common;

            if (custVendTrans.TableId == tableNum(VendTrans))
            {
                VendTrans vendTrans  = custVendTrans;
                vendTrans.HMField1   = ledgerJournalTrans.HMField1;
                custVendTrans        = vendTrans;
            }

        }
    }

}
```

## Add new validation logic when posted LedgerJournalTrans

Please coc the validation logic of LedgerJournalCheckPost class if developer wants to add new validation logic for LedgerJournalTrans post operation.

```c#
/// <summary>
/// Willie Yao - 07/08/2025
/// </summary>
[ExtensionOf(classstr(LedgerJournalCheckPost))]
final class HMClass_LedgerJournalCheckPost_Extension
{
    /// <summary>
    /// Willie Yao - 07/08/2025
    /// </summary>
    /// <param name = "_calledFrom">CalledFrom</param>
    /// <returns>True if validate right</returns>
    public boolean validate(Object _calledFrom)
    {
        boolean ret = next validate(_calledFrom);

        ret = this.newValidation() && ret;

        return ret;
    }

    /// <summary>
    /// Willie Yao - 07/08/2025
    /// </summary>
    /// <returns>True if validate right</returns>
    internal boolean newValidation()
    {
        LedgerJournalTrans  ledgerJournalTrans;
        boolean             ok = true;

        ok = this.newValidationLogic(ledgerJournalTrans) && ok;

        return ok;    
    }
}
```

## Extend CustTrans/VendTrans Reversal Logic

Developers must coc the reversal method and write the customized logic after the next invoke method.

```c#
/// <summary>
/// Willie Yao - 07/08/2025
/// </summary>
[ExtensionOf(classstr(TransactionReversal_Cust))]
final class HMClass_TransactionReversal_Cust_Extension
{ 
    /// <summary>
    /// Willie Yao - 07/08/2025
    /// </summary>
    /// <param name = "args">Args</param>
    void reversal(Args args)
    {
        next reversal(args);

        // New Logic
    }    
}

/// <summary>
/// Willie Yao - 07/08/2025
/// </summary>
[ExtensionOf(classstr(TransactionReversal_Vend))]
final class HMClass_TransactionReversal_Vend_Extension
{
    /// <summary>
    /// Willie Yao - 07/08/2025
    /// </summary>
    /// <param name = "args">Args</param>
    void reversal(Args args)
    {
        next reversal(args);

        // New Logic
    }  
}
```

> More expansion points will be updated successively in the future