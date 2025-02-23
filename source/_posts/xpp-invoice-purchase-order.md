---
title: Invoice Purchase Order Example
comments: true
date: 2025-02-23 20:51:59
tags:
categories:
 - d365
 - x++
description: Invoice Purchase Order Example
---

> 本文记录了 x++ 代码 invoice 采购订单的示例代码

```c#
Here is the C# code you provided with all comments removed:

```csharp
public class GNCAutoInvoiceVendConsignedPOBatch extends RunBaseBatch implements BatchRetryable
{
    SysLookupMultiSelectValues  gInvoiceIds;
    boolean                     gCreateParmTable;
    PurchTable                  gPurchTable;
    boolean                     gMultiPOs;
    PurchId                     gPurchId;
    RecordInsertList            gParmLineRecordInsertList, gParmSubLineRecordInsertList, gMarkupTrans;
    systemSequence              gSystemSequence;
    Num                         gInvoiceNumber;
    TransDate                   gInvoiceDate        = DateTimeUtil::getSystemDate(DateTimeUtil::getUserPreferredTimeZone());
    InvoiceDescriptionLarge     gDescription;
    Map                         gTableRefIdMap      = new Map(Types::String, Types::String);
    Map                         gItemMap            = new Map(Types::String, Types::Real);
    DialogField     gDlgInvoiceIds;

    #define.CurrentVersion(1)
    #localmacro.CurrentList
        gInvoiceIds
    #endmacro

    public boolean canGoBatch()
    {
        return true;
    }

    public boolean canGoBatchJournal()
    {
        return true;
    }

    public Object dialog()
    {
        DialogRunbase       dialog = super();
    
        gDlgInvoiceIds = dialog.addFieldValue(extendedTypeStr(SysLookupMultiSelectValues), gInvoiceIds, "@SYS12128");
    
        return dialog;
    }

    public void dialogPostRun(DialogRunbase _dialog)
    {
        super(_dialog);

        Query                   query;
        QueryBuildDataSource    qbds;

        query   = new Query(queryStr(GNCAutoInvoiceVendConsignedLookupQuery));

        SysLookupMultiSelectCtrl::constructWithQuery(
                    _dialog.dialogForm().formRun(),
                    gDlgInvoiceIds.control(),
                    query,
                    false,
                    [tableNum(CustInvoiceJour), fieldNum(CustInvoiceJour, InvoiceId)]
        );
    }

    public boolean getFromDialog()
    {
        gInvoiceIds = gDlgInvoiceIds.value();
    
        return super();
    }

