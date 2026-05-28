# Enterprise-ERP-Inventory-Control-Analytics-System
Decoupled Inventory Analytics Dashboard utilizing optimized MSSQL Stored Procedures, SSRS XML data streams, and javascript, a reactive Vue.js


🏗️ System Architecture
[ SQL Server (MSSQL) ] ──► [ Microsoft SSRS (XML Feed) ] ──► [ Vue.js Web Portal ]

Step 1: Server-Side Optimization (MSSQL)
A dynamic Stored Procedure handles real-time stock balances and triggers a REORDER ALERT if an item's stock drops to 10 units or below. It aggregates data using a Common Table Expression (CTE) to optimize performance.

SQL
USE Inventory_Stock_Check;
GO

CREATE PROCEDURE dbo.GetOptimizedWarehouseStock
AS
BEGIN
    SET NOCOUNT ON; 

    -- 1. Aggregating total active capacity per warehouse
    WITH WarehouseTotal AS (
        SELECT t.WarehouseID,
               SUM(CASE WHEN t.TransactionType = 'IN' THEN t.Quantity ELSE 0 END) -
               SUM(CASE WHEN t.TransactionType = 'OUT' THEN t.Quantity ELSE 0 END) AS Total_Warehouse_Capacity
        FROM FactInventoryTransactions t
        GROUP BY t.WarehouseID
    ),
    -- 2. Calculating current on-hand stock per product
    ProductStock AS (
        SELECT t.WarehouseID, t.ProductID,
               SUM(CASE WHEN t.TransactionType = 'IN' THEN t.Quantity ELSE 0 END) -
               SUM(CASE WHEN t.TransactionType = 'OUT' THEN t.Quantity ELSE 0 END) AS ItemStock
        FROM FactInventoryTransactions t
        GROUP BY t.WarehouseID, t.ProductID
    )
    -- 3. Combining Dimension tables for the Analytics Layer
    SELECT p.ProductID, p.ProductName, w.WarehouseID, w.WarehouseName,
           ps.ItemStock, wt.Total_Warehouse_Capacity,
           CASE WHEN ps.ItemStock <= 10 THEN '🚨 REORDER ALERT' ELSE '✅ STABLE' END AS StockStatus
    FROM ProductStock ps
    INNER JOIN WarehouseTotal wt ON ps.WarehouseID = wt.WarehouseID
    INNER JOIN DimProducts p ON ps.ProductID = p.ProductID
    INNER JOIN DimWarehouses w ON ps.WarehouseID = w.WarehouseID
    WHERE ps.ItemStock > 0;
END;
GO

Step 2: Middle-Tier Data Feed Setup (SSRS)
The query logic is mapped directly into Microsoft Report Builder to serve as a decoupled API endpoint.

Dataset Linkage: Map your report dataset (ds_Warehouse_Contribution) directly to the GetOptimizedWarehouseStock Stored Procedure.

XML Attribute Configuration: In the Tablix Properties, set DataElementOutput to Output and row elements style to Attribute. This forces SSRS to output clean data nodes like <Details ProductID="P001" ItemStock="7" ... />.

API Stream URL: Access the raw data stream instantly via the browser using this URL format:

Plaintext
http://localhost/ReportServer?%2fInventory_Stock_Check%2fds_Warehouse_Contribution&rs:Command=Render&rs:Format=XML
🌐 Step 3: Front-End UI Web Portal (Vue.js + XML Parser)
The client-side interface is built as a single-page dashboard using Bootstrap 5, FontAwesome 6, and Vue.js 2. It natively parses the raw XML string from the SSRS engine into structured data objects.

Core XML Dom Parser Engine:
JavaScript
parseXMLData() {
    const parser = new DOMParser();
    const xmlDoc = parser.parseFromString(this.xmlRawString, "text/xml");
    const elements = xmlDoc.getElementsByTagName("Details");
    
    let list = [];
    for (let i = 0; i < elements.length; i++) {
        list.push({
            productId: elements[i].getAttribute("ProductID"),
            productName: elements[i].getAttribute("ProductName"),
            warehouseId: elements[i].getAttribute("WarehouseID"),
            warehouseName: elements[i].getAttribute("WarehouseName"),
            itemStock: parseInt(elements[i].getAttribute("ItemStock")),
            totalCapacity: elements[i].getAttribute("Total_Warehouse_Capacity"),
            status: elements[i].getAttribute("StockStatus")
        });
    }
    this.stockItems = list; // Bound directly to reactive Vue views
}

Executive Presentation Highlights (For Interview Board)
Architecture Decoupling: The presentation layer has zero dependency on the raw SQL schema. Database adjustments happen inside the Stored Procedure tier without breaking the UI.

Client-Side Compute Optimization: Data is streamed in flat rows. The front-end dynamically normalizes the stream into an associative JavaScript Map() to generate active lists in real-time.

Proactive Monitoring: Incorporating contextual flags (REORDER ALERT) directly into the operational database view demonstrates an enterprise product management mindset.
