# 📊 Retail Sale & Stock Analytics — Power BI Data Model

A Power BI solution for tracking inventory movement and sales performance across retail fashion stores in South India.
 
---
 
## 🗂️ Project Structure
 
```
├── Sales Table          # Point-of-sale transactions (fact table)
├── GRN Table            # Goods received / stock movement (fact table)
├── Dim_Stylecode        # Style & category dimension
├── Dim_Date             # Calendar dimension (2023–2026)
├── Dim_Location         # Store & warehouse location dimension
├── Dim_Size             # Size dimension
└── Style Sheet          # Style master metadata
```
 
---
 
## 📦 Data Sources
 
### Sales Table
Captures every line item sold at POS terminals across all stores.
 
| Key Column | Description |
|---|---|
| `OrderNumber` | Unique bill identifier |
| `Order_Date` | Date of sale → joins `Dim_Date` |
| `Storename` | Store name → joins `Dim_Location` |
| `StyleCode` | Style → joins `Dim_Stylecode` |
| `Size` | Size → joins `Dim_Size` |
| `Order_Quantity` | Units sold |
| `MRP_Value` | Full retail price |
| `Order_Amount` | Actual billed amount |
 
### GRN Table
Tracks all stock movement — supplier → warehouse → stores → returns.
 
| Key Column | Description |
|---|---|
| `Facility` | Source of movement (e.g., `Garments`, `Warehouse`, store name) |
| `GRN Facility` | Destination → joins `Dim_Location` |
| `GRN_Date` | Receipt date → joins `Dim_Date` |
| `StyleCode` | Style → joins `Dim_Stylecode` |
| `GRN_Qty` | Quantity moved |
 
---
 
## 🔗 Relationships
 
| # | From Table | From Column | Cardinality | To Table | To Column | Status | Purpose |
|---|---|---|:---:|---|---|:---:|---|
| 1 | `Sales Table` | `Order_Date` | `*→1` | `Dim_Date` | `Date` | ✅ Active | Primary sales date filter |
| 2 | `Sales Table` | `Storename` | `*→1` | `Dim_Location` | `Location` | ✅ Active | Store location filter |
| 3 | `Sales Table` | `StyleCode` | `*→1` | `Dim_Stylecode` | `StyleCode` | ✅ Active | Style dimension for sales |
| 4 | `Sales Table` | `Size` | `*→1` | `Dim_Size` | `Size` | ✅ Active | Size filter for sales |
| 5 | `GRN Table` | `GRN_Date` | `*→1` | `Dim_Date` | `Date` | ✅ Active | GRN date filter |
| 6 | `GRN Table` | `GRN Facility` | `*→1` | `Dim_Location` | `Location` | ✅ Active | Destination location for GRN |
| 7 | `GRN Table` | `StyleCode` | `*→1` | `Dim_Stylecode` | `StyleCode` | ✅ Active | Style dimension for GRN |
| 8 | `GRN Table` | `Size` | `*→1` | `Dim_Size` | `Size` | ✅ Active | Size filter for GRN |
| 9 | `Style Sheet` | `StyleCode` | `1→1` | `Dim_Stylecode` | `StyleCode` | ✅ Active | Style master metadata linkage |
| 10 | `Dim_Stylecode` | `LaunchDate` | `*→1` | `Dim_Date` | `Date` | ⛔ Inactive | Launch date time intelligence |
 
> ⚠️ Relationship #10 (`LaunchDate → Dim_Date`) is **Inactive** to prevent filter ambiguity with the primary `Order_Date` path. Activate it inside measures using `USERELATIONSHIP('Dim_Stylecode'[LaunchDate], 'Dim_Date'[Date])` where launch-date filtering is needed.
 
### Model Diagram (Star Schema)
 
