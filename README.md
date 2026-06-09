# 📚 Book Store — End-to-End Data Warehouse (SSIS + SQL Server + Power BI)

A complete **data warehousing and business intelligence project** for a global book retail store. Raw transactional data is extracted from a normalised OLTP SQL Server database, transformed and loaded into a star-schema data warehouse using **SSIS (SQL Server Integration Services)**, and visualised in a **Power BI** dashboard that gives the business instant insight into sales, customers, shipping, and inventory.

---

## 🏢 Business Context

The book store operates globally — selling books in dozens of languages, shipping to customers across multiple countries via Express, Standard, International, and Priority methods. The operational database records every customer, order, order line, shipping detail, and fulfilment status update in real time.

But that OLTP schema is optimised for writes, not reads. Analysts cannot easily answer questions like:

- What were total sales per year, and which year performed best?
- Which shipping method carries the most volume — and is it the most cost-effective?
- Which book languages are most popular, and are foreign-language sales growing?
- Which authors generate the most sales revenue?
- How many orders are stuck in "Pending Delivery" vs successfully delivered?
- What is the geographical spread of customers by country?
- What is the average revenue per customer (customer lifetime value)?

This project builds the full pipeline — from raw OLTP tables to a polished Power BI dashboard — that answers all of those questions on demand.

---

## 📊 Dashboard Preview

