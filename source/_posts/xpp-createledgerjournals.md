---
title: Create ledger journals in D365FO using X++
comments: true
date: 2025-07-09 19:39:27
tags:
 - LedgerJournalTrans
categories:
 - x++
description: Create ledger journals in D365FO using X++
---

> Reprinted from: https://denistrunin.com/xpptools-createledgerjournal/

Creating LedgerJournalTrans using X++ is quite a common task, but sometimes I still see some mistakes(mostly related to fields initialization from account/offset account and voucher assignment). In this post, I'll try to describe possible options how to create ledger journals in D365FO.

## Test scenario

To test different methods, I wrote a Runnable class DEVTutorialCreateLedgerJournal that creates a new journal based on the data from the existing source journal using different methods. The dialog looks like this

{% asset_img "CopyJournalDialog.png" "CopyJournalDialog" %}

Let's discuss possible copy options(Copy type parameter):

## LedgerJournalEngine class

This method is using LedgerJournalEngine class - the same class that is used when the user creates a journal manually on the Journal lines form. This increases the chances that the resulting line will be the same as a manually created line.

```c#
while select ledgerJournalTransOrig
    order by RecId
            where ledgerJournalTransOrig.JournalNum == _ledgerJournalTableOrig.JournalNum
{
    numLines++;
    if (!ledgerJournalTable.RecId)
    {
        //Often journal name from parameters is specified here
        DEV::validateCursorField(ledgerJournalName, fieldNum(LedgerJournalName, JournalName));

        ledgerJournalTable.clear();
        ledgerJournalTable.initValue();
        ledgerJournalTable.JournalName = ledgerJournalName.JournalName;
        ledgerJournalTable.initFromLedgerJournalName();
        ledgerJournalTable.JournalNum = JournalTableData::newTable(ledgerJournalTable).nextJournalId();
        ledgerJournalTable.Name = strFmt("Copy of %1, Date %2", _ledgerJournalTableOrig.JournalNum, DEV::systemdateget());
        ledgerJournalTable.insert();

        info(strFmt("Journal %1 created", ledgerJournalTable.JournalNum));

        ledgerJournalEngine = LedgerJournalEngine::construct(ledgerJournalTable.JournalType);
        ledgerJournalEngine.newJournalActive(ledgerJournalTable);    }

    ledgerJournalTrans.clear();
    ledgerJournalTrans.initValue();
    ledgerJournalEngine.initValue(ledgerJournalTrans);
    ledgerJournalTrans.JournalNum           =   ledgerJournalTable.JournalNum;
    ledgerJournalTrans.TransDate            =   DEV::systemdateget();
    ledgerJournalTrans.AccountType          =   ledgerJournalTransOrig.AccountType;
    ledgerJournalTrans.modifiedField(fieldNum(LedgerJournalTrans, AccountType));

    ledgerJournalTrans.LedgerDimension = ledgerJournalTransOrig.LedgerDimension;
    if (!ledgerJournalTrans.LedgerDimension)
    {
        throw error("Missing or invalid ledger dimension for journal process");
    }
    ledgerJournalTrans.modifiedField(fieldNum(LedgerJournalTrans, LedgerDimension));
    ledgerJournalEngine.accountModified(LedgerJournalTrans);
    ledgerJournalTrans.OffsetAccountType = ledgerJournalTransOrig.OffsetAccountType;
    ledgerJournalTrans.modifiedField(fieldNum(LedgerJournalTrans, OffsetAccountType));
    ledgerJournalTrans.OffsetLedgerDimension = ledgerJournalTransOrig.OffsetLedgerDimension;
    ledgerJournalTrans.modifiedField(fieldNum(LedgerJournalTrans, OffsetLedgerDimension));
    ledgerJournalEngine.offsetAccountModified(ledgerJournalTrans);

    //amounts
    LedgerJournalTrans.CurrencyCode         =   ledgerJournalTransOrig.CurrencyCode;
    ledgerJournalEngine.currencyModified(LedgerJournalTrans);
    LedgerJournalTrans.AmountCurCredit      =   ledgerJournalTransOrig.AmountCurCredit;
    LedgerJournalTrans.AmountCurDebit       =   ledgerJournalTransOrig.AmountCurDebit;

    //additional fields
    LedgerJournalTrans.Approver           = HcmWorker::userId2Worker(curuserid());
    LedgerJournalTrans.Approved           = NoYes::Yes;
    ledgerJournalTrans.Txt                = ledgerJournalTransOrig.Txt;
    LedgerJournalTrans.SkipBlockedForManualEntryCheck = true;

    DEV::validateWriteRecordCheck(ledgerJournalTrans);
    ledgerJournalTrans.insert();
    ledgerJournalEngine.write(ledgerJournalTrans);}
```

