# Data Dictionary

Galaxy schema (fact constellation): 6 dimensions shared across 6 fact tables, plus a row-level security table and a measures-only table. Naming conventions are documented in [`schema_conventions.md`](schema_conventions.md).

---

## Dimensions

### dim_customer
One row per customer.

| Column | Description | Source |
|---|---|---|
| `customer_id` | Customer identifier (source ID, used directly as the key — no surrogate needed) | `CUST_MASTER.CustomerID` |
| `customer_name` | Customer/company name | `CUST_MASTER.CustomerName` |
| `segment` | Customer segment (e.g. Mid-Market, Enterprise) | `CUST_MASTER.Segment` |
| `account_manager` | Assigned account manager | `CUST_MASTER.AccountManager` |
| `payment_terms` | Payment terms (e.g. Net 30) | `CUST_MASTER.PaymentTerms` |
| `street` | Street address | `Address.Street` (joined via `CUST_MASTER.AddressID`) |
| `city` | City | `Address.CityName` (joined via `AddressID`) |
| `region` | Region | `cities.RegionName` (joined via `CityName`) |
| `contact_name` | Primary contact name | `customer_contacts.ContactName` (where `IsPrimary`) |
| `contact_email` | Primary contact email | `customer_contacts.Email` (where `IsPrimary`) |
| `phone` | Phone number | `user_details.Phone` (joined via `UserID` = `CustomerID`) |
| `credit_limit` | Approved credit limit | `user_details.CreditLimit` (joined via `UserID` = `CustomerID`) |

Dropped from source: `CUST_MASTER.hash_key`, `CUST_MASTER.source_id` — technical/lineage columns, not needed downstream.

### dim_product
One row per product.

| Column | Description | Source |
|---|---|---|
| `product_key` | Surrogate key | generated |
| `product_code` | Product code | `products.ProductCode` |
| `product_name` | Product name | `products.ProductName` |
| `brand` | Brand | `products.Brand` |
| `category` | Product category | `subcategories.CategorySubcategory`, split on `\|` (text before) |
| `subcategory` | Product subcategory | `subcategories.CategorySubcategory`, split on `\|` (text after), matched to `products.SubcategoryName` |
| `price` | List/unit price | `products.UnitPrice` |
| `supplier` | Primary supplier | `products.PrimarySupplier` |

Dropped from source: `products.ProductDescription`, `hash_key`, `source_id`.

### dim_date
One row per calendar date.

| Column | Description | Source |
|---|---|---|
| `date` | Calendar date | generated date table |
| `month` | Month | derived from `date` |
| `year` | Year | derived from `date` |

### dim_campaign
One row per marketing campaign.

| Column | Description | Source |
|---|---|---|
| `campaign_key` | Surrogate key | generated |
| `campaign_name` | Campaign name | `CAMPAIGN_LOG.CampaignName` (deduplicated — source repeats one row per campaign-day) |
| `channel` | Marketing channel | `CAMPAIGN_LOG.Channel` |
| `budget` | Total campaign budget | `CAMPAIGN_LOG.Budget` (deduplicated) |
| `start_date` | Campaign start date | `CAMPAIGN_LOG.StartDate` |
| `end_date` | Campaign end date | `CAMPAIGN_LOG.EndDate` |

Daily performance metrics from the same source sheet (`Date`, `Impressions`, `Clicks`, `Spend`) go to `fact_campaign_spend` instead, since they're a different grain (daily, not per-campaign).

### dim_geo
One row per city.

| Column | Description | Source |
|---|---|---|
| `geo_key` | Surrogate key | generated |
| `city` | City name | `cities.CityName` / `Address.CityName` |
| `region` | Region | `cities.RegionName` |

Used twice from `fact_sales` via role-playing relationships (`bill_to_city_key`, `ship_to_city_key`) to model billing vs. shipping city separately.

### dim_order_flags
One row per distinct (channel, status, priority) combination.

| Column | Description | Source |
|---|---|---|
| `flag_key` | Surrogate key | generated |
| `channel` | Order channel | `ORDERS_2025` / `ORDERS_2026`.`OrderChannel` |
| `channel_code` | Encoded/abbreviated channel | derived from `channel` in Power Query |
| `status` | Order status | `ORDERS_2025` / `ORDERS_2026`.`Status` |
| `priority` | Order priority | `ORDERS_2025` / `ORDERS_2026`.`Priority` |

---

## Facts

### fact_sales
Grain: **one row per order line item.**

