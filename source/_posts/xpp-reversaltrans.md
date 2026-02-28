---
title: Reverse customer transaction in X++
comments: true
date: 2025-07-11 19:50:44
tags:
 - CustTrans
 - VendTrans
 - Reversal
categories:
 - x++
description: Reverse customer transaction in X++
---
![Reverse customer transaction in X++ technical flow diagram](xpp-reversaltrans/cover.png)

<!--
AI Image Prompt:
Create a clean, minimal, professional technical diagram for Microsoft Dynamics 365 Finance & Operations (D365 F&O). Topic: Reverse customer transaction in X++. Show key components, data flow arrows, extension points, transaction boundaries, and where X++ logic executes. Use simple boxes, labels, and directional connectors on a white background. Style should look like an enterprise architecture blueprint, no decorative art, no characters, no 3D effects.
-->

> Reprinted from: https://community.dynamics.com/blogs/post/?postid=d7f28166-cd5f-469c-979c-4dc412212af1

## Purpose

The purpose of this document is to demonstrate how we can reverse a posted customer transaction through X++. The code below can be used as a script to automate reversal of posted customer transactions.

## Product

Dynamics 365 for Finance and Operations.

## Development

### Code

```c#
class MAKCustTransReversal extends TransactionReversal_Cust
{
    public static MAKCustTransReversal construct()
    {
        return new MAKCustTransReversal();
    }

    public boolean showDialog()
    {
        return false;
    }

    public static void main(Args _args)
    {
        CustTrans custTrans;
        MAKCustTransReversal makCustTransReversal;
        ReasonTable reasonTable;
        ReasonCode reasonCode;
        ReasonRefRecID reasonRefRecID;
        InvoiceId invoiceId;
        Args args;
        ;

        invoiceId = "3392";
        reasonCode = "ERROR";        
        reasonTable = ReasonTable::find(reasonCode);
        reasonRefRecID = ReasonTableRef::createReasonTableRef(
            reasonTable.Reason, reasonTable.Description);

        custTrans = CustTrans::findFromInvoice(invoiceId);
            
        if (custTrans.RecId && !custTrans.LastSettleVoucher)
        {
            args = new Args();
            args.record(custTrans);

            makCustTransReversal = MAKCustTransReversal::construct();
            makCustTransReversal.parmReversalDate(systemDateGet());
            makCustTransReversal.parmReasonRefRecId(reasonRefRecID);
            makCustTransReversal.reversal(args);
            
            info(strFmt("%1 %2 %3 %4 reversed.",
                custTrans.Voucher,
                custTrans.TransDate,
                custTrans.Invoice,
                custTrans.Txt));
        }        
    }
}
```