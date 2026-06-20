# Retail Sale & Stock Analytics — Power BI Data Model

A Power BI solution for tracking inventory movement, inventory health, stock availability, and sales performance across retail fashion stores in South India.

---

## 🗂️ Project Structure

```text
├── Sales Table          # Point-of-sale transactions (fact table)
├── GRN Table            # Goods received / stock movement (fact table)
├── Fact_Stock_Today     # Current stock snapshot by Store-Style-Size
├── Stock_Monthwise      # Historical month-end stock snapshots
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
| `Facility` | Source of movement |
| `GRN Facility` | Destination location |
| `GRN_Date` | Receipt date |
| `StyleCode` | Style |
| `Size` | Size |
| `GRN_Qty` | Quantity moved |

### Fact_Stock_Today

Current inventory snapshot table containing the latest stock position by Store, Style and Size.

| Key Column | Description |
|---|---|
| `FacilityName` | Store / Warehouse |
| `StyleCode` | Style |
| `Sku` | SKU |
| `EANCode` | EAN |
| `ItemCode` | Item Code |
| `Name` | Product Name |
| `Color` | Color |
| `Size` | Size |
| `CostPrice` | Cost Price |
| `OpeningStock` | Opening Quantity |
| `StockIn` | Stock In |
| `StockOut` | Stock Out |
| `ClosingQty` | Current Stock |
| `ClosingAmount` | Current Stock Value |

### Stock_Monthwise

Historical inventory snapshot table used for stock trend analysis.

| Key Column | Description |
|---|---|
| `DATE` | Snapshot Date |
| `FacilityName` | Store |
| `StyleCode` | Style |
| `ClosingQty` | Month-end Stock |
| `ClosingAmount` | Month-end Stock Value |

---

## 🔗 Relationships

| # | From Table | From Column | Cardinality | To Table | To Column | Status |
|---|---|---|:---:|---|---|:---:|
| 1 | Sales Table | Order_Date | *→1 | Dim_Date | Date | ✅ |
| 2 | Sales Table | Storename | *→1 | Dim_Location | Location | ✅ |
| 3 | Sales Table | StyleCode | *→1 | Dim_Stylecode | StyleCode | ✅ |
| 4 | Sales Table | Size | *→1 | Dim_Size | Size | ✅ |
| 5 | GRN Table | GRN_Date | *→1 | Dim_Date | Date | ✅ |
| 6 | GRN Table | GRN Facility | *→1 | Dim_Location | Location | ✅ |
| 7 | GRN Table | StyleCode | *→1 | Dim_Stylecode | StyleCode | ✅ |
| 8 | GRN Table | Size | *→1 | Dim_Size | Size | ✅ |
| 9 | Fact_Stock_Today | FacilityName | *→1 | Dim_Location | Location | ✅ |
| 10 | Fact_Stock_Today | StyleCode | *→1 | Dim_Stylecode | StyleCode | ✅ |
| 11 | Stock_Monthwise | DATE | *→1 | Dim_Date | Date | ✅ |
| 12 | Stock_Monthwise | FacilityName | *→1 | Dim_Location | Location | ✅ |
| 13 | Stock_Monthwise | StyleCode | *→1 | Dim_Stylecode | StyleCode | ✅ |
| 14 | Style Sheet | StyleCode | 1→1 | Dim_Stylecode | StyleCode | ✅ |
| 15 | Dim_Stylecode | LaunchDate | *→1 | Dim_Date | Date | ⛔ Inactive |

> Relationship #15 remains inactive to prevent ambiguity with Order_Date.

### Model Diagram (Star Schema)

```text
                           Dim_Date
                              ▲
          ┌──────────────┬────┼───────┬──────────────┐
          │              │    │       │              │
     Order_Date     GRN_Date DATE  LaunchDate⛔
          │              │    │       │
          ▼              ▼    ▼       ▼

     Sales Table    GRN Table    Stock_Monthwise
          │              │              │
          │              │              ├──── FacilityName
          │              │              └──── StyleCode
          │              │
          │              │
          ▼              ▼
       Dim_Size     Dim_Location
          ▲              ▲
          │              │
          │              │
     Fact_Stock_Today ───┘
          │
          └──── StyleCode ─────► Dim_Stylecode ◄──── Style Sheet
