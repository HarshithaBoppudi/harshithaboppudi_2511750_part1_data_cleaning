# Cleaning Log – Retail Order Data (Part 1)

## 1. Issues Found in Raw Data

| # | Issue | Details |
|---|-------|---------|
| 1 | Inconsistent text casing/spacing | `customer_name`, `segment`, `region`, `state`, `city`, `category`, `sub_category`, `ship_mode`, `payment_status`, `order_status` contained mixed case (e.g. `SMALL BUSINESS`, `corporate`), leading/trailing spaces, and double internal spaces (e.g. `"Vikram  Iyer"`). |
| 2 | Multiple date formats | `order_date` and `ship_date` contained four different formats: `YYYY-MM-DD`, `MM/DD/YYYY`, `DD-MM-YYYY`, and `DD Mon YYYY` (e.g. `21 Jul 2024`). |
| 3 | Ship date before order date | 21–22 records had a `ship_date` earlier than the `order_date`, which is logically impossible. |
| 4 | Missing values | `region` missing in 25–26 rows; `ship_mode` missing in 21–22 rows; `discount` missing in 18 rows. |
| 5 | Invalid discount values | Negative discounts (e.g. `-0.19`), discounts stored as text percentages (`"70%"`, `"85%"`), and discounts above a reasonable cap (e.g. `0.55`, `0.65`). |
| 6 | Exact duplicate rows | 20 fully identical rows (10 pairs) were found across the dataset. |
| 7 | Duplicate `order_id` with conflicting data | 12 `order_id` values appeared more than once with different `sales`, `profit`, or `order_status` values in each occurrence — these are NOT simple duplicates. |
| 8 | Sales/profit calculation mismatches | 59 rows had a reported `sales` value that did not equal `quantity × unit_price × (1 − discount)`, and a corresponding mismatch in `profit`. |
| 9 | Order/payment status inconsistencies | Status fields used inconsistent casing (`Completed`, `COMPLETED`, `completed`, `  Cancelled`). |

## 2. Cleaning Actions Performed

- **Text fields**: Trimmed leading/trailing spaces, collapsed multiple internal spaces, and applied Title Case to standardize casing across `customer_name`, `segment`, `region`, `state`, `city`, `category`, `sub_category`, `ship_mode`, `payment_status`, and `order_status`.
- **Dates**: Parsed all four date formats found in the raw file into a single, consistent `YYYY-MM-DD` date type for both `order_date` and `ship_date`.
- **Duplicates**:
  - Exact duplicate rows were removed, keeping the first occurrence of each (20 rows removed).
  - Duplicate `order_id`s with conflicting data were **kept** and flagged (`duplicate_conflict_flag = TRUE`) for manual review rather than silently deleted, since it isn't possible to know which version is correct without going back to the source system.
- **Discount cleaning**: Text-percentage values (`"70%"`) were converted to decimals (`0.70`). Missing discounts were set to `0` only when `quantity`, `unit_price`, and `cost` were all present and valid. Negative and above-range discounts were flagged as invalid and capped to the nearest valid bound (`0%–50%`) purely for downstream sales/profit calculations, while the original issue remains visible via `discount_negative_flag` / `discount_above_range_flag`.
- **Calculated columns added**: `cleaned_discount`, `calculated_sales`, `calculated_profit`, `profit_margin`, `shipping_delay_days`, `order_month`, `order_year`, `data_quality_flag` (all built as live Excel formulas in `cleaned_orders.xlsx` so they recalculate automatically).

## 3. Business Rules Applied

| Rule Area | Action Taken |
|---|---|
| Missing `region` | Filled as `"Unknown"`, flagged in quality report (`region_missing_flag`). |
| Missing `ship_mode` | Filled as `"Unknown"`, flagged in quality report (`ship_mode_missing_flag`). |
| Missing `discount` | Treated as `0` only if `quantity`, `unit_price`, and `cost` were all valid; otherwise left flagged. |
| Negative discount | Flagged invalid (`discount_negative_flag`); not silently corrected. |
| Discount above allowed range (>50%) | Flagged invalid (`discount_above_range_flag`). |
| Cancelled orders | Excluded from the completed-sales pivot summaries via `contributes_to_completed_sales = FALSE`. |
| Failed payments | Excluded from the completed-sales pivot summaries via `contributes_to_completed_sales = FALSE`. |
| Refunded orders | Summarized separately in `pivot_summary.xlsx` → "Refunded Orders Summary" sheet. |
| Ship date before order date | Flagged as an invalid shipping record (`ship_before_order_flag`); not deleted. |

## 4. Assumptions Made

1. The valid discount range is assumed to be **0%–50%**, consistent with the bulk of the clean discount values found in the data (max clean value observed was 25–30%; outliers like 55%–85% were treated as data entry errors).
2. Where multiple spelling/casing variants clearly referred to the same category (e.g. `"Office  Supplies"` vs `"Office Supplies"`), they were merged into a single standardized value.
3. For duplicate `order_id`s with conflicting `order_status`/`sales`, no single record was assumed "correct" — both were preserved and flagged rather than guessing which is the source of truth.
4. A record is treated as "Invalid" overall if it has at least one hard data-integrity issue (invalid date, ship-before-order, invalid discount, sales/profit mismatch, or unresolved duplicate conflict); it is "Warning" if it only has a filled missing-value flag; otherwise it is "Clean".

## 5. Records Removed

- **20 rows** removed — exact duplicate rows (10 duplicate pairs), keeping one copy of each.
- No other rows were deleted; all other issues were flagged rather than removed, per the assignment instructions.

## 6. Records Flagged

- **24 rows** flagged for `duplicate_conflict_flag` (12 conflicting `order_id` groups).
- **~25 rows** flagged for missing `region`.
- **~21 rows** flagged for missing `ship_mode`.
- **18 rows** flagged for missing `discount`.
- **~30 rows** flagged for invalid discount (negative or above range).
- **~21 rows** flagged for `ship_before_order_flag`.
- **59 rows** flagged for `sales_mismatch_flag` / `profit_mismatch_flag`.
- Overall: **112 rows = Invalid**, **44 rows = Warning**, **756 rows = Clean** (out of 912 rows remaining after duplicate removal).

## 7. Limitations of This Cleaning Process

- Conflicting duplicate `order_id` records could not be resolved automatically — a human reviewer with access to the source transactional system would need to determine which version is correct.
- The 50% discount cap is an assumption based on the shape of the data, not a stated company policy; the actual company discount policy may differ.
- Sales/profit mismatches were flagged but the original (reported) `sales`/`profit` values were preserved alongside the recalculated ones rather than overwritten, since it isn't certain whether the error lies in the original values or in the discount/quantity/unit price fields.
- "Same Day" shipments show an average shipping delay of ~4 days in the cleaned data, which is logically inconsistent with the ship-mode label — this points to a likely upstream data-entry issue in either `ship_mode` or the dates that could not be corrected with the information available.

