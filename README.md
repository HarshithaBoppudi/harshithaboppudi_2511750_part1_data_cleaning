# Business Data Cleaning, Validation & Excel Reporting

## 1. Problem Summary

A retail company exported order-level sales data from multiple internal systems. The export had inconsistent text formatting, mixed date formats, duplicate records, missing values, invalid discounts, sales/profit calculation mismatches, and inconsistent order statuses. This project cleans and validates the dataset, documents every issue found and every cleaning decision made, and produces summary reports an analyst could use for a business review.

## 2. Dataset Description

- **Source file**: `data/raw_orders.xlsx` (932 order-level records, 21 columns)
- **Key fields**: order identifiers, order/ship dates, customer demographics (segment, region, state, city), product hierarchy (category, sub-category, product name), shipping mode, quantity, unit price, discount, sales, cost, profit, payment status, and order status.
- **Cleaned output**: `data/cleaned_orders.xlsx` (912 records after removing exact duplicates, 44 columns including new calculated and flag columns)

## 3. Tools Used

- **Excel ** — all deliverable workbooks, formatting, conditional fills, and live formulas

## 4. Cleaning Steps Performed

1. Trimmed and standardized casing/spacing across all text fields (customer_name, segment, region, state, city, category, sub_category, ship_mode, payment_status, order_status).
2. Parsed four different raw date formats (`YYYY-MM-DD`, `MM/DD/YYYY`, `DD-MM-YYYY`, `DD Mon YYYY`) into a single consistent date type for `order_date` and `ship_date`.
3. Flagged 21 records where `ship_date` was earlier than `order_date`.
4. Removed 20 exact duplicate rows; flagged (did not delete) 24 rows belonging to 12 `order_id`s with conflicting data.
5. Cleaned the `discount` field — converted text percentages, flagged negative values and values above the assumed 50% cap.
6. Added calculated columns: `cleaned_discount`, `calculated_sales`, `calculated_profit`, `profit_margin`, `shipping_delay_days`, `order_month`, `order_year`, and an overall `data_quality_flag` (Clean / Warning / Invalid) — all built as live Excel formulas.



## 5. Business Rules Applied

| Rule Area | Action |
|---|---|
| Missing `region` / `ship_mode` | Filled as "Unknown", flagged |
| Missing `discount` | Set to 0 only if quantity/unit price/cost were valid |
| Negative or out-of-range discount | Flagged invalid, not silently corrected |
| Cancelled orders / Failed payments | Excluded from completed-sales pivot summaries |
| Refunded orders | Summarized separately |
| Ship date before order date | Flagged as invalid shipping record |

## 6. Summary of Data Quality Issues Found

| Issue | Count |
|---|---|
| Exact duplicate rows (removed) | 20 |
| Duplicate order_ids with conflicting data (flagged) | 12 groups / 24 rows |
| Missing region | ~25 |
| Missing ship_mode | ~21 |
| Missing discount | 18 |
| Invalid discount (negative / above 50%) | ~30 |
| Ship date before order date | 21 |
| Sales/profit calculation mismatches | 59 |
| **Final: Clean / Warning / Invalid** | **756 / 44 / 112** (of 912 records) |



## 7. Summary of Final Pivot Reports

`outputs/pivot_summary.xlsx` contains:
1. Sales & profit by region
2. Sales & profit by category and sub-category
3. Order count by ship mode
4. Profit margin by customer segment
5. Refunded/cancelled/failed orders by region
6. Monthly sales trend
7. Refunded orders — separate summary

All completed-sales pivots exclude Cancelled orders and Failed payments, per the business rules.

## 8. Key Business Insights

- **South and West** are the top-performing regions by completed sales (~₹1.80M each), but **East** has the highest profit despite slightly lower sales, indicating better margins there.
- **Furniture** has the highest profit margin (~29.3%) among the three product categories, narrowly ahead of Technology and Office Supplies.
- **Corporate** customers generate the highest profit margin (~29.6%) of all segments, even though **Small Business** generates the highest total sales volume.
- Roughly **16% of all orders** are Cancelled, Failed-payment, or Refunded and are excluded from / reported separately from the completed-sales figures.
- "Same Day" shipments show an average shipping delay of ~4 days in the data — a likely data-entry inconsistency worth raising with the source system owner.

## 9. Assumptions and Limitations

- Valid discount range assumed to be 0%–50%, based on the distribution of clean values in the data.
- Conflicting duplicate `order_id` records were flagged, not resolved — a business decision is needed on which version is authoritative.
- Original (reported) `sales`/`profit` values were preserved alongside recalculated ones rather than overwritten, since the source of the mismatch (discount, quantity, or unit price) could not be confirmed.



## 10. Screenshots

| File | Shows |
|---|---|
| `screenshots/raw_data_preview.png` | Raw dataset before cleaning |
| `screenshots/cleaned_data_preview.png` | Cleaned dataset with calculated columns |
| `screenshots/pivot_summary_1.png` | Sales & profit by region |
| `screenshots/pivot_summary_2.png` | Profit margin by customer segment |