When you modify Account/Offset account fields in this example you need to call two methods(on ledgerJournalTrans and ledgerJournalEngine). This ensures that the line will be properly initialized from the Account/Offset account field.

```c#
ledgerJournalTrans.modifiedField(fieldNum(LedgerJournalTrans, LedgerDimension));  
ledgerJournalEngine.accountModified(LedgerJournalTrans);
```

Voucher assignment here is processing in ledgerJournalEngine.write().

Also, an interesting flag here is LedgerJournalTrans.SkipBlockedForManualEntryCheck. It is useful when you don't want to allow users to create manually lines with the same accounts as in your procedure.

## LedgerJournalTrans defaultRow() method

This is a new approach in D365FO and it uses a new defaultRow() table method. Data entities also call this method during the import process. Its idea is that we don't control the sequence of different modifiedField method calls, we just populate the fields that we know, all other logic happens in the defaultRow() method.

Code for journal creation in this case:

```c#
while select ledgerJournalTransOrig
    order by RecId
    where ledgerJournalTransOrig.JournalNum == _ledgerJournalTableOrig.JournalNum
{
    numLines++;

    if (!ledgerJournalTable.RecId)
    {
        ledgerJournalTable.clear();
        ledgerJournalTable.initValue();
        ledgerJournalTable.JournalName = _ledgerJournalTableOrig.JournalName;
        ledgerJournalTable.initFromLedgerJournalName();
        ledgerJournalTable.JournalNum = JournalTableData::newTable(ledgerJournalTable).nextJournalId();
        ledgerJournalTable.Name = strFmt("Copy of %1, Date %2", _ledgerJournalTableOrig.JournalNum, DEV::systemdateget());
        ledgerJournalTable.insert();

        info(strFmt("Journal %1 created", ledgerJournalTable.JournalNum));
    }

    ledgerJournalTrans.clear();
    ledgerJournalTrans.initValue();

    ledgerJournalTrans.JournalNum             = ledgerJournalTable.JournalNum;
    ledgerJournalTrans.TransDate              = DEV::systemdateget();
    ledgerJournalTrans.AccountType            = ledgerJournalTransOrig.AccountType;
    ledgerJournalTrans.LedgerDimension        = ledgerJournalTransOrig.LedgerDimension;
    ledgerJournalTrans.DefaultDimension       = ledgerJournalTransOrig.DefaultDimension;

    ledgerJournalTrans.OffsetAccountType      = ledgerJournalTransOrig.OffsetAccountType;
    ledgerJournalTrans.OffsetLedgerDimension  = ledgerJournalTransOrig.OffsetLedgerDimension;
    ledgerJournalTrans.OffsetDefaultDimension = ledgerJournalTransOrig.OffsetDefaultDimension;

    ledgerJournalTrans.CurrencyCode           =   ledgerJournalTransOrig.CurrencyCode;
    ledgerJournalTrans.AmountCurCredit        =   ledgerJournalTransOrig.AmountCurCredit;
    ledgerJournalTrans.AmountCurDebit         =   ledgerJournalTransOrig.AmountCurDebit;

    //addition fields
    ledgerJournalTrans.Approver           = HcmWorker::userId2Worker(curuserid());
    ledgerJournalTrans.Approved           = NoYes::Yes;
    ledgerJournalTrans.Txt                = ledgerJournalTransOrig.Txt;
    ledgerJournalTrans.SkipBlockedForManualEntryCheck = true;

    ledgerJournalTrans.defaultRow();
    DEV::validateWriteRecordCheck(ledgerJournalTrans);
    ledgerJournalTrans.insert();
}
```

You can find an example of this approach in the MCRLedgerJournal class (and the usage is here)

## Data entity

We can also use a data entity for journal creation. Technically it produces the same result as the previous method because a data entity at the end will call a table defaultRow() method.