| Column | Description | Source |
|---|---|---|
| `line_id` | Line item ID | `order_line_items.LineID` |
| `order_id` | Order ID | `order_line_items.OrderID` |
| `order_date` | Order date | `ORDERS_2025`/`2026.OrderDate` (via `OrderID`) |
| `customer_id` | Customer FK → `dim_customer` | resolved via `CustomerName` → `CUST_MASTER.CustomerID` |
| `product_key` | Product FK → `dim_product` | resolved via `ProductName` → `dim_product` |
| `flag_key` | Order flags FK → `dim_order_flags` | resolved via (channel, status, priority) |
| `bill_to_city_key` | Billing city FK → `dim_geo` | `ORDERS_2025`/`2026.BillToCity` |
| `ship_to_city_key` | Shipping city FK → `dim_geo` | `ORDERS_2025`/`2026.ShipToCity` |
| `quantity` | Quantity sold | `order_line_items.Quantity` |
| `price` | Unit price | `order_line_items.UnitPrice` |
| `cost` | Unit cost | `order_line_items.UnitCost` |
| `discount` | Discount percentage | `order_line_items.DiscountPct` |
| `line_total` | Line revenue total | `order_line_items.LineTotal` |

### fact_order_process
Grain: **one row per order** (order lifecycle / fulfillment tracking).

| Column | Description | Source |
|---|---|---|
| `order_id` | Order ID | `ORDERS_2025` + `ORDERS_2026` (appended) |
| `customer_id` | Customer FK → `dim_customer` | resolved via `CustomerName` |
| `order_date` | Order placed date | `ORDERS_2025`/`2026.OrderDate` |
| `invoice_date` | Invoice issued date | `INVOICES.InvoiceDate` (via `OrderID`) |
| `pay_date` | Payment received date | `payments.PayDate` (via `InvoiceID`) |
| `ship_date` | Shipping date | `shipments.ShipDate` (via `OrderID`) |
| `delivery_date` | Delivery date | `shipments.DeliveryDate` (via `OrderID`) |
| `order_to_pay` | Calculated column: days elapsed from `order_date` to `pay_date` | calculated in the model |

`shipments` and `Sheet1` are identical (`Sheet1` is a duplicate) — `Sheet1` was discarded.

### fact_inventory
Grain: **one row per product per month.**

| Column | Description | Source |
|---|---|---|
| `product_key` | Product FK → `dim_product` | `inventory.ProductName` → `dim_product` |
| `month` | Month | `inventory` column headers (`2025-01` … `2025-12`), unpivoted |
| `untis` ⚠️ | Units in stock — **likely a typo for `units`**, flagged for review | `inventory` monthly columns, unpivoted |

### fact_campaign_spend
Grain: **one row per campaign per day.**

| Column | Description | Source |
|---|---|---|
| `campaign_key` | Campaign FK → `dim_campaign` | `CAMPAIGN_LOG.CampaignName` |
| `date` | Date | `CAMPAIGN_LOG.Date` |
| `impressions` | Daily impressions | `CAMPAIGN_LOG.Impressions` |
| `clicks` | Daily clicks | `CAMPAIGN_LOG.Clicks` |
| `spend` | Daily spend | `CAMPAIGN_LOG.Spend` |

### fact_promotion_coverage
Grain: **one row per (campaign, product) promoted.** Bridge table.

| Column | Description | Source |
|---|---|---|
| `campaign_key` | Campaign FK → `dim_campaign` | `campaign_skus.CampaignName` |
| `product_key` | Product FK → `dim_product` | `campaign_skus.PromotedSKUs`, split and matched to `dim_product` |

### fact_sales_targets
Grain: **one row per period.**

| Column | Description | Source |
|---|---|---|
| `date` | Target period | `sales_targets.Period` |
| `target_revenue` | Revenue target | `sales_targets.TargetRevenue` |

---

## Other tables

### security
Row-level security table — restricts each user to their assigned region.

| Column | Description | Source |
|---|---|---|
| `user_email` | User's login email | `security.UserEmail` |
| `region` | Region the user is allowed to see | `security.Region` |

### _measures
Empty table holding only DAX measures (no data columns). Descriptions below are inferred from the measure names — the `.pbix` stores DAX in a compressed binary format that couldn't be extracted for this dictionary, so exact formulas aren't listed.

| Measure | Inferred purpose |
|---|---|
| `total_sales` | Sum of `fact_sales.line_total` |
| `total_orders` | Distinct count of orders |
| `total_active_customers` | Distinct count of customers with at least one sale |
| `base_total_customers` | Baseline/total customer count (all of `dim_customer`, regardless of activity) |
| `avg_order_to_pay` | Average of `fact_order_process.order_to_pay` |
| `Column` ⚠️ | Unnamed/default measure — looks like a leftover, worth reviewing or deleting |

---

## Source sheets not used in the final model

| Sheet | Reason |
|---|---|
| `exchange_rates` | No table in the final model has a currency or rate column, and no amount column carries a currency — unused in the model. |
| `dim_order` | Plain list of order IDs; not published as its own table — looks like a staging/anchor query used only inside Power Query. |
| `regions` | Plain list of region names; merged into `cities` / `dim_geo` rather than published on its own. |
| `Sheet1` | Exact duplicate of `shipments` — discarded. |
