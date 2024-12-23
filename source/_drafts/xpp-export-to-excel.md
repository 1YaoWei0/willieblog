---
title: Export to excel
comments: true
date: 2024-12-23 22:20:30
tags:
categories:
 - X++
description: This article describes the way to export to excel
---

### Example Code

```c#
using System.IO;
using OfficeOpenXml;
using OfficeOpenXml.Style;
using OfficeOpenXml.Table;
using OfficeOpenXml.Style.ExcelFillStyle;
using System.Drawing;
class LMSMidStatesPullService extends SysOperationServiceBase
{
    public date                     gFromDate;
    public date                     gToDate;
    public boolean                  hasheader = false;
    public EcoResCategoryId         gCategory;
    public LMSCateGoryTable         gCategoryTable;
    public EcoResCategory           gEcoCategoryTable;
    RetailReportSessionIdentifier   gsessionId = guid2Str(newGuid());

    /// <summary>
    /// LMS_SUN-029403_EXT-00015 Load to Truck Validation
    /// Lance Shi - 7/17/2024
    /// Process
    /// </summary>
    /// <param name = "_contract">LMSCleanUpLoadedLPTableContract</param>
    public void process(LMSMidStatesPullContract _contract)
    {
        gFromDate           = _contract.parmFromDate();
        gToDate             = _contract.parmToDate();
        // LMS_SUN-030708_Mid-StatesPullReport add by Ina Wang on 11/21/2024 begin

        gCategory           = _contract.parmCategory();
        gEcoCategoryTable   = EcoResCategory::find(gCategory);

        select firstonly RecId from gCategoryTable
            where gCategoryTable.EcoResCategory     == gEcoCategoryTable.RecId
              && gCategoryTable.SessionId           == gsessionId;

        if(!gCategoryTable.RecId)
        {
            gCategoryTable.EcoResCategory   = gEcoCategoryTable.RecId;
            gCategoryTable.SessionId        = gsessionId;
            gCategoryTable.insert();
        }

        ttsbegin;
        this.getAllCategory(gEcoCategoryTable.RecId);
        this.createtab(_contract);

        delete_from gCategoryTable
            where gCategoryTable.SessionId == gsessionId;
        ttscommit;
        // LMS_SUN-030708_Mid-StatesPullReport add by Ina Wang on 11/21/2024 end
    }

    /// <summary>
    /// LMS_SUN-030708_Mid-StatesPullReport
    /// Bruce Niu -10/11/2024
    /// Create tab.
    /// </summary>
    /// <param name = "_contract">LMSMidStatesPullContract</param>
    public void createtab(LMSMidStatesPullContract _contract)
    {
        QueryRun                        queryRun    = new QueryRun(_contract.getQuery());
        LMSSalesHistoryView             historyView;
        container                       itemIdCon   = conNull();
        System.IO.Stream                stream;
        ExcelSpreadsheetName            sheet;
        MemoryStream                    memoryStream = new MemoryStream();

        using (var package = new ExcelPackage(memoryStream))
        {
            var currentRow = 1;

            var                             worksheets          = package.get_Workbook().get_Worksheets();
            OfficeOpenXml.ExcelWorksheet    SupplierWorksheet   = worksheets.Add("@LMS:LMSSupplierInformation");
            OfficeOpenXml.ExcelWorksheet    SalesDataWorksheet  = worksheets.Add("@LMS:LMSSalesData");

            itemIdCon = this.generateSalesData(SalesDataWorksheet, _contract);

            if(itemIdCon)
            {
                this.generateSupplier(itemIdCon, SupplierWorksheet);
        
            }

            package.Save();
            file::SendFileToUser(memoryStream, "@LMS:LMSMidStatsPullExcel");
        }
    }

    /// <summary>
    /// LMS_SUN-030708_Mid-StatesPullReport
    /// Bruce Niu -10/11/2024
    /// Get query for range.
    /// </summary>
    /// <param name = "_query">Query</param>
    /// <returns>Query</returns>
    public Query  getQueryForRange(Query _query)
    {
        Query                   query = _query;
        QueryBuildDataSource    qbds = _query.dataSourceTable(tableNum(LMSSalesHistorySummaryView)).addDataSource(tableNum(LMSSalesHistoryView));

        qbds.joinMode(JoinMode::ExistsJoin);
        qbds.addLink(fieldNum(LMSSalesHistorySummaryView, ProductNumber), fieldNum(LMSSalesHistoryView, ProductNumber));

        // LMS_SUN-030708_Mid-StatesPullReport add by Ina Wang on 11/11/2024 begin
        qbds.addLink(fieldNum(LMSSalesHistorySummaryView, Color), fieldNum(LMSSalesHistoryView, Color));
        qbds.addLink(fieldNum(LMSSalesHistorySummaryView, Size), fieldNum(LMSSalesHistoryView, Size));
        // LMS_SUN-030708_Mid-StatesPullReport add by Ina Wang on 11/11/2024 end

        qbds.addRange(fieldNum(LMSSalesHistoryView, SalesHistoryDate)).value(SysQueryRangeUtil::dateRange(gFromDate, gToDate));

        // LMS_SUN-030708_Mid-StatesPullReport add by Ina Wang on 11/21/2024 begin
        if(gCategory)
        {
            EcoResCategoryNamedHierarchyRole    hierarchyId             = EcoResCategoryHierarchyRole::getHierarchiesByRole(EcoResCategoryNamedHierarchyRole::Retail).NamedCategoryHierarchyRole;
            QueryBuildDataSource                qbdsCategoryExpanded    = _query.dataSourceTable(tableNum(LMSSalesHistoryView)).addDataSource(tableNum(EcoResProductCategoryExpanded));

            qbdsCategoryExpanded.joinMode(JoinMode::ExistsJoin);
            qbdsCategoryExpanded.addLink(fieldNum(LMSSalesHistoryView, ProductNumber), fieldNum(EcoResProductCategoryExpanded, ItemId));
            qbdsCategoryExpanded.addRange(fieldNum(EcoResProductCategoryExpanded, NamedCategoryHierarchyRole)).value(queryValue(hierarchyId));

            QueryBuildDataSource    qbdsCategoryTable     = _query.dataSourceTable(tableNum(EcoResProductCategoryExpanded)).addDataSource(tableNum(LMSCateGoryTable));
            qbdsCategoryTable.joinMode(JoinMode::ExistsJoin);
            qbdsCategoryTable.addLink(fieldNum(EcoResProductCategoryExpanded, RecIdCategory), fieldNum(LMSCateGoryTable, EcoResCategory));
            qbdsCategoryTable.addRange(fieldNum(LMSCateGoryTable, SessionId)).value(queryValue(gsessionId));
        }
        // LMS_SUN-030708_Mid-StatesPullReport add by Ina Wang on 11/21/2024 end
        return query;

    }

    /// <summary>
    /// LMS_SUN-030708_Mid-StatesPullReport
    /// Bruce Niu -10/11/2024
    /// Generate sales data.
    /// </summary>
    /// <param name = "_SalesDataWorksheet">ExcelWorksheet</param>
    /// <param name = "_contract">Contract</param>
    /// <returns>container</returns>
    public container generateSalesData(OfficeOpenXml.ExcelWorksheet _SalesDataWorksheet, LMSMidStatesPullContract _contract)
    {
        var                             SalesDatacells      = _SalesDataWorksheet.get_Cells();
        Map                             map                 = new Map(Types::String, Types::Container);
        int                             row                 = 1;
        QueryRun                        queryRun            = new QueryRun(this.getQueryForRange(_contract.getQuery()));
        LMSSalesHistorySummaryView      historyView;
        LMSSalesHistoryView             historyViewRecord;
        container                       supplerRow          = conNull();
        container                       itemIdCon           = conNull();

        OfficeOpenXml.ExcelRange cell = SalesDatacells.get_Item(1, 1);
        cell.set_Value("@LMS:LMSMemberName");

        cell = SalesDatacells.get_Item(1, 2);
        cell.set_Value("@LMS:LMSReportingVendor");

        cell = SalesDatacells.get_Item(1, 3);
        cell.set_Value("@LMS:LMSItemNumber");

        cell = SalesDatacells.get_Item(1, 4);
        cell.set_Value("@LMS:LMSMFGPart");

        cell = SalesDatacells.get_Item(1, 5);
        cell.set_Value("@LMS:LMSItemDesc");

        cell = SalesDatacells.get_Item(1, 6);
        cell.set_Value("@LMS:LMSUPC");

        cell = SalesDatacells.get_Item(1, 7);
        cell.set_Value("@SYS850");

        cell = SalesDatacells.get_Item(1, 8);
        cell.set_Value("@LMS:LMSClassLine");

        cell = SalesDatacells.get_Item(1, 9);
        cell.set_Value("@LMS:LMSSubClassFineLine");

        int currentRow = 9;

        container con = this.generateMonthColumn();

        if(con)
        {
            for(int i = 1; i <= conlen(con); i ++)
            {
                currentRow ++;
                cell = SalesDatacells.get_Item(1, currentRow);
                cell.set_Value(conpeek(con, i));
            }
        }

        currentRow ++;
        cell = SalesDatacells.get_Item(1, currentRow);
        cell.set_Value("@LMS:LMSTotalMonthSales");

        currentRow ++;
        cell = SalesDatacells.get_Item(1, currentRow);
        cell.set_Value("@LMS:LMSRepalcementCost");

        currentRow ++;
        cell = SalesDatacells.get_Item(1, currentRow);
        cell.set_Value("@LMS:LMSCompanyOnHand");

        currentRow ++;
        cell = SalesDatacells.get_Item(1, currentRow);
        cell.set_Value("@LMS:LMSSystemRetail");

        currentRow ++;
        cell = SalesDatacells.get_Item(1, currentRow);
        cell.set_Value("@LMS:LMSAverageUnitRetail");

        currentRow = 2;
        container alldateCollection = this.getAlldateCollection();

        while(queryRun.next())
        {
            historyView                 = queryRun.get(tableNum(LMSSalesHistorySummaryView));

            container   itemIdDim       = this.getItemId(historyView);
            ItemId      itemId          = conPeek(itemIdDim, 1);
            InventDimId inventdIm       = conPeek(itemIdDim, 2);
            str         description     = conPeek(itemIdDim, 3);
            int         currentColumn   ;

            InventTable inventTable;

            select firstonly PrimaryVendorId from inventTable
                where inventTable.ItemId == itemId;

            VendTable   vendTable;

            select firstonly Party from vendTable
                where vendTable.AccountNum == inventTable.PrimaryVendorId;

            if(itemIdCon == conNull())
            {
                itemIdCon += itemId;
            }
            else
            {
                if(!conFind(itemIdCon, itemId))
                {
                    itemIdCon += itemId;
                }
            }

            cell = SalesDatacells.get_Item(currentRow, 1);
            cell.set_Value("@LMS:LMSLMSupply");

            cell = SalesDatacells.get_Item(currentRow, 2);
            cell.set_Value(vendTable.vendorName());

            cell = SalesDatacells.get_Item(currentRow, 3);
            cell.set_Value(historyView.ProductNumber);

            cell = SalesDatacells.get_Item(currentRow, 4);
            cell.set_Value(CustVendExternalItem::find(ModuleInventPurchSalesVendCustGroup::Vend, itemId, inventdIm, inventTable.PrimaryVendorId).ExternalItemId);

            cell = SalesDatacells.get_Item(currentRow, 5);
            cell.set_Value(description);

            cell = SalesDatacells.get_Item(currentRow, 6);
            cell.set_Value(this.getItemBarcode(itemId, inventdIm));

            container conCate = this.getCategory(itemId);

            cell = SalesDatacells.get_Item(currentRow, 7);
            cell.set_Value(conPeek(conCate, 1));

            cell = SalesDatacells.get_Item(currentRow, 8);
            cell.set_Value(conPeek(conCate, 2));

            cell = SalesDatacells.get_Item(currentRow, 9);
            cell.set_Value(conPeek(conCate, 3));

            for(currentColumn  = 10; currentColumn <= conLen(alldateCollection) + 9; currentColumn++)
            {
                container   conLine = conPeek(alldateCollection, currentColumn - 9);

                select sum(Quantity) from historyViewRecord
                    where historyViewRecord.ProductNumber       == historyView.ProductNumber
                       && historyViewRecord.SalesHistoryDate    >= conPeek(conLine, 1)
                       && historyViewRecord.SalesHistoryDate    <= conPeek(conLine, 2);

                cell = SalesDatacells.get_Item(currentRow, currentColumn);
                cell.set_Value(strFmt("%1", historyViewRecord.Quantity));
            
            }

            select sum(Quantity),sum(Amount) from historyViewRecord
                    where historyViewRecord.ProductNumber       == historyView.ProductNumber
                       && historyViewRecord.SalesHistoryDate    >= gFromDate
                       && historyViewRecord.SalesHistoryDate    <= gToDate;

            cell = SalesDatacells.get_Item(currentRow, currentColumn);
            cell.set_Value(strFmt("%1", historyViewRecord.Quantity));

            currentColumn ++;

            cell = SalesDatacells.get_Item(currentRow, currentColumn);
            cell.set_Value(strFmt("%1", InventTableModule::find(itemId, ModuleInventPurchSales::Purch).price));

            currentColumn ++;

            cell = SalesDatacells.get_Item(currentRow, currentColumn);
            cell.set_Value(this.getinventOnhand(itemId, inventdIm));

            currentColumn ++;

            cell = SalesDatacells.get_Item(currentRow, currentColumn);
            cell.set_Value(strFmt("%1", InventTableModule::find(itemId, ModuleInventPurchSales::Sales).price));

            currentColumn ++;

            cell = SalesDatacells.get_Item(currentRow, currentColumn);
            cell.set_Value(strFmt("%1", historyViewRecord.Amount));

            currentRow ++;
        }

        return itemIdCon;
    }

    /// <summary>
    /// BAB_EXT-0002 – Dynamic Seasonality Coefficients
    /// Bruce Niu 11/14/2024
    /// getAllCategory
    /// </summary>
    public void getAllCategory(RecId _parentRecId)
    {
        EcoResCategory  cate;

        while select RecId from cate
            where cate.ParentCategory == _parentRecId
        {
            select firstonly RecId from gCategoryTable
                where gCategoryTable.EcoResCategory     == cate.RecId
                  && gCategoryTable.SessionId           == gsessionId;

            if(!gCategoryTable)
            {
                gCategoryTable.EcoResCategory   = cate.RecId;
                gCategoryTable.SessionId        = gsessionId;
                gCategoryTable.insert();

                this.getAllCategory(cate.recid);
            }
        }
    }

    /// <summary>
    /// LMS_SUN-030708_Mid-StatesPullReport
    /// Bruce Niu -10/11/2024
    /// Get invent on hand.
    /// </summary>
    /// <param name = "_itemId">Item Id</param>
    /// <param name = "_inventdIm">Invent Dim</param>
    /// <returns>the string of inventOnhand</returns>
    public str getinventOnhand(str _itemId, str _inventdIm)
    {
        InventDimParm         dimParm;
        InventOnhand          onhand;
        InventDim             inventDimNew;

        inventDimNew = InventDim::find(_inventdIm);

        dimParm.initFromInventDim(inventDimNew);

        onhand      = InventOnhand::newItemDim(_itemId, inventDimNew, dimParm);

        return strFmt("%1", onhand.physicalInvent());
    }

    /// <summary>
    /// LMS_SUN-030708_Mid-StatesPullReport
    /// Bruce Niu -10/11/2024
    /// Get category.
    /// </summary>
    /// <param name = "_itemId">Item Id</param>
    /// <returns>container</returns>
    public container  getCategory(str _itemId)
    {
        EcoResCategory          c1,c2,c3;
        EcoResProductCategory   ecoResProductCate   = EcoResProductCategory::findByItemIdCategoryHierarchyRole(_itemId, EcoResCategoryNamedHierarchyRole::Retail);

        c1 = EcoResCategory::find(ecoResProductCate.Category);
        c2 = EcoResCategory::find(c1.ParentCategory);
        c3 = EcoResCategory::find(c2.ParentCategory);
        
        return [c3.Name, c2.Name, c1.Name];
    }

    /// <summary>
    /// LMS_SUN-030708_Mid-StatesPullReport
    /// Bruce Niu -10/11/2024
    /// Get Item Barcode
    /// </summary>
    /// <param name = "_itemid">Item Id</param>
    /// <param name = "_inventDimId">Invent dim id</param>
    /// <returns>ItemBarcode</returns>
    public str getItemBarcode(str _itemid, str _inventDimId)
    {
        InventItemBarcode barcode;

        select firstonly itemBarCode from barcode
            where barcode.barcodeSetupId    == '@LMS:LMSUPC'
               && barcode.inventDimId       == _inventDimId
               && barcode.itemId            == _itemid;

        return barcode.itemBarCode;
    }

    /// <summary>
    /// LMS_SUN-030708_Mid-StatesPullReport
    /// Bruce Niu -10/11/2024
    /// Get all date collection.
    /// </summary>
    /// <returns>container</returns>
    public container getAlldateCollection()
    {
        container con = conNull();
        date        fD, tD;
        
        fD = gFromDate;
        tD = endMth(fd);
        
        while(tD <= gToDate)
        {
            con += [[fD, td]];

            fD = nextMth(fD);
            fD = fD - (dayOfMth(fd) - 1);
            tD = endMth(fd);
        }

        if(tD > gToDate && fD <= gToDate)
        {
            con += [[fD, gToDate]];
        }

        return con;
    }

    /// <summary>
    /// LMS_SUN-030708_Mid-StatesPullReport
    /// Bruce Niu -10/11/2024
    /// Generate month column.
    /// </summary>
    /// <returns>container</returns>
    public container generateMonthColumn()
    {
        container   con = conNull();
        date        fd  = gFromDate;
        date        Td  = endMth(gToDate);

        if(gFromDate && gToDate)
        {
            while(fd <= Td)
            {
                con += strFmt("@LMS:LMSSalesUnits", this.getMthName(mthOfYr(fd)), year(fd) mod 1000);

                fd      = nextMth(fd);
            }
        }

        return con;
    }

    /// <summary>
    /// LMS_SUN-030708_Mid-StatesPullReport
    /// Bruce Niu -10/11/2024
    /// Get month name.
    /// </summary>
    /// <param name = "_number">number</param>
    /// <returns>month name string</returns>
    public str getMthName(int _number)
    {
        str mth = '';

        switch(_number)
        {
            case 1:
                mth = '@LMS:LMSJan';
                break;
            case 2:
                mth = '@LMS:LMSFeb';
                break;
            case 3:
                mth = '@LMS:LMSMar';
                break;
            case 4:
                mth = '@LMS:LMSApr';
                break;
            case 5:
                mth = '@LMS:LMSMay';
                break;
            case 6:
                mth = '@LMS:LMSJun';
                break;
            case 7:
                mth = '@LMS:LMSJul';
                break;
            case 8:
                mth = '@LMS:LMSAug';
                break;
            case 9:
                mth = '@LMS:LMSSep';
                break;
            case 10:
                mth = '@LMS:LMSOct';
                break;
            case 11:
                mth = '@LMS:LMSNov';
                break;
            case 12:
                mth = '@LMS:LMSDec';
                break;
        
        }

        return mth;
    
    }

    /// <summary>
    /// LMS_SUN-030708_Mid-StatesPullReport
    /// Bruce Niu -10/11/2024
    /// Generate Supplier
    /// </summary>
    /// <param name = "_con">container</param>
    /// <param name = "_SupplierWorksheet">ExcelWorksheet</param>
    public void generateSupplier(container _con,  OfficeOpenXml.ExcelWorksheet    _SupplierWorksheet)
    {
        var         Suppliercells       = _SupplierWorksheet.get_Cells();
        Map         map                 = new Map(Types::String, Types::Container);
        int         currentColumn       = 1;
        container   vendorCon           = conNull();
        InventTable inventtable;

        OfficeOpenXml.ExcelRange cell = Suppliercells.get_Item(1, 1);
        cell.set_Value("@LMS:LMSItemCategory");

        cell = Suppliercells.get_Item(2, 1);
        cell.set_Value("@LMS:LMSSupplierVendorName");

        cell = Suppliercells.get_Item(3, 1);
        cell.set_Value("@LMS:LMSPaymentTermsDays");

        cell = Suppliercells.get_Item(4, 1);
        cell.set_Value("@LMS:LMSFreightTerms");

        cell = Suppliercells.get_Item(5, 1);
        cell.set_Value("@LMS:LMSFOBOriginPoint");

        cell = Suppliercells.get_Item(6, 1);
        cell.set_Value("@LMS:LMSDelveryMethod");

        cell = Suppliercells.get_Item(7, 1);
        cell.set_Value("@LMS:LMSMinimumOrder");

        cell = Suppliercells.get_Item(8, 1);
        cell.set_Value("@LMS:LMSMinimumOrderAmount");

        cell = Suppliercells.get_Item(9, 1);
        cell.set_Value("@LMS:LMSMidStates");

        cell = Suppliercells.get_Item(10, 1);
        cell.set_Value("@LMS:LMSInitialStoreAllowance");


        cell = Suppliercells.get_Item(11, 1);
        cell.set_Value("@LMS:LMSDefetiveAllowance");


        cell = Suppliercells.get_Item(12, 1);
        cell.set_Value("@LMS:LMSCoopMarketingAllowance");

        cell = Suppliercells.get_Item(13, 1);
        cell.set_Value("@LMS:LMSVolumeRebateAllowance");

        cell = Suppliercells.get_Item(14, 1);
        cell.set_Value("@LMS:LMSOtherDiscounts");

        cell = Suppliercells.get_Item(15, 1);
        cell.set_Value("@LMS:LMSReplacementCost");

        cell = Suppliercells.get_Item(16, 1);
        cell.set_Value('@LMS:LMSReplacementCostInclude');

        cell = Suppliercells.get_Item(17, 1);
        cell.set_Value("@LMS:LMSComments");

        for(int i = 1; i<= conLen(_con); i++)
        {
            select firstonly PrimaryVendorId from inventtable
                where inventtable.ItemId    == conPeek(_con, i);

            if(inventtable.PrimaryVendorId && !conFind(vendorCon, inventtable.PrimaryVendorId))
            {
                vendorCon += inventtable.PrimaryVendorId;
            }
        }

        for(int j = 1; j <= conLen(vendorCon); j++)
        {
            currentColumn ++;

            if(currentColumn == 2)
            {
                cell = Suppliercells.get_Item(1, currentColumn);
                cell.set_Value("@LMS:LMSCurrentSupplier");
            }
            else
            {
                cell = Suppliercells.get_Item(1, currentColumn);
                cell.set_Value(strFmt("@LMS:LMSSupplier", currentColumn - 1));
            }

            VendTable   vendTable = VendTable::find(conPeek(vendorCon, j));

            if(vendTable)
            {
                cell = Suppliercells.get_Item(2, currentColumn);
                cell.set_Value(vendTable.name());

                cell = Suppliercells.get_Item(3, currentColumn);
                cell.set_Value(this.getPaymentTerms(vendTable.CashDisc, vendTable.PaymTermId));

                cell = Suppliercells.get_Item(4, currentColumn);
                cell.set_Value(this.getFreightTerms(vendTable.DlvTerm));

                cell = Suppliercells.get_Item(5, currentColumn);
                cell.set_Value(this.getFOBOriginPoint(vendTable.Party));
                //6
                cell = Suppliercells.get_Item(7, currentColumn);
                cell.set_Value(int2Str(VendTable.LMSMinDollarOrder));

                cell = Suppliercells.get_Item(8, currentColumn);
                cell.set_Value(int2Str(VendTable.LMSMinUnitFreight) + " " + "@SYS314952");

                cell = Suppliercells.get_Item(9, currentColumn);
                cell.set_Value(this.getautoChargeLine(vendTable.accountnum, "@LMS:LMSCBA"));

                //10

                cell = Suppliercells.get_Item(11, currentColumn);
                cell.set_Value(this.getautoChargeLine(vendTable.accountnum, "@LMS:LMSDefective"));

                cell = Suppliercells.get_Item(12, currentColumn);
                cell.set_Value(this.getautoChargeLine(vendTable.accountnum, "@LMS:LMSCoop"));

                // 13 14 15 16 17 
            }
        }
    }

    /// <summary>
    /// LMS_SUN-030708_Mid-StatesPullReport
    /// Bruce Niu -10/11/2024
    /// Get auto Charge Line
    /// </summary>
    /// <param name = "_account">Account</param>
    /// <param name = "_markupcode">markup code</param>
    /// <returns>str</returns>
    public str getautoChargeLine(str _account, str _markupcode)
    {
        MarkupAutoTable autoTable;
        MarkupAutoLine  autoLine;

        select firstonly Value from autoLine
            where autoLine.MarkupCode           == _markupcode
            exists join autoTable
                where autoTable.ModuleType      == MarkupModuleType::Vend
                   && autoTable.AccountCode     == TableGroupAll::Table
                   && autoTable.AccountRelation == _account
                   && autoTable.ModuleCategory  == HeadingLine::Heading
                    && autoTable.ItemCode       == TableGroupAll::All
                   && autoTable.RecId           == autoLine.TableRecId
                   && autoTable.TableId         == autoLine.TableTableId;

        if(autoLine)
        {
            return strFmt("%1 %", autoLine.Value);
        }
        else
        {
            return "";
        }
    }

    /// <summary>
    /// LMS_SUN-030708_Mid-StatesPullReport
    /// Bruce Niu -10/11/2024
    /// Get fob origin point.
    /// </summary>
    /// <param name = "_party">Int64</param>
    /// <returns>str</returns>
    public str getFOBOriginPoint(Int64 _party)
    {
        LogisticsPostalAddress  address     = DirParty::primaryPostalAddress(_party);
        str                     originPoint = "";
        container               con         = conNull();

        con += address.City;
        con += address.State;

        return con2Str(con);
    }

    /// <summary>
    /// LMS_SUN-030708_Mid-StatesPullReport
    /// Bruce Niu -10/11/2024
    /// Get freight terms.
    /// </summary>
    /// <param name = "_dlvTerm">dlvTerm</param>
    /// <returns>FreightTerms str</returns>
    public str getFreightTerms(str _dlvTerm)
    {
        container con = ['@LMS:LMSFFAFOBD', '@LMS:LMSFFAFOBO', '@LMS:LMSPPAFOBD', '@LMS:LMSPPAFOBO', '@LMS:LMSPPDFOBD', '@LMS:LMSPPDFOBO'];

        if(conFind(con, _dlvTerm))
        {
            return 'Yes';
        }
        else
        {
            return 'No';
        }
    
    }

    /// <summary>
    /// LMS_SUN-030708_Mid-StatesPullReport
    /// Bruce Niu -10/11/2024
    /// Get Payment Terms
    /// </summary>
    /// <param name = "_CashDisc">CashDisc</param>
    /// <param name = "_PaymTermId">PaymTermId</param>
    /// <returns>str</returns>
    public str getPaymentTerms(str _CashDisc, str _PaymTermId)
    {
        CashDisc    cd  = CashDisc::find(_CashDisc);
        container   con = conNull();
        PaymTerm    pt  = PaymTerm::find(_PaymTermId);

        if(cd.NumOfDays)
        {
            con += int2Str(cd.NumOfDays);
        }

        if(pt.PaymMethod)
        {
            con += enum2Str(pt.PaymMethod);
        }

        if(pt.NumOfDays)
        {
            con += int2Str(pt.NumOfDays);
        }

        if(conLen(con) > 0)
        {
            return con2Str(con, " ");
        }
        else
        {
            return "";
        }
    }

    /// <summary>
    /// LMS_SUN-030708_Mid-StatesPullReport
    /// Bruce Niu -10/11/2024
    /// Get item id.
    /// </summary>
    /// <param name = "_displayProductNumber">DisplayProductNumber</param>
    /// <returns>container</returns>
    public container getItemId(LMSSalesHistorySummaryView _summaryView)
    {
        InventTable             inventtable;
        InventDimCombination    inventDimCombination;
        EcoResProduct           ecoResProduct;

        // LMS_SUN-030708_Mid-StatesPullReport add by Ina Wang on 11/11/2024 begin
        InventDim               inventDimLoc;

        select firstonly ItemId, InventDimId, RetailVariantId, DistinctProductVariant from inventDimCombination
            where inventDimCombination.ItemId       == _summaryView.ProductNumber
            exists join inventDimLoc
                where inventDimLoc.inventDimId      == inventDimCombination.InventDimId
                    && inventDimLoc.InventColorId   == _summaryView.Color
                    && inventDimLoc.InventSizeId    == _summaryView.Size;
        
        if(inventDimCombination.ItemId)
        {
            return [inventDimCombination.ItemId,inventDimCombination.InventDimId, inventDimCombination.productDescription("en-us")];
        }
        else
        {
            select firstonly ItemId, Product from inventtable
                exists join ecoResProduct
                    where ecoResProduct.RecId                   == inventtable.Product
                       && ecoResProduct.DisplayProductNumber    == _summaryView.ProductNumber;

            return [inventtable.ItemId, InventDim::inventDimIdBlank(), inventtable.productDescription("en-us")];
        }

        // LMS_SUN-030708_Mid-StatesPullReport add by Ina Wang on 11/11/2024 end
    }
}
```