```

---

## 📐 Dimension Tables (DAX)

### Dim_Stylecode

```dax
Dim_Stylecode =
DISTINCT(
    UNION(
        SELECTCOLUMNS('Sales Table',"StyleCode",'Sales Table'[StyleCode],"Category",'Sales Table'[Category]),
        SELECTCOLUMNS('GRN Table',"StyleCode",'GRN Table'[StyleCode],"Category",'GRN Table'[Category])
    )
)
```

### Dim_Date

```dax
Dim_Date =
ADDCOLUMNS(
    CALENDAR(DATE(2023,01,01), DATE(2026,10,31)),
    "MonthYear", FORMAT([Date], "YYYY-MM"),
    "Year", YEAR([Date]),
    "Month", MONTH([Date])
)
```

### Dim_Location

```dax
Dim_Location =
DISTINCT(
    UNION(
        SELECTCOLUMNS('Sales Table',"Location",'Sales Table'[Storename]),
        SELECTCOLUMNS('GRN Table',"Location",'GRN Table'[GRN Facility])
    )
)
```

### Dim_Size

```dax
Dim_Size =
DISTINCT(
    UNION(
        SELECTCOLUMNS('Sales Table',"Size",'Sales Table'[Size]),
        SELECTCOLUMNS('GRN Table',"Size",'GRN Table'[Size])
    )
)
```

---

## 📏 DAX Measures

### Inventory Flow

- Garments to Warehouse
- Warehouse_to_Store
- Store_Inward_Qty
- Store_Return_to_Warehouse
- Total_Warehouse_IN
- Warehouse_Remaining_Stock
- Store Stock

### 📦 Current Stock Snapshot

| Measure |
|---|
| Total Qty |
| Total Value |
| Total Styles |
| Full Size Style Count |
| Full Size Qty |
| Broken Style Count |
| Broken Qty |

### Sales

```dax
Sold_Qty = SUM('Sales Table'[Order_Quantity])
Total Sale = SUM('Sales Table'[Order_Amount])
Total Orders = DISTINCTCOUNT('Sales Table'[OrderNumber])
Avg Bill Value = DIVIDE([Total Sale],[Total Orders])
```

### Style Performance & Time Intelligence

- Days On Sale
- Days Since Launch
- Avg 30 Days Sale Qty
- Last 30 Days Sale Qty
- Style Count

---

## 👕 Size Availability Logic

Full Size Style:

- S available
- M available
- L available
- XL available
- XXL available
- ClosingQty > 0 in all required sizes

Broken Style:

- One or more required sizes missing

Metrics:

- Full Size Style Count
- Full Size Qty
- Broken Style Count
- Broken Qty

---

## 📊 Dashboard

- Style Performance Matrix
- Top 10 Styles
- Sales by Location
- GRN by Facility
- Inventory Health Dashboard
- Size Availability Dashboard
- Historical Inventory Dashboard

**Slicers:** Location · Category · StyleCode · LaunchDate · Days On Sale

---

## ⚙️ Business Rules

| # | Rule |
|---|---|
| BR-01 | MRP sale = Order_Amount == MRP_Value exactly. |
| BR-02 | Days On Sale adds +2 days to LaunchDate. |
| BR-03 | Avg 30 Days Sale Qty uses actual days for styles < 30 days old. |
| BR-04 | Days On Sale returns BLANK if no stock was ever received. |
| BR-05 | Store transfers cancel at aggregate level. |
| BR-06 | LaunchDate relationship is inactive and should be activated with USERELATIONSHIP(). |
| BR-07 | Warehouse is a reserved location name. |
| BR-08 | TODAY() is used throughout. |
| BR-09 | Fact_Stock_Today represents the latest inventory snapshot. |
| BR-10 | Warehouse inventory is excluded from retail stock KPIs. |
| BR-11 | Full Size requires S, M, L, XL and XXL with stock > 0. |
| BR-12 | Missing any required size classifies a style as Broken Size. |
| BR-13 | Full Size calculations exclude Warehouse inventory. |
| BR-14 | Stock_Monthwise is only for historical trend analysis. |

---

## 📊 KPI Dictionary

| KPI | Definition |
|---|---|
| Sold Qty | Total units sold |
| Store Stock | Current stock available in stores |
| Total Qty | Total inventory excluding Warehouse |
| Total Value | Total stock value excluding Warehouse |
| Full Size Style Count | Styles having complete size availability |
| Broken Style Count | Styles missing one or more required sizes |
| Full Size Qty | Stock quantity of full-size styles |
| Broken Qty | Stock quantity of broken styles |
| Days On Sale | Days since launch date |
| Sell Through % | Sold Qty ÷ Warehouse_to_Store |

---

## 📋 Changelog

| Version | Date | Changes |
|---|---|---|
| 1.0 | Apr 2026 | Initial model |
| 1.1 | Jun 2026 | Launch analytics |
| 1.2 | Jun 2026 | Fact_Stock_Today added |
| 1.3 | Jun 2026 | Stock_Monthwise added |
| 1.4 | Jun 2026 | Full Size / Broken Size analysis |
| 1.5 | Jun 2026 | Documentation synchronized with current Power BI model |