![Power BI Dashboard](https://raw.githubusercontent.com/ahmedmohsenfawzy/book_store_model-SSIS-/master/Screenshot%202026-04-06%20000838.png)

**Key KPIs surfaced by the dashboard:**

| Metric | Value |
|--------|-------|
| Total Customers | 2K |
| Total Orders | 7,544 |
| Customer Lifetime Value | $154.18K |
| Average Sales per Customer | $88.46 |
| Total Sales | $154K |

**Visuals included:**
- **Total Sales by Year** — line chart tracking revenue from 2020 to 2023, peaking around 2022
- **Books Sold by Shipping Method** — pie chart: Express 46%, Standard 40%, International 13%, Priority 1%
- **Orders by Language** — area chart showing English dominates, with Spanish, British, French, German tracked
- **Orders by Status** — bar chart: Delivered, Pending Delivery, Delivery In Progress, Order Received, Cancelled, Returned
- **Orders by Country** — bubble map on Azure Maps showing the global customer footprint

---

## 🗄️ Source Database — OLTP Schema

The raw operational database is a fully normalised relational model built for transactional integrity across the entire order lifecycle, including the book catalogue with authors, languages, and publishers.

![Source OLTP Schema](https://raw.githubusercontent.com/ahmedmohsenfawzy/book_store_model-SSIS-/master/Screenshot%202026-06-09%20112848.png)

### Source Tables

**Customer & Order domain:**

| Table | Description |
|-------|-------------|
| `customer` | Customer records: `customer_id`, `first_name`, `last_name`, `email` |
| `cust_order` | One row per order — links to customer, shipping method, destination address, and `order_date` |
| `order_line` | One row per book per order: `line_id`, `order_id`, `book_id`, `price` |
| `order_history` | Audit log of every status change per order: `history_id`, `order_id`, `status_id`, `status_date` |
| `order_status` | Lookup: `status_id` → `status_value` (Delivered, Pending Delivery, Cancelled, etc.) |
| `shipping_method` | Lookup: `method_id` → `method_name`, `cost` |

**Address domain:**

| Table | Description |
|-------|-------------|
| `address` | Full address: `address_id`, `street_number`, `street_name`, `city`, `country_id` |
| `address_status` | Whether an address is current or historical: `status_id`, `address_status` |
| `customer_address` | Bridge table linking customers to their one or more addresses: `address_id`, `status_id`, `customer_id` |
| `country` | Lookup: `country_id` → `country_name` |

**Book catalogue domain:**

| Table | Description |
|-------|-------------|
| `book` | Product catalogue: `book_id`, `title`, `isbn13`, `language_id`, `num_pages`, `publication_date`, `publisher_id` |
| `book_language` | Lookup: `language_id` → `language_code`, `language_name` |
| `publisher` | Lookup: `publisher_id` → `publisher_name` |
| `author` | Author reference: `author_id`, `author_name` |
| `book_author` | Bridge table resolving the many-to-many between books and authors: `author_id`, `book_id` |

---

## 🏗️ Data Warehouse — Star Schema

The DWH flattens the normalised OLTP model into a star schema optimised for analytical queries. All foreign keys from the OLTP schema are resolved to surrogate keys, and reference tables are denormalised into their parent dimensions.

![Data Warehouse Star Schema](https://raw.githubusercontent.com/ahmedmohsenfawzy/book_store_model-SSIS-/master/Screenshot%202026-02-23%20214658.png)

### Fact Table

**`fact_sales`** — the central grain is one row per order line per status event:

| Column | Source | Description |
|--------|--------|-------------|
| `fact_key` | Generated | Surrogate PK |
| `date_key` | `order_history.status_date` | FK → `dim_date` |
| `order_id` | `cust_order.order_id` | Original order reference |
| `customer_key` | `cust_order.customer_id` | FK → `dim_customer` |
| `shipping_method_key` | `cust_order.shipping_method_id` | FK → `fact_shipping_method` |
| `address_key` | `cust_order.dest_address_id` | FK → `dim_addres` |
| `status_key` | `order_history.status_id` | FK → `dim_order_status` |
| `book_key` | `order_line.book_id` | FK → `dim_book` |
| `price` | `order_line.price` | Line item price |
| `status_date` | `order_history.status_date` | Date of this status update |
| `order_date` | `cust_order.order_date` | Original order placement date |

### Dimension Tables

| Dimension | Surrogate Key | Source Tables | Key Attributes |
|-----------|--------------|---------------|----------------|
| `dim_customer` | `customer_key` | `customer` | `customer_id`, `first_name`, `last_name`, `email` |
| `dim_book` | `book_key` | `book`, `book_language`, `publisher` | `book_id`, `title`, `isbn13`, `language_id`, `num_pages`, `publication_date`, `publisher_id`, `language_code`, `language_name`, `publisher_name` |
| `dim_date` | `date_key` | `order_history`, `cust_order` | `order_date`, `status_date`, `year` |
| `dim_addres` | `address_key` | `address`, `country`, `address_status` | `address_id`, `street_number`, `street_name`, `city`, `country_name`, `address_status` |
| `dim_order_status` | `status_key` | `order_status` | `status_id`, `status_value` |
| `fact_shipping_method` | `shipping_method_key` | `shipping_method` | `method_id`, `method_name`, `cost` |

> `author` and `book_author` from the OLTP are resolved during the ETL via the `book_author_pridge` bridge task and baked into `dim_book` for query simplicity.

---

## 🔄 ETL Pipeline — SSIS

### Control Flow Overview

![SSIS Control Flow](https://raw.githubusercontent.com/ahmedmohsenfawzy/book_store_model-SSIS-/master/Screenshot%202026-04-19%20092038.png)

The SSIS project uses a **Sequence Container** to enforce execution order. All dimension packages run in parallel where possible, and `fact_sales` only executes after every dimension is populated. A dedicated **Data Flow Task** handles the `book_author_pridge` bridge resolution before loading `fact_sales`.

### Execution Order

```
Sequence Container
│
├── dim_customer          (no dependencies)
├── dim_book              (needs book_language + publisher lookups)
├── dim_author            (feeds book_author_pridge)
├── address               (needs country + address_status)
├── dim_shipping_method   (no dependencies)
└── dim_order_status      (no dependencies)
         │
         ▼
   [Data Flow Task]
   book_author_pridge     ← resolves book ↔ author bridge
         │
         ▼
   fact_sales             ← loaded last, after ALL dimensions complete
```

### Data Flow Pattern (per package)

Each SSIS package follows this standard pattern:

```
[OLE DB Source]          ← SELECT from OLTP table(s)
       ↓
[Data Conversion]        ← cast types: varchar→nvarchar, string prices→decimal
       ↓
[Derived Column]         ← generate surrogate keys, derive year from date fields
       ↓
[Lookup]                 ← resolve FK values to DWH surrogate keys
       ↓
[Conditional Split]      ← route new records vs. existing (upsert logic)
       ↓
[OLE DB Destination]     ← INSERT into DWH dimension or fact table
```

---

## 📁 Project Structure

```
book_store_model-SSIS-/
├── Integration Services Project2/     # SSIS Visual Studio project
│   ├── *.dtsx                         # One package per dimension + fact
│   ├── Project.params                 # Project-level parameters
│   └── *.conmgr                       # Connection managers (OLTP + DWH)
├── full-project.pbix                  # Power BI report
├── Screenshot 2026-06-09 112848.png   # OLTP source ERD (updated)
├── Screenshot 2026-02-23 214658.png   # DWH star schema
├── Screenshot 2026-04-19 092038.png   # SSIS control flow
└── Screenshot 2026-04-06 000838.png   # Power BI dashboard
```

---

## ⚙️ Setup & Requirements

### Prerequisites

- SQL Server 2019+ (Developer Edition is free)
- SQL Server Data Tools (SSDT) with SSIS extension for Visual Studio 2019/2022
- Power BI Desktop (free from Microsoft)

### Steps to Run

**1. Create and populate the OLTP source database**

Create a database named `gravity_books` (visible in the SSIS taskbar). Create all source tables using the ERD above and load with sample data.

**2. Create the DWH database**

Create an empty database (e.g. `bookstore_dwh`) and run the DDL to create all dimension and fact tables matching the star schema above.

**3. Configure SSIS connection managers**

Open the `.sln` in Visual Studio. In the Connection Managers panel update:
- `OLTP_Connection` → your `gravity_books` database
- `DWH_Connection` → your `bookstore_dwh` database

**4. Run the SSIS packages in order**

Execute via SSDT or deploy to the SSIS Catalog and schedule via SQL Server Agent. Always run in the order shown in the control flow: dimensions first, `book_author_pridge` next, `fact_sales` last.

**5. Open Power BI**

Open `full-project.pbix` in Power BI Desktop. Update the data source to point to your DWH database. Click **Refresh** to load the latest data.

---

## 🛠️ Tech Stack

| Tool | Purpose |
|------|---------|
| SQL Server | OLTP source + DWH destination |
| SSIS (SQL Server Integration Services) | ETL — extract, transform, load |
| Visual Studio + SSDT | SSIS package development environment |
| Power BI Desktop | Dashboard and reporting |
| Microsoft Azure Maps | Geo visualisation in Power BI |

---

## 📄 License

Educational / portfolio project. Book store data is synthetic sample data based on the Gravity Books dataset.
