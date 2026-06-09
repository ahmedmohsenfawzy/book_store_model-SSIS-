# 📚 Book Store — End-to-End Data Warehouse (SSIS + SQL Server + Power BI)

A complete **data warehousing and business intelligence project** for a book retail store. Raw transactional data is extracted from an OLTP SQL Server database, transformed and loaded into a star-schema data warehouse using **SSIS (SQL Server Integration Services)**, and visualised in a **Power BI** dashboard that gives the business instant insight into sales, customers, shipping, and inventory.

---

## 🏢 Business Context

The book store operates globally — selling books in dozens of languages, shipping to customers across multiple countries via Express, Standard, International, and Priority methods. The operational database records every customer, order, order line, shipping detail, and fulfilment status update in real time. But that OLTP schema is designed for writes, not reads: analysts can't easily answer questions like:

- What were total sales per year, and which year performed best?
- Which shipping method accounts for the most orders — and is it the cheapest one?
- Which book languages are most popular, and are foreign-language sales growing?
- How many orders are stuck in "Pending Delivery" vs successfully delivered?
- What is the geographical spread of customers, and which countries drive the most volume?
- What is the average revenue per customer (customer lifetime value)?

This project builds the data pipeline and reporting layer that answers all of those questions with a single Power BI dashboard refresh.

---

## 📊 Dashboard Preview