    public container pack()
    {
        return [#CurrentVersion, #CurrentList];
    }

    static GNCAutoInvoiceVendConsignedPOBatch construct()
    {
        return new GNCAutoInvoiceVendConsignedPOBatch();
    }

    static ClassDescription description()
    {
        return "@GNC:AutoInvoiceVendorConsignedPurchaseOrder";
    }

    static void main(Args _args)
    {
        GNCAutoInvoiceVendConsignedPOBatch      batch;
    
        batch = GNCAutoInvoiceVendConsignedPOBatch::construct();
    
        if (batch.prompt())
        {
            batch.runOperation();
        }
    }

    protected boolean canRunInNewSession()
    {
        return false;
    }

    [Hookable(false)]
    public final boolean isRetryable()
    {
        return false;
    }

    public Qty getFinalInvoiceQty(ItemId _itemId, Qty _qty)
    {
        Qty retQty, storeQty;

        GNCProcessedVendorConsignedInvAdjustments processedVendorConsignedInvAdjustments = GNCProcessedVendorConsignedInvAdjustments::find(_itemId, true);

        if (processedVendorConsignedInvAdjustments.RecId)
        {
            ttsbegin;
            storeQty = processedVendorConsignedInvAdjustments.Qty;
            retQty = _qty - storeQty;
            processedVendorConsignedInvAdjustments.Qty = retQty < 0 ? abs(retQty) : 0;
            processedVendorConsignedInvAdjustments.update();
            ttscommit;
        }
        else
        {
            retQty = _qty;
        }

        return retQty;
    }

    public void run()
    {
        #OCCRetryCount

        setPrefix("@GNC:AutoInvoiceVendorConsignedPurchaseOrder");
        
        query                       query_Item      = new Query(queryStr(GNCAutoInvoiceVendConsignedItemQuery));
        QueryBuildDataSource        qbds;
        SysLookupMultiSelectValues  qrvInvoiceIds   = strReplace(gInvoiceIds, ';', ',');
        
        if (qrvInvoiceIds)
        {
            qbds = query_Item.dataSourceTable(tableNum(CustInvoiceJour));
            qbds.addRange(fieldNum(CustInvoiceJour, InvoiceId)).value(qrvInvoiceIds);
        }

        QueryRun                    queryRunItem    = new QueryRun(query_Item);
        TransDateTime               earliestDateTime;

        while (queryRunItem.next())
        {
            CustInvoiceTrans custInvoiceTrans_Item = queryRunItem.get(tableNum(CustInvoiceTrans));

            Qty finalInvoiceQty = this.getfinalInvoiceQty(custInvoiceTrans_Item.ItemId, custInvoiceTrans_Item.Qty);
            PurchLine   purchLine_earliest;
            PurchTable  purchTable_earliest;
            
            select firstonly minof(CreatedDateTime) from purchTable_earliest
                where purchTable_earliest.PurchaseType              == PurchaseType::Purch
                exists join purchLine_earliest
                    where purchLine_earliest.PurchId                == purchTable_earliest.PurchId
                        && !purchLine_earliest.Blocked
                        && !purchLine_earliest.IsDeleted
                        && (purchLine_earliest.PurchStatus          == PurchStatus::Backorder
                        || purchLine_earliest.PurchStatus           == PurchStatus::Received)
                        && purchLine_earliest.ItemId                == custInvoiceTrans_Item.ItemId
                        && purchLine_earliest.RemainPurchFinancial  > 0;

            if (!earliestDateTime || earliestDateTime > purchTable_earliest.CreatedDateTime)
            {
                earliestDateTime = purchTable_earliest.CreatedDateTime;
            }
        }

        if (!gItemMap.empty())
        {
            while select gPurchTable
                where gPurchTable.PurchaseType      == PurchaseType::Purch
                    && gPurchTable.CreatedDateTime  >= earliestDateTime
                    && (gPurchTable.PurchStatus     == PurchStatus::Backorder
                    || gPurchTable.PurchStatus      == PurchStatus::Received)
            {
                try
                {
                    this.processSinglePO();
                }
                catch (Exception::Deadlock)
                {
                    retry;
                }
                catch (Exception::UpdateConflict)
                {
                    if (appl.ttsLevel() == 0)
                    {
                        if (xSession::currentRetryCount() >= #RetryNum)
                        {
                            throw Exception::UpdateConflictNotRecovered;
                        }
                        else
                        {
                            retry;
                        }
                    }
                    else
                    {
                        throw Exception::UpdateConflict;
                    }
                }
                catch
                {
                    if (appl.ttsLevel() != 0)
                    {
                        ttsabort;
                    }
                    continue;
                }
            }

            this.insertExceptions();
        }


        Query   query_jour = new Query();

        qbds = query_jour.addDataSource(tableNum(CustInvoiceJour));
        
        if (qrvInvoiceIds)
        {
            qbds = query_jour.dataSourceTable(tableNum(CustInvoiceJour));
            qbds.addRange(fieldNum(CustInvoiceJour, InvoiceId)).value(qrvInvoiceIds);
        }

        QueryRun    queryRunJour    = new QueryRun(query_jour);
        
        while (queryRunJour.next())
        {
            try
            {
                CustInvoiceJour custInvoiceJourUpd = queryRunJour.get(tableNum(CustInvoiceJour));

                custInvoiceJourUpd.selectForUpdate(true);

                ttsbegin;
                custInvoiceJourUpd.GNCProcessedVendConsignedInvoice = true;
                custInvoiceJourUpd.update();
                ttscommit;
            }
            catch (Exception::Deadlock)
            {
                retry;
            }
            catch (Exception::UpdateConflict)
            {
                if (appl.ttsLevel() == 0)
                {
                    if (xSession::currentRetryCount() >= #RetryNum)
                    {
                        throw Exception::UpdateConflictNotRecovered;
                    }
                    else
                    {
                        retry;
                    }
                }
                else
                {
                    throw Exception::UpdateConflict;
                }
            }
            catch
            {
                if (appl.ttsLevel() != 0)
                {
                    ttsabort;
                }
                continue;
            }
        }
    }

    public void processSinglePO()
    {
        PurchLine                   purchLine, prevPurchLine;
        VendPackingSlipTrans        vendPackingSlipTrans;
        Map                         packingSlipTransMap = new Map(Types::Int64, Types::Real);
        QueryRun                    queryRun;
        CustInvoiceTrans            custInvoiceTrans;
        Qty                         qtyToInvoice;
            
        gPurchId = gPurchTable.PurchId;

        this.initParameters();
        this.initRecordLists();

        while select purchLine
            where purchLine.PurchId                             == gPurchTable.PurchId
                && !purchLine.Blocked
                && !purchLine.IsDeleted
                && (purchLine.PurchStatus                       == PurchStatus::Backorder
                || purchLine.PurchStatus                        == PurchStatus::Received)
                && purchLine.RemainPurchFinancial               > 0
                join vendPackingSlipTrans
                    where vendPackingSlipTrans.InventTransId    == purchLine.InventTransId
        {
            Qty         purchRemain = vendPackingSlipTrans.remainPurchFinancial();
            Qty         curQty;
            
            if (!purchRemain)
            {
                continue;
            }

            if (!gItemMap.exists(purchLine.ItemId))
            {
                continue;
            }

            qtyToInvoice = gItemMap.lookup(purchLine.ItemId);

            if (!qtyToInvoice)
            {
                continue;
            }

            if (qtyToInvoice > purchRemain)
            {
                curQty          = purchRemain;
                qtyToInvoice    -= purchRemain;
            }
            else
            {
                curQty          = abs(qtyToInvoice);
                qtyToInvoice    = 0;
            }

            if (purchLine.RecId != prevPurchLine.RecId)
            {
                if (packingSlipTransMap.elements() > 0)
                {
                    this.createParmLineAndSubLines(prevPurchLine, packingSlipTransMap.pack());
                }
                prevPurchLine       = purchLine.data();
                packingSlipTransMap = new Map(Types::Int64, Types::Real);
            }

            packingSlipTransMap.insert(vendPackingSlipTrans.RecId, curQty);
            gItemMap.insert(purchLine.ItemId, qtyToInvoice);
        }

        if (prevPurchLine.RecId && packingSlipTransMap.elements() > 0)
        {
            this.createParmLineAndSubLines(prevPurchLine, packingSlipTransMap.pack());
        }

        if (gCreateParmTable)
        {
            gInvoiceNumber = gPurchTable.PurchId;

            VendInvoiceJour vendInvoiceJour;
            Counter         i;

            select firstonly RecId from vendInvoiceJour
                where vendInvoiceJour.InvoiceId == gInvoiceNumber;

            while (vendInvoiceJour.RecId)
            {
                i++;
                vendInvoiceJour.clear();

                gInvoiceNumber = strfmt('%1_%2', gPurchTable.PurchId, i);

                select firstonly RecId from vendInvoiceJour
                    where vendInvoiceJour.InvoiceId == gInvoiceNumber;
            }

            VendInvoiceInfoTable        vendInvoiceInfoTable;

            ttsbegin;
            vendInvoiceInfoTable = this.createParmTable();

            this.insertRecordLists();
            
            PurchFormLetter_Invoice purchFormLetter = PurchFormLetter_Invoice::newFromSavedInvoice(vendInvoiceInfoTable);
            purchFormLetter.purchTable(gPurchTable);
            purchFormLetter.printFormLetter(false);
            purchFormLetter.proforma(false);
            purchFormLetter.parmId(vendInvoiceInfoTable.ParmId);
            purchFormLetter.update(vendInvoiceInfoTable, vendInvoiceInfoTable.Num);

            ttscommit;

            select firstonly RecId from vendInvoiceJour
                where vendInvoiceJour.InvoiceId == vendInvoiceInfoTable.Num;

            if (vendInvoiceJour)
            {
                Info(strFmt("Vendor invoice journal %1 is posted.", vendInvoiceInfoTable.Num));
            }
        }
    }

    public void insertExceptions()
    {
        MapEnumerator itemME = gItemMap.getEnumerator();

        while (itemME.moveNext())
        {
            ItemId                      itemId  = itemME.currentKey();
            Qty                         qtyExce = itemME.currentValue(); 

            if (!qtyExce)
            {
                continue;
            }

            query                       query   = new Query(queryStr(GNCAutoInvoiceVendConsignedQuery));
            QueryBuildDataSource        qbds;
            SysLookupMultiSelectValues  qrvInvoiceIds   = strReplace(gInvoiceIds, ';', ',');
            QueryRun                    queryRun;
            
            qbds = query.dataSourceTable(tableNum(CustInvoiceTrans));
            qbds.addRange(fieldNum(CustInvoiceTrans, ItemId)).value(itemId);

            if (qrvInvoiceIds)
            {
                qbds = query.dataSourceTable(tableNum(CustInvoiceJour));
                qbds.addRange(fieldNum(CustInvoiceJour, InvoiceId)).value(qrvInvoiceIds);
            }

            queryRun = new QueryRun(query);

            while (queryRun.next())
            {
                CustInvoiceTrans    custInvoiceTrans = queryRun.get(tableNum(CustInvoiceTrans));
                Qty                 qtyRemain, curQty;

                qtyRemain = custInvoiceTrans.Qty;

                if (qtyExce > qtyRemain)
                {
                    curQty      = qtyRemain;
                    qtyExce     -= qtyRemain;
                }
                else
                {
                    curQty      = qtyExce;
                    qtyExce     = 0;
                }

                GNCAutoInvoiceVendConsignedException::create(gInvoiceDate, 
                    custInvoiceTrans.SalesId, 
                    custInvoiceTrans.InvoiceDate, 
                    custInvoiceTrans.ItemId, 
                    custInvoiceTrans.Qty, 
                    curQty);

                if (!qtyExce)
                {
                    break;
                }
            }
        }
    }

    public void initParameters()
    {
        gSystemSequence     = new systemSequence();
        gCreateParmTable    = false;
    }

    public TradeLineRefId getTableRefId()
    {
        if (!gTableRefIdMap.exists(gPurchId))
        {
            gTableRefIdMap.insert(gPurchId, formletterParmData::getNewTableRefId());
        }

        return gTableRefIdMap.lookup(gPurchId);
    }

    public void initRecordLists()
    {
        gParmLineRecordInsertList       = new RecordInsertList(tableNum(VendInvoiceInfoLine));
        gParmSubLineRecordInsertList    = new RecordInsertList(tableNum(VendInvoiceInfoSubLine));
        gMarkupTrans    = new RecordInsertList(tableNum(MarkupTrans));
    }

    public void insertParmLine(VendInvoiceInfoLine _vendInvoiceInfoLine, PurchLine _purchLine)
    {
        gSystemSequence.suspendRecIds(_vendInvoiceInfoLine.TableId);
        _vendInvoiceInfoLine.RecId  = gSystemSequence.reserveValues(1, _vendInvoiceInfoLine.TableId);

        gParmLineRecordInsertList.add(_vendInvoiceInfoLine);
    }

    public void createVendInvoiceMatchingLine(PurchLine _purchLine, VendDocumentLineMap _parmLine)
    {
        VendInvoiceMatchingLine vendInvoiceMatchingLine;

        if (_parmLine.TableId && _parmLine.RecId && _purchLine)
        {
            vendInvoiceMatchingLine.clear();
            vendInvoiceMatchingLine.initExpectedValues(_purchLine, _parmLine);
            vendInvoiceMatchingLine.RefTableId = _parmLine.TableId;
            vendInvoiceMatchingLine.RefRecId = _parmLine.RecId;
            vendInvoiceMatchingLine.insert();
        }
    }

    public void createMarkupTrans(PurchLine _purchLine, VendInvoiceInfoLine _vendInvoiceInfoLine)
    {
        MarkupTrans markupTrans, markupTransNew;

        while select markupTrans
            where markupTrans.TransTableId  == _purchLine.TableId
                && markupTrans.TransRecId   == _purchLine.RecId
                && markupTrans.Value        != 0
        {
            buf2Buf(markupTrans, markupTransNew);
            markupTransNew.TransTableId       = _vendInvoiceInfoLine.TableId;
            markupTransNew.TransRecId         = _vendInvoiceInfoLine.RecId;
            gMarkupTrans.add(markupTransNew);
        }
    }

    public void insertParmSubLine(VendInvoiceInfoSubLine _vendInvoiceInfoSubLine)
    {
        if (_vendInvoiceInfoSubLine.ReceiveNow == 0)
        {
            return;
        }

        gParmSubLineRecordInsertList.add(_vendInvoiceInfoSubLine);
    }

    public void insertRecordLists()
    {
        gParmLineRecordInsertList.insertDatabase();
        gParmSubLineRecordInsertList.insertDatabase();
        gMarkupTrans.insertDatabase();
    }

    public void createParmLineAndSubLines(PurchLine _purchLine, container _packedSubLinesMap)
    {
        MapEnumerator           me;
        VendInvoiceInfoLine     vendInvoiceInfoLine;
        Qty                     newPostingQty;
        VendPackingSlipTrans    vendPackingSlipTrans;
        Set                     subLineSet      = new Set(Types::Record);

        me = Map::create(_packedSubLinesMap).getEnumerator();

        while (me.moveNext())
        {
            RefRecId        vendPackingSlipTransRecId   = me.currentKey();
            Qty             qty                         = me.currentValue();

            vendPackingSlipTrans            = VendPackingSlipTrans::findRecId(vendPackingSlipTransRecId);

            vendPackingSlipTrans.Qty        = qty;
            vendPackingSlipTrans.InventQty  = qty;
            newPostingQty                   += qty;

            subLineSet.add(vendPackingSlipTrans);
        }
        
        if (newPostingQty == 0)
        {
            return;
        }

        vendInvoiceInfoLine = this.createParmLine(_purchLine, newPostingQty);

        this.createParmSubLines(vendInvoiceInfoLine, subLineSet);
        
        gCreateParmTable = true;
    }

    public VendInvoiceInfoLine createParmLine(PurchLine _purchLine, Qty _receiveNow)
    {
        VendInvoiceInfoLine vendInvoiceInfoLine;

        vendInvoiceInfoLine.Ordering        = DocumentStatus::Invoice;
        vendInvoiceInfoLine.TableRefId      = this.getTableRefId();
        vendInvoiceInfoLine.DocumentOrigin  = DocumentOrigin::Manual;
        vendInvoiceInfoLine.initFromPurchLine(_purchLine);
        vendInvoiceInfoLine.defaultRow(_purchLine, null, _receiveNow, _receiveNow);

        this.insertParmLine(vendInvoiceInfoLine, _purchLine);
        
        gCreateParmTable = true;

        return vendInvoiceInfoLine;
    }

    public void createParmSubLines(VendInvoiceInfoLine _vendInvoiceInfoLine, Set _subLineSet)
    {
        VendPackingSlipTrans    vendPackingSlipTrans;
        VendInvoiceInfoSubLine  vendInvoiceInfoSubLine;
        SetEnumerator           se = _subLineSet.getEnumerator();

        while (se.moveNext())
        {
            vendPackingSlipTrans = se.current();

            vend

InvoiceInfoSubLine.clear();
            vendInvoiceInfoSubLine.initFromLine(_vendInvoiceInfoLine);
            vendInvoiceInfoSubLine.initFromVendPackingSlipTrans(vendPackingSlipTrans);
            this.insertParmSubLine(vendInvoiceInfoSubLine);
        }

        if (!_vendInvoiceInfoLine.vendInvoiceInfoSubTable())
        {
            VendInvoiceInfoSubTable::createFromVendInvoiceInfoLine(_vendInvoiceInfoLine, _vendInvoiceInfoLine.ParmId, gDescription, true);
        }
    }

    public VendInvoiceInfoTable createParmTable()
    {
        VendInvoiceInfoTable        vendInvoiceInfoTable;

        vendInvoiceInfoTable.clear();
        vendInvoiceInfoTable.initValue();
        vendInvoiceInfoTable.defaultRow(gPurchTable);
        vendInvoiceInfoTable.initFromPurchTable(gPurchTable);
        vendInvoiceInfoTable.DocumentOrigin         = DocumentOrigin::Manual;
        vendInvoiceInfoTable.Num                    = gInvoiceNumber;
        vendInvoiceInfoTable.VendInvoiceSaveStatus  = VendInvoiceSaveStatus::Pending;
        vendInvoiceInfoTable.Description            = 'Retail POS Sale';
        vendInvoiceInfoTable.DocumentDate           = gInvoiceDate;
        vendInvoiceInfoTable.TransDate              = gInvoiceDate;
        vendInvoiceInfoTable.LastMatchVariance      = LastMatchVarianceOptions::OK;
        vendInvoiceInfoTable.ParmJobStatus          = ParmJobStatus::Waiting;
        vendInvoiceInfoTable.Approved               = NoYes::Yes;
        vendInvoiceInfoTable.Approver               = HcmWorker::userId2Worker(curUserId());
        vendInvoiceInfoTable.TableRefId             = this.getTableRefId();
        vendInvoiceInfoTable.Ordering               = DocumentStatus::Invoice;
        vendInvoiceInfoTable.initInvoiceTotals();
        vendInvoiceInfoTable.insert();
        
        return vendInvoiceInfoTable;
    }

    public void createParmSubTable(VendInvoiceInfoTable _vendInvoiceInfoTable)
    {
        VendInvoiceInfoSubTable::createFromVendInvoiceInfoTable(_vendInvoiceInfoTable, true);
    }
}
```