```
                        ┌─────────────┐
                        │  Dim_Date   │
                        │─────────────│
                        │ Date (PK)   │
                        │ MonthYear   │
                        │ Month&Year  │
                        │ Week No     │
                        └──────┬──────┘
               ┌───────────────┼────────────────────┐
               │ (Order_Date)  │ (GRN_Date)  (LaunchDate⛔)
               │               │                    │
  ┌────────────┴───┐     ┌─────┴──────────┐  ┌──────┴──────────┐
  │  Sales Table   │     │   GRN Table    │  │  Dim_Stylecode  │
  │────────────────│     │────────────────│  │─────────────────│
  │ OrderNumber    │     │ GRN            │  │ StyleCode (PK)  │
  │ Order_Date  ───┼─┐   │ GRN_Date    ───┼─┘│ Category        │
  │ Storename   ───┼─┼─┐ │ GRN Facility───┼──┼─LaunchDate      │
  │ StyleCode   ───┼─┼─┼─┼─StyleCode   ───┘  └─────────────────┘
  │ Size        ───┼─┼─┼─┼─Size            │       ↑ 1:1
  │ Order_Quantity │ │ │ │ GRN_Qty         │  ┌────┴────────────┐
  │ MRP_Value      │ │ │ │ Facility        │  │  Style Sheet    │
  │ Order_Amount   │ │ │ └─────────────────┘  │─────────────────│
  └────────────────┘ │ │                      │ StyleCode       │
                     │ │  ┌──────────────────┐│ LaunchDate      │
                     │ └─►│  Dim_Location    ││ Image URL       │
                     │    │──────────────────││ ...             │
                     │    │ Location (PK)    │└─────────────────┘
                     │    └──────────────────┘
                     │    ┌──────────────────┐
                     └───►│   Dim_Size       │
                          │──────────────────│
                          │ Size (PK)        │
                          └──────────────────┘
```
 
---
 
## 📐 Dimension Tables (DAX)
 
### Dim_Stylecode
```dax
Dim_Stylecode = 
DISTINCT(
    UNION(
        SELECTCOLUMNS('Sales Table', "StyleCode", 'Sales Table'[StyleCode], "Category", 'Sales Table'[Category]),
        SELECTCOLUMNS('GRN Table',   "StyleCode", 'GRN Table'[StyleCode],   "Category", 'GRN Table'[Category])
    )
)
```
 
### Dim_Date
```dax
Dim_Date = 
ADDCOLUMNS(
    CALENDAR(DATE(2023,01,01), DATE(2026,10,31)),
    "MonthYear",       FORMAT([Date], "YYYY-MM"),
    "Year",            YEAR([Date]),
    "Month",           MONTH([Date]),
    "Month&Year",      FORMAT([Date], "MMM-YY"),
    "Week No",         WEEKNUM([Date]),
    "WeekOfMonth",     INT((DAY([Date]) - 1) / 7) + 1,
    "Year-Month-Week", YEAR([Date]) & "-" & UPPER(FORMAT([Date],"MMM")) & "-" & "Week " & (INT((DAY([Date]) - 1) / 7) + 1)
)
```
 
### Dim_Location
```dax
Dim_Location = 
DISTINCT(
    UNION(
        SELECTCOLUMNS('Sales Table', "Location", 'Sales Table'[Storename]),
        SELECTCOLUMNS('GRN Table',   "Location", 'GRN Table'[GRN Facility])
    )
)
```
 
### Dim_Size
```dax
Dim_Size = 
DISTINCT(
    UNION(
        SELECTCOLUMNS('Sales Table', "Size", 'Sales Table'[Size]),
        SELECTCOLUMNS('GRN Table',   "Size", 'GRN Table'[Size])
    )
)
```
 
---
 
## 📏 DAX Measures
 
### Inventory Flow
 
| Measure | Formula |
|---|---|
| `Garments to Warehouse` | GRN_Qty where Facility=Garments & GRN Facility=Warehouse |
| `Warehouse_to_Store` | GRN_Qty where Facility=Warehouse & GRN Facility≠Warehouse |
| `Store_Inward_Qty` | GRN_Qty received at stores (excl. Warehouse & Garments source) |
| `Store_Return_to_Warehouse` | GRN_Qty returned from stores to Warehouse |
| `Total_Warehouse_IN` | Garments to Warehouse + Store_Return_to_Warehouse |
| `Warehouse_Remaining_Stock` | Total_Warehouse_IN − Warehouse_to_Store |
| `Store Stock` | Store_Inward_Qty − Sold_Qty − Store_Return_to_Warehouse |
 
**Pipeline:**
```
Garments Supplier
      ↓  [Garments to Warehouse]
  Warehouse
      ↓  [Warehouse_to_Store]        ↑  [Store_Return_to_Warehouse]
    Stores
      ↓  [Sold_Qty]
  Customers
```
 
---
 
### Sales
 
```dax
Sold_Qty             = SUM('Sales Table'[Order_Quantity])
Total Sale           = SUM('Sales Table'[Order_Amount])
Total Orders         = DISTINCTCOUNT('Sales Table'[OrderNumber])
Avg Bill Value       = DIVIDE([Total Sale], [Total Orders])
% Sold Quantity      = DIVIDE([Sold_Qty], [Warehouse_to_Store])
Last 30 Days Sale Qty = CALCULATE([Sold_Qty], FILTER('Sales Table',
                           'Sales Table'[Order_Date] >= TODAY()-30 &&
                           'Sales Table'[Order_Date] <= TODAY()))
```
 
