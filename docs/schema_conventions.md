# Schema Conventions

Naming standards applied consistently across the model, set up front before building any table.

## Naming

- **Tables and columns**: `snake_case`
- **Table prefixes**: `dim_` for dimensions, `fact_` for fact tables
- **Key suffix**: `_key` = surrogate key generated during modeling · `_id` = identifier that comes as-is from the source system (no surrogate needed when the source ID is already stable and unique — see `dim_customer.customer_id`)
- **Displayed values** (labels shown to end users in visuals): `Capitalize Each Word`

## Schema type

This is a **galaxy schema (fact constellation)**, not a simple star schema: six fact tables share a common set of conformed dimensions, rather than each fact having its own private dimensions.

**Dimensions**: `dim_customer` · `dim_product` · `dim_date` · `dim_campaign` · `dim_geo` · `dim_order_flags`
**Facts**: `fact_sales` · `fact_order_process` · `fact_inventory` · `fact_campaign_spend` · `fact_promotion_coverage` · `fact_sales_targets`
**Other**: `security` (row-level security by region — `region`, `user_email`) · `_measures` (DAX measures table, no data columns)

`dim_geo` is used twice from `fact_sales` (`bill_to_city_key`, `ship_to_city_key`) as a role-playing dimension — billing city and shipping city are modeled separately without duplicating the dimension.

See [`data_dictionary.md`](data_dictionary.md) for the full column-by-column breakdown of every table.