```c#
while select ledgerJournalTransOrig
    order by RecId
    where ledgerJournalTransOrig.JournalNum == _ledgerJournalTableOrig.JournalNum
{
    numLines++;
    select dimensionCombinationEntity
        where dimensionCombinationEntity.RecId == ledgerJournalTransOrig.LedgerDimension;
    select dimensionCombinationEntityOffset
        where dimensionCombinationEntityOffset.RecId == ledgerJournalTransOrig.OffsetLedgerDimension;

    ledgerJournalEntity.initValue();
    ledgerJournalEntity.JournalBatchNumber     = journalNum;
    ledgerJournalEntity.Description            = strFmt("Copy of %1, Date %2", _ledgerJournalTableOrig.JournalNum, DEV::systemdateget());
    ledgerJournalEntity.JournalName            = _ledgerJournalTableOrig.JournalName;
    ledgerJournalEntity.LineNumber++;

    //if you have string and want to convert to ID - DimensionDefaultResolver::newResolver(_dimensionDisplayValue).resolve();
    //AX > General ledge > Chart of accounts > Dimensions > Financial dimension configuration for integrating applications
    ledgerJournalEntity.AccountType                 = ledgerJournalTransOrig.AccountType;
    ledgerJournalEntity.AccountDisplayValue         = dimensionCombinationEntity.DisplayValue;

    ledgerJournalEntity.OffsetAccountType           = ledgerJournalTransOrig.OffsetAccountType;
    ledgerJournalEntity.OffsetAccountDisplayValue   = dimensionCombinationEntityOffset.DisplayValue;

    ledgerJournalEntity.CreditAmount                = ledgerJournalTransOrig.AmountCurCredit;
    ledgerJournalEntity.DebitAmount                 = ledgerJournalTransOrig.AmountCurDebit;
    ledgerJournalEntity.CurrencyCode                = ledgerJournalTransOrig.CurrencyCode;
    ledgerJournalEntity.TEXT                        = ledgerJournalTransOrig.Txt;
    ledgerJournalEntity.TRANSDATE                   = ledgerJournalTransOrig.TransDate;

    ledgerJournalEntity.defaultRow();
    DEV::validateWriteRecordCheck(ledgerJournalEntity);
    ledgerJournalEntity.insert();

    journalNum = ledgerJournalEntity.JournalBatchNumber;
}
```

As you see this method requires less code: we don't even need to write code for the journal header creation, it is all handled by the data entity insert() method. Dimensions can be also specified as strings. However there are some limitations: sometimes data entity doesn't contain all the table fields and it doesn't support all account types.

## Performance testing

Let's test the performance. First I created a test journal with the 1000 lines(**createByCombination method**) and then copied it using these 3 different methods. I got the following results:

|Method|Time to create 1000 lines(sec)|
|-|-|
|Using ledgerJournalEngine|30.54|
|Using DataEntity|33.09|
|Using Table defaultRow method|15.18|

There are some differences between the copy speed in my example, but it is caused by a different logic for the dimension creation, so the result is that all methods are almost equal and quite fast. In a real-life scenario, you can expect an insert speed 10-30 lines per second.

## Choosing the right method and things to avoid

In general, you have 2 options - to create a journal similar to the manual user entry or create it similar to the import procedure(for the second scenario choice between entity and table mostly depends on what input data you have and whether the entity supports all the required fields). So the choice between these two should be made by answering the question: if the user wants to create the same journal manually, does he use manual entry or data import? Use [createJournalUsingLedgerJournalEngine](https://github.com/TrudAX/XppTools/blob/3d69455f4b8157bc28bfc9560e9f70c8bacd634a/DEVTutorial/DEVTutorial/AxClass/DEVTutorialCreateLedgerJournal.xml#L196) as default, as it is more flexible.

Probably in D365FO it is better to avoid creation using JournalTransData classes or when you simply populate ledgerJournalTrans fields and call insert(). This initially can work, but later users may complain - e.g. "*Why when I create a journal manually and specify a vendor account the Due date field is calculated, but your procedure doesn't fill it*".

## Summary

You can download this class using the following link [createJournalUsingLedgerJournalEngine](https://github.com/TrudAX/XppTools/blob/master/DEVTutorial/DEVTutorial/AxClass/DEVTutorialCreateLedgerJournal.xml). The idea is that you can use this code as a template when you have a task to create(or post) a ledger journal using X++. Comments are welcome.