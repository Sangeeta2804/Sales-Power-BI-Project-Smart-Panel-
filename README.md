# Sales-Power-BI-Project-Smart-Panel-divison
A production-grade Sales Business Intelligence pipeline built for Smart Panel Division. This project ingests, cleans, validates, and structures large-scale sales data from raw CSVs into a clean Star Schema optimized for Power BI reporting and analytics.

## Database Schema

┌─────────────────┐
                    │  dim_Products   │
                    │  (ProductID PK) │
                    └────────┬────────┘
                             │
┌──────────────┐    ┌────────▼──────────┐    ┌─────────────────┐
│ dim_Customers│◄───┤ fact_SalesOrders  ├───►│  dim_SalesReps  │
│(CustomerID PK│    │   (OrderID PK)    │    │  (SalesRepID PK)│
└──────────────┘    └────────┬──────────┘    └─────────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
    ┌─────────▼──────┐  ┌────▼───────────┐
    │  fact_Payments │  │  fact_Returns  │
    │ (PaymentID PK) │  │ (ReturnID PK)  │
    └────────────────┘  └────────────────┘

## ETL Pipeline — 9 Steps

CSV Files
   │
   ▼
[STEP 1] Create Staging Tables (NVARCHAR — absorbs 100% dirty data)
   │
   ▼
[STEP 2] BULK INSERT → Staging (BATCHSIZE=100K, MAXERRORS=500, TABLOCK)
   │
   ▼
[STEP 3] Staging Validation (null checks, negative qty, dirty data scan)
   │
   ▼
[STEP 4] Create Production Tables (typed schema — DATE, DECIMAL, INT, BIT)
   │
   ▼
[STEP 5] Clean & Insert → Dimension Tables (Products → Customers → SalesReps)
   │
   ▼
[STEP 6] Clean & Insert → Fact Tables (500K row batches via WHILE loop)
   │
   ▼
[STEP 7] Derived Column UPDATE (GrossAmount, DiscountAmount, NetAmount, TaxAmount, TotalAmount)
   │
   ▼
[STEP 8] Nonclustered Index Creation (post-load — 5–10× faster than pre-load)
   │
   ▼
[STEP 9] Final Validation (row counts, revenue sanity check, overdue analysis)


## Key Technical Highlights

### 1. Two-Stage Loading (Staging → Production)
All raw CSVs are first loaded into staging tables where every column is NVARCHAR(500). This prevents bulk load failures on dirty data. Cleaning, type-casting, and validation happen only during the move to production tables.

### 2. Batch ETL for 5M Row Fact Tables

WHILE @Offset < @TotalRows
BEGIN
    INSERT INTO fact_SalesOrders
    SELECT ... FROM stg_SalesOrders
    ORDER BY OrderID
    OFFSET @Offset ROWS FETCH NEXT @BatchSize ROWS ONLY;

    SET @Offset = @Offset + @BatchSize;
END

- Batch size: 500,000 rows per iteration
- Prevents full table locks during long inserts
- Single batch failure doesn't roll back all 5M rows

###  3. Intelligent Data Cleaning Rules

Column - Rule Applied
Dates  -  ISDATE() validation before CAST
Quantity  -  Must be BETWEEN 1 AND 499
GST%  -  Must be IN (5, 12, 18, 28) — valid Indian GST slabs
Phone  -  10 digits, starts with 6/7/8/9 — Indian mobile format
Email  -  Contains @ check
Booleans  -  Handles YES/TRUE/1/Y → BITE
numsOrderStatus, Channel, Segment — whitelist validation + 'Other' fallback

### 4. MERGE for Idempotent Payment Loads
fact_Payments uses a MERGE statement — safe to re-run without creating duplicates.

### 5. Post-Load Indexes (Performance Optimised)

CREATE NONCLUSTERED INDEX IX_SO_CustomerID  ON fact_SalesOrders (CustomerID);
CREATE NONCLUSTERED INDEX IX_SO_OrderDate   ON fact_SalesOrders (OrderDate);
-- + 8 more indexes across 3 fact tables

Indexes are created after bulk load — avoids index rebuild overhead on every batch insert.

### 6. Derived Amount Columns
GrossAmount    = Quantity * UnitPrice
DiscountAmount = GrossAmount * (Discount_Pct / 100)
NetAmount      = GrossAmount * (1 - Discount_Pct / 100)
TaxAmount      = NetAmount   * (TaxPct / 100)
TotalAmount    = NetAmount   * (1 + TaxPct / 100)

## Analytics-Ready Metrics
Once loaded, the pipeline enables:

📈 Revenue Analysis — Gross, Net, Tax, Total by region / rep / product
💳 Payment Health — Overdue amounts, 60+ day overdue buckets
🔄 Returns Intelligence — Return rate by product, category, reason
🏆 Sales Rep Performance — Actual vs target, incentive calculations
🗓️ Delivery SLA — Computed DeliveryDays per order
🧾 Customer Segmentation — SME, Enterprise, Government, Education etc.