![Power BI Dashboard](https://raw.githubusercontent.com/ahmedmohsenfawzy/book_store_model-SSIS-/master/Screenshot%202026-04-06%20000838.png)

**Key metrics surfaced by the dashboard:**

| Metric | Value |
|--------|-------|
| Total Customers | 2K |
| Total Orders | 7,544 |
| Customer Lifetime Value | $154.18K |
| Average Sales per Customer | $88.46 |
| Total Sales | $154K |

**Visuals included:**
- **Total Sales by Year** (line chart) — tracks revenue growth from 2020 to 2023, peaking around 2022
- **Books Sold by Shipping Method** (pie chart) — Express 46%, Standard 40%, International 13%, Priority 1%
- **Orders by Language** (area chart) — English dominates, but Spanish, British, French, German are all tracked
- **Orders by Status** (bar chart) — Delivered, Pending Delivery, Delivery In Progress, Order Received, Cancelled, Returned
- **Orders by Country** (bubble map) — global customer spread visualised on an Azure map

---

## 🗄️ Source Database — OLTP Schema

The raw operational database is a normalised relational model designed for transactional integrity:

![Source OLTP Schema](https://raw.githubusercontent.com/ahmedmohsenfawzy/book_store_model-SSIS-/master/Screenshot%202026-02-21%20153412.png)

### Source Tables

| Table | Description |
|-------|-------------|
| `customer` | Customer records: `customer_id`, `first_name`, `last_name`, `email` |
| `cust_order` | One row per order: links customer, shipping method, destination address, and order date |
| `order_line` | One row per book per order: `book_id`, `order_id`, `price` |
| `order_history` | Audit log of every status change per order with a timestamp |
| `order_status` | Lookup: status codes → status labels (Delivered, Cancelled, etc.) |
| `address` | Full address with `street_number`, `street_name`, `city`, `country_id` |
| `address_status` | Whether an address is current or historical for a customer |
| `customer_address` | Bridge table linking customers to their one or more addresses |
| `country` | Lookup: `country_id` → `country_name` |
| `shipping_method` | Lookup: `method_id` → `method_name`, `cost` |

The `book` table (visible in the DWH schema) holds the product catalogue: title, ISBN, language, number of pages, publication date, publisher.

---

## 🏗️ Data Warehouse — Star Schema

The DWH flattens and denormalises the OLTP model into a classic star schema optimised for analytical queries:

![Data Warehouse Star Schema](https://raw.githubusercontent.com/ahmedmohsenfawzy/book_store_model-SSIS-/master/Screenshot%202026-06-09%112848.png)

### Fact Table

**`fact_sales`** — the central grain is one row per order line per status event:

| Column | Description |
|--------|-------------|
| `fact_key` | Surrogate PK |
| `date_key` | FK → `dim_date` |
| `order_id` | Original order ID from OLTP |
| `customer_key` | FK → `dim_customer` |
| `shipping_method_key` | FK → `fact_shipping_method` |
| `address_key` | FK → `dim_addres` |
| `status_key` | FK → `dim_order_status` |
| `book_key` | FK → `dim_book` |
| `price` | Line item price |
| `status_date` | Date of this status event |
| `order_date` | When the order was originally placed |

### Dimension Tables

| Dimension | Surrogate Key | Key Attributes |
|-----------|--------------|----------------|
| `dim_customer` | `customer_key` | `customer_id`, `first_name`, `last_name`, `email` |
| `dim_book` | `book_key` | `book_id`, `title`, `isbn13`, `language_id`, `num_pages`, `publication_date`, `publisher_id`, `language_code`, `language_name`, `publisher_name` |
| `dim_date` | `date_key` | `order_date`, `status_date`, `year` |
| `dim_addres` | `address_key` | `address_id`, `street_number`, `street_name`, `city`, `country_name`, `address_status` |
| `dim_order_status` | `status_key` | `status_id`, `status_value` (Delivered, Pending, etc.) |
| `fact_shipping_method` | `shipping_method_key` | `method_id`, `method_name`, `cost` |

> Note: `book_author` and `author` tables are preserved as lookup helpers to support author-level analysis.

---

## 🔄 ETL Pipeline — SSIS

The SSIS project (`Integration Services Project2`) contains one package per dimension and one for the fact table. Each package follows the same pattern:

```
[SQL Server OLTP Source]
        ↓
[OLE DB Source — SELECT from raw tables]
        ↓
[Data Transformations]
   • Derived Column   — compute surrogate key expressions, derive year from date
   • Lookup           — resolve foreign keys to their DWH surrogate keys
   • Data Conversion  — cast varchar → nvarchar, string prices → decimal
   • Conditional Split — route new vs. existing records (upsert logic)
        ↓
[OLE DB Destination — INSERT into DWH dimension/fact table]
```

### Package Execution Order

```
1. dim_date           ← no dependencies
2. dim_customer       ← no dependencies
3. dim_addres         ← no dependencies
4. dim_order_status   ← no dependencies
5. dim_book           ← depends on author/language/publisher lookups
6. fact_shipping_method
7. fact_sales         ← depends on ALL dimensions above
```

Dimensions are always loaded before facts. Running packages out of order will cause lookup failures.

---

## 📁 Project Structure

```
book_store_model-SSIS-/
├── Integration Services Project2/   # SSIS project (Visual Studio)
│   ├── *.dtsx                       # One DTSX package per dimension + fact
│   ├── Project.params               # Project-level parameters (connection strings)
│   └── *.conmgr                     # Connection managers (OLTP + DWH)
├── full-project.pbix                # Power BI report connected to DWH
├── Screenshot 2026-02-21 153412.png # OLTP ERD
├── Screenshot 2026-02-23 214658.png # DWH star schema
└── Screenshot 2026-04-06 000838.png # Power BI dashboard
```

---

## ⚙️ Setup & Requirements

### Prerequisites

- SQL Server 2019+ (or SQL Server Developer Edition — free)
- SQL Server Data Tools (SSDT) with SSIS extension for Visual Studio
- Power BI Desktop (free)

### Steps to Run

**1. Restore / create the source OLTP database**

Create the source schema using the ERD above. Load sample data or restore a backup into a database named `bookstore_oltp` (or update the connection managers accordingly).

**2. Create the DWH database**

Create an empty database named `bookstore_dwh`. Run the DDL scripts to create the star schema tables (all dimension and fact tables as shown in the DWH schema above).

**3. Configure SSIS connections**

Open the `.sln` in Visual Studio with SSDT. In the Connection Managers panel, update:
- `OLTP_Connection` → point to your `bookstore_oltp` database
- `DWH_Connection` → point to your `bookstore_dwh` database

**4. Execute SSIS packages in order**

Run the packages in the order listed above (dimensions first, fact last). You can either execute them individually from SSDT or deploy the project to the SSIS Catalog and run via SQL Server Agent.

**5. Open Power BI**

Open `full-project.pbix` in Power BI Desktop. Update the data source credentials to point to your `bookstore_dwh`. Click **Refresh** to pull the latest data.

---

## 🛠️ Tech Stack

| Tool | Purpose |
|------|---------|
| SQL Server | OLTP source + DWH destination |
| SSIS (SQL Server Integration Services) | ETL — extract, transform, load |
| Visual Studio + SSDT | SSIS package development |
| Power BI Desktop | Dashboard and visualisation |
| Microsoft Azure Maps | Geo visualisation in Power BI |

---

## 📄 License

Educational / portfolio project. Book store data is synthetic sample data.