---
 
### MRP vs Offer
 
```dax
-- Full-price units & revenue
MRP sold Qnty = CALCULATE(SUM([Order_Quantity]), [MRP_Value] = [Order_Amount])
MRP Sales     = CALCULATE(SUM([Order_Amount]),   [MRP_Value] = [Order_Amount])
MRP Sold %    = DIVIDE([MRP sold Qnty], [Sold_Qty], 0)
 
-- Discounted units & revenue
Offer Sold Qnty = CALCULATE(SUM([Order_Quantity]), [MRP_Value] <> [Order_Amount])
Offer Sales     = CALCULATE(SUM([Order_Amount]),   [MRP_Value] <> [Order_Amount])
Offer Sold %    = DIVIDE([Offer Sold Qnty], [Sold_Qty], 0)
```
 
---
 
### Style Performance & Time Intelligence
 
```dax
Days On Sale =
VAR LaunchDate   = MIN('Dim_Stylecode'[LaunchDate])
VAR HasStock     = NOT ISBLANK([Store_Inward_Qty])
RETURN IF(NOT HasStock || ISBLANK(LaunchDate), BLANK(),
    DATEDIFF(LaunchDate + 2, TODAY(), DAY))
-- (+2 day buffer: gap between system entry and store availability)
 
Days Since Launch =
VAR LaunchDate = MIN('Dim_Stylecode'[LaunchDate])
RETURN IF(ISBLANK(LaunchDate), BLANK(), DATEDIFF(LaunchDate, TODAY(), DAY))
 
Avg 30 Days Sale Qty =
VAR LaunchDate      = MIN('Dim_Stylecode'[LaunchDate])
VAR DaysSinceLaunch = DATEDIFF(LaunchDate, TODAY(), DAY)
VAR EffectiveDays   = IF(DaysSinceLaunch < 30, DaysSinceLaunch, 30)
VAR SalesQty        = [Last 30 Days Sale Qty]
RETURN IF(ISBLANK(EffectiveDays) || EffectiveDays <= 0, BLANK(),
    DIVIDE(SalesQty, EffectiveDays))
-- Uses min(DaysSinceLaunch, 30) so new styles aren't penalised
 
Style Count    = DISTINCTCOUNT('Dim_Stylecode'[StyleCode])
DayOnSaleText  = "Day on Sale: " & FORMAT([Days On Sale], "0")
```
 
---
 
## 📊 Dashboard
 
The main report page includes:
 
- **Style Performance Matrix** — StyleCode × GRN Qty, Sold Qty, Remaining Stock, Last 30 Days, Avg 30 Days, % Sold, MRP Sold %, Offer Sold %, Days On Sale
- **Top 10 Styles** — ranked by Last 30 Days Sale Qty with product image, LaunchDate, Days On Sale
- **Sales by Location** — store-level sold quantity ranking
- **GRN by Facility** — warehouse/facility inward quantity
 
**Slicers:** Location · Category · StyleCode (keyword) · LaunchDate · Days On Sale (range slider)
 
---
 
## ⚙️ Business Rules
 
| # | Rule |
|---|---|
| BR-01 | MRP sale = `Order_Amount == MRP_Value` exactly. Any difference (even rounding) is classified as Offer. |
| BR-02 | `Days On Sale` adds +2 days to LaunchDate to account for system entry → store availability lag. |
| BR-03 | `Avg 30 Days Sale Qty` caps the denominator at actual days since launch for styles < 30 days old. |
| BR-04 | `Days On Sale` returns BLANK for styles with no `Store_Inward_Qty` (no stock ever received). |
| BR-05 | `Store_Inward_Qty` includes store-to-store transfers; they cancel at aggregate level. |
| BR-06 | `Dim_Stylecode (LaunchDate) → Dim_Date` is Inactive — activate with `USERELATIONSHIP()` as needed. |
| BR-07 | `"Warehouse"` is a reserved string in location logic. Do not name any store "Warehouse". |
| BR-08 | `TODAY()` is used throughout — metrics update dynamically each day without manual refresh parameters. |
 
---
 
## 📋 Changelog
 
| Version | Date | Changes |
|---|---|---|
| 1.0 | Apr 2026 | Initial model — 2 fact tables, 4 dimensions, 23 DAX measures |
 
---
 
*Retail Fashion Analytics Platform — Internal Use Only*
