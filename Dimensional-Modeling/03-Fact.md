# Fact Tables

A fact table stores the numeric measurements produced by a business process, at the grain declared in the design process. It is the largest table in a dimensional model (typically more than 90% of the storage), and its design determines whether the model gives correct answers.

## Contents

1. Fact table anatomy
2. Fact table types
   * 2.1 Transaction fact tables
   * 2.2 Periodic snapshot fact tables
   * 2.3 Accumulating snapshot fact tables
   * 2.4 Factless fact tables
   * 2.5 Consolidated fact tables
   * 2.6 Aggregate fact tables
   * 2.7 Comparison of the types
3. Measure types: additive, semi-additive, non-additive
4. Ratios, percentages, and unit prices
5. Nulls in fact table foreign keys and measures
6. Fact table surrogate keys
7. Late-arriving facts
8. Header/line item patterns and allocation
9. Multiple currencies and units of measure
10. Year-to-date facts
11. Heterogeneous products: supertype and subtype fact tables
12. Real-time fact table partitions
13. Common mistakes
14. Self-check questions

---

## 1. Fact Table Anatomy

A fact table row represents one measurement event at the declared grain. A typical fact table has four kinds of columns.

| Column type | Purpose | Example |
|---|---|---|
| **Foreign keys** | Surrogate keys that link to each dimension | `date_key`, `product_key`, `store_key` |
| **Measures (facts)** | Numeric values from the process | `sales_quantity`, `net_sales_amount` |
| **Degenerate dimensions** | Identifiers with no dimension table of their own | `receipt_number`, `invoice_number` |
| **Metadata/audit** | Lineage and ETL information | `audit_key`, load timestamp |

Key characteristics:

* **Grain first.** Every row, every foreign key, and every measure must be consistent with the grain.
* **Deep and narrow.** Many rows, relatively few columns. Dimension tables are the opposite: few rows and many columns.
* **Sparse (for transactions).** A row exists only when the event happens. There are no rows for products that did not sell.
* **Primary key.** Usually a **composite key** made from a subset of the foreign keys (and often a degenerate dimension) that uniquely identifies a row at the grain. Not every combination of foreign keys is a valid key. Some designs add a surrogate row key (see Section 6).
* **Foreign keys are never null.** See Section 5.
* **Textual descriptions do not belong in fact tables.** Put them in dimensions. Numeric values that are used for filtering or grouping (such as a list price used for banding) are often better as dimension attributes, and numeric values used in calculations are facts. Some values are both.

Example transaction fact table:

```sql
CREATE TABLE fact_sales (
    date_key            INT           NOT NULL,   -- FK to dim_date
    product_key         INT           NOT NULL,   -- FK to dim_product
    store_key           INT           NOT NULL,   -- FK to dim_store
    promotion_key       INT           NOT NULL,   -- FK to dim_promotion
    receipt_number      VARCHAR(20)   NOT NULL,   -- degenerate dimension
    line_number         SMALLINT      NOT NULL,   -- degenerate dimension
    sales_quantity      INT,
    gross_sales_amount  DECIMAL(12,2),
    discount_amount     DECIMAL(12,2),
    net_sales_amount    DECIMAL(12,2),
    cost_amount         DECIMAL(12,2),
    audit_key           INT           NOT NULL,
    PRIMARY KEY (receipt_number, line_number)     -- unique at the grain
);
```

---

## 2. Fact Table Types

There are three fundamental grains (transaction, periodic snapshot, accumulating snapshot), plus related patterns: factless, consolidated, and aggregate fact tables.

### 2.1 Transaction Fact Tables

* **Grain:** one row per measurement event at a point in time (one scan, one payment, one shipment line).
* Row exists **only when the event occurs**, so the table is sparse across dimensions.
* The most **atomic and flexible** type. It supports the greatest number of dimensions and the most detailed questions.
* Measures are usually **fully additive**.
* Rows are generally **inserted and never updated** (corrections are handled as new adjusting rows or reloads).
* Typically has a date key and often a timestamp (or a separate time-of-day dimension key) to capture when the event occurred.
* Examples: retail POS line items, call detail records, web click events, bank transactions.

### 2.2 Periodic Snapshot Fact Tables

* **Grain:** one row per entity per regular period (for example, one row per account per day or month).
* A row exists for **every entity in every period**, even when nothing happened, so the table is **dense**.
* Used to see **status at the end of a period**, which is hard to get from a transaction table (for example, "what was the balance on this date?").
* Typical measures: balances and levels (**semi-additive**), plus period activity totals (additive).
* Dimensions are usually fewer than in a transaction table, because the snapshot summarizes activity over the period.
* Load on a predictable schedule, appending a set of rows each period.
* Examples: daily account balances, month-end inventory levels, weekly open-order backlog.
* Often built **alongside** a transaction fact table: the transaction table for detail, the snapshot for easy balance and trend analysis.
* The size can grow quickly. Consider snapshotting only the changed entities, or using a longer period for older data, if volume is a concern, and document the trade-off.

### 2.3 Accumulating Snapshot Fact Tables

* **Grain:** one row per instance of a **process with a defined beginning and end**, such as one row per order line, claim, or student application.
* The row is **updated** as the instance moves through its milestones, so it is the one fact table type where rows are routinely revised.
* Typical structure:
  * **Several date foreign keys** (role-played from the date dimension), one for each milestone: order date, requested ship date, actual ship date, delivery date, payment date.
  * Dimensions that are known at the start (customer, product), and sometimes ones that are filled in later (such as a warehouse).
  * **Lag or duration facts** between milestones (for example, days from order to ship), which are calculated and updated by ETL.
  * Often a **status** or current-stage attribute (via a dimension).
  * A count fact of 1 for convenience.
* Milestone dates not yet reached point to a **special date dimension row** such as "Not yet occurred" (never null).
* Best suited to **short-lived, predictable pipelines**. A process with loops, many repeated steps, or no clear end is better modelled with a transaction fact table and a step dimension.
* Examples: order fulfilment pipeline, insurance claim lifecycle, mortgage application, student admission funnel.

Example of the update pattern when an order ships:

```sql
UPDATE fact_order_pipeline
SET    ship_date_key       = :ship_date_key,
       order_to_ship_days  = :lag_days,
       audit_key           = :audit_key
WHERE  order_number = :order_number AND order_line = :order_line;
```

### 2.4 Factless Fact Tables

A factless fact table has **no numeric measures**. It records that an event occurred, or that a relationship exists, using only foreign keys. Often a column holding the constant `1` is included so counting is easy, but it carries no information.

There are two uses:

* **Event tracking tables:** record that an event took place with no measurement associated with it. Example: student attendance (date, student, class, facility, teacher). Questions are answered by **counting rows**.
* **Coverage tables:** record what **could** happen or which combinations are eligible, so you can answer questions about what did **not** happen. Example: a table of which products are on promotion in which stores on which days. Joined to the sales table, it answers "which promoted products did not sell?" which sales alone cannot show, because no sale means no row.

The pattern for "what didn't happen":

> (all eligible combinations from the coverage table) **minus** (combinations that actually occurred in the transaction fact table)

### 2.5 Consolidated Fact Tables

* Combine facts from **multiple business processes** into one table because they share the **same grain and conformed dimensions** and are frequently analyzed together.
* Classic example: **actual vs. budget** (or forecast) at the same grain, such as month, product, and region.
* Benefits: simpler queries, since there is no need for drill-across SQL, and easy calculation of variances and ratios.
* Costs and cautions:
  * Requires the processes to be expressed at the same grain, which may force a higher-level (less atomic) grain.
  * More complex ETL, because several sources must be aligned and loaded together.
  * Should **supplement** the separate, atomic fact tables rather than replace them.
* Build the separate fact tables first, and add the consolidated one as a convenience when the demand is clear.

### 2.6 Aggregate Fact Tables

* Summarized copies of the atomic fact table, built for **query performance**.
* Created by rolling up one or more dimensions to a higher level, using **shrunken dimension** versions (for example, month instead of day, or category instead of product) or by dropping a dimension.
* Principles:
  * Aggregates are **derived from** the atomic table, and must give the **same answers** for the same query.
  * They are an optimization, so they should be **transparent to users**. An **aggregate navigator** (in the BI tool, database optimizer, or a materialized view or OLAP cube) redirects queries to the smallest table that can answer them.
  * Use the same conformed dimension attributes (or shrunken dimensions that are subsets of the base dimension).
  * Rebuild or update aggregates as part of the ETL load, so they stay in sync with the atomic data.
* Aggregates **never replace** atomic data.

### 2.7 Comparison of the Types

| | Transaction | Periodic snapshot | Accumulating snapshot |
|---|---|---|---|
| **Grain** | One row per event | One row per entity per period | One row per process instance |
| **Row exists when** | Event occurs | Every period (dense) | Process instance starts |
| **Updates** | Insert only | Insert only (new period) | Rows updated as milestones occur |
| **Date keys** | One (the event date) | One (the period) | Multiple (milestones) |
| **Typical measures** | Additive amounts | Balances (semi-additive) and period totals | Lags/durations, amounts, counts |
| **Typical size** | Largest | Large (grows by schedule) | Smallest (one row per instance) |
| **Answers** | What happened? | What was the status at the end of the period? | How is the pipeline performing, and how long does each stage take? |

---

## 3. Measure Types: Additive, Semi-Additive, Non-Additive

The additivity of a measure determines how it can be summarized across dimensions. It is the most important property to understand about each fact.

| Type | Definition | Examples | How to summarize |
|---|---|---|---|
| **Additive** | Can be summed across **all** dimensions | Sales quantity, sales amount, cost | `SUM` across any dimension |
| **Semi-additive** | Can be summed across **some** dimensions, but not across others (usually time) | Account balance, inventory on hand, headcount | `SUM` across non-time dimensions; use **average, min, max, or last value** across time |
| **Non-additive** | Cannot be meaningfully summed across **any** dimension | Unit price, ratios, percentages, temperatures | Use `AVG`, `MIN`, `MAX`, `COUNT`, or compute from additive components |

Important points:

* **Account balances:** summing balances across accounts on a given day is meaningful. Summing the same account's balance across days is not. Use the average (sum of balances divided by the number of days), or the period-end value, over time.
* **Semi-additive facts are most common in periodic snapshots.** Watch out for BI tools that apply `SUM` by default.
* **Counts:** a row count (`COUNT(*)`) or a count of distinct values (`COUNT(DISTINCT ...)`) is not the same as summing a fact. Counting distinct values is not additive across dimensions.
* **Prefer additive facts.** Whenever a non-additive measure can be broken into additive components, store the components (Section 4).
* Document the additivity of every measure in the model documentation and in the semantic layer.

Example of using a semi-additive balance across time:

```sql
-- Average daily balance for each customer in a month
SELECT customer_key, SUM(balance_amount) / COUNT(DISTINCT date_key) AS avg_daily_balance
FROM   fact_account_daily_balance
WHERE  date_key BETWEEN 20260901 AND 20260930
GROUP  BY customer_key;
```

---

## 4. Ratios, Percentages, and Unit Prices

**Rule: store the additive components, and calculate the ratio after aggregation.**

* A ratio (such as gross margin %, or discount %) cannot be summed, and the **average of ratios is not the ratio of sums** unless the denominators are equal.
* Store the **numerator and denominator** as separate additive facts (for example, `gross_margin_amount` and `net_sales_amount`). Compute `SUM(numerator) / SUM(denominator)` in the query, report, or semantic layer.
* This applies to unit prices as well: store **quantity** and **extended amount**, and calculate `SUM(extended_amount) / SUM(quantity)` for an average price at any level of aggregation. If you store a unit price, mark it as non-additive.
* Do the division in the BI layer, to avoid divide-by-zero errors and to keep the results consistent.

Example:

| Row | Sales amount | Cost amount | Margin % (stored) |
|---|---|---|---|
| A | 100 | 80 | 20% |
| B | 900 | 540 | 40% |

* Average of the stored percentages = 30%, which is **wrong**.
* Ratio of sums = (1000 − 620) / 1000 = **38%**, which is correct.

Where a derived value is needed for convenience (for example, a unit price used in a lookup), it can be stored in addition to the components, as long as it is flagged as non-additive.

---

## 5. Nulls in Fact Table Foreign Keys and Measures

### Foreign keys: never null

* A null foreign key breaks referential integrity, and inner joins silently **drop** those rows from query results.
* Instead, point every foreign key to a **default dimension row** that explains the absence. Typical default rows (each with its own surrogate key):
  * **Unknown:** the value exists but we don't know it.
  * **Not applicable:** the dimension does not apply to this event.
  * **Not yet occurred / TBD:** the event hasn't happened yet (common in accumulating snapshots, where the date dimension has a future "unknown" row).
  * **Invalid / corrupted:** the source value was bad (also logged in the error event schema).
* Use consistent labels, so users see "Unknown" and not a blank.

### Measures: nulls are acceptable

* A **null measure** means "no value is available", and standard SQL aggregate functions (`SUM`, `AVG`, `COUNT(column)`, `MIN`, `MAX`) **ignore nulls** correctly.
* Do **not** replace unknown measures with zero. Zero is a value, and it distorts averages and counts.
* Be aware of the difference between **null and zero** in the business definition (for example, "no sale" vs. "unknown sales").
* In some designs (such as sparse fact tables), consider whether a row should exist at all, rather than a row full of nulls.

---

## 6. Fact Table Surrogate Keys

A **fact table surrogate key** (also called a row key) is a single-column, meaningless, system-assigned integer that identifies each fact row.

When it is useful:

* **ETL management:** uniquely identify a row so it can be updated, deleted, or reloaded, particularly when the natural primary key is a long composite.
* **Restart and recovery:** support restartable loads and track which rows a batch inserted, together with the audit key.
* **Replacing a long composite key** for operations such as updates (as in accumulating snapshots) and as a **single-column reference** for child tables.
* **Parent/child fact structures:** linking a child fact table to its parent fact row.
* **Resolving duplicates and corrections:** distinguish rows that are identical on every other column.

Cautions:

* The surrogate key has **no business meaning** and is not used by users or BI tools to join to other tables.
* It does **not** replace the need to define the **grain** and the natural composite key that uniquely identifies a row at that grain.
* It is not required for every fact table, and it adds storage. Many designs work well without one.

---

## 7. Late-Arriving Facts

A **late-arriving fact** is a measurement that reaches the warehouse **after** the time the event occurred (for example, a sale that is loaded days later, or a transaction from an offline system).

Handling:

1. **Use the dimension version that was in effect at the time of the event**, not the current version. For Type 2 dimensions, look up the surrogate key using the **business key and the transaction date**, within the row's effective and expiry dates.
2. **Missing dimension rows:** if the dimension member does not exist yet (the fact arrives before its dimension), create an **inferred member** placeholder row, and update it when the full data arrives.
3. **Use the correct date key.** Use the event date (not the load date) for the date foreign key. Record the load date separately, for example in the audit dimension.
4. **Update dependent tables.** A late fact may require reprocessing:
   * **Periodic snapshots** for the periods after the event (balances change).
   * **Accumulating snapshots** (milestones are updated).
   * **Aggregates** and OLAP cubes.
5. **Monitor and report** the amount of late data, and agree with the business on how long the warehouse will accept it, and how corrections show in already published reports.

The related case of **early-arriving facts** (the fact arrives before the dimension row exists) is handled with inferred members.

---

## 8. Header/Line Item Patterns and Allocation

Many transactions have a **header** (order, invoice, or claim) and several **lines** (items on the document). Dimensional design handles this with the grain set at the **lowest level, the line item**.

### Design

* **Grain:** one row per line item.
* The **header's dimensions** (customer, order date, sales rep, ship-to location) are **copied to every line row** as foreign keys. They do not need a separate header table, so queries need no extra joins.
* The **header identifier** (order number) is stored as a **degenerate dimension**. It lets users group lines back into orders.
* Where a header carries **descriptive** attributes (such as the payment terms or order type), place them in a dimension, which may be a junk dimension.

### Header-level facts: allocation

Some facts exist only at the header level, such as a **shipping charge**, an order-level discount, or tax on the whole invoice. If they are repeated on each line, the totals are **overstated**. There are two options:

1. **Allocate** the header-level amount to the lines (for example, proportionally by line amount or weight), so that the allocated amounts add up to the header total. The measure is then additive at the line grain.
2. **Keep it at the header grain** in a separate header-level fact table, which can be analyzed alongside the line table by the shared dimensions and degenerate order number.

Allocation rules:

* The business must **agree on the allocation method** and document it.
* The allocated amounts must **sum to the original** header amount, with rounding differences assigned deterministically (for example, to the last line).
* Never repeat the whole header amount on every line.

### Multi-level documents

Some processes have three levels (for example, purchase order header, line, and delivery schedule). Choose the lowest level that users need as the grain, and allocate or represent the upper-level facts as above.

---

## 9. Multiple Currencies and Units of Measure

### Currencies

* A global business records transactions in many currencies and reports in one or a few.
* Recommended pattern:
  * Store the amounts in the **local (transaction) currency** and also in the **standard (reporting) currency**, converted by ETL at the agreed rate. Both are facts in the same row.
  * Include a **currency dimension** key to identify the transaction currency.
  * Maintain a **currency conversion fact table** (grain: date, from-currency, to-currency, rate) to support conversions to other currencies at query time, and for rate types that differ (daily rate, month-end rate, budget rate).
* Agree on which **exchange rate and date** apply (transaction date, month-end, or a fixed budget rate), and apply it consistently.
* Never add amounts in different currencies together without converting them.

### Units of measure

* A product may be recorded in different units (each, case, pallet, kilogram) across sources.
* Options:
  * Store quantity in a **standard unit** (and optionally also in the original unit), converted in ETL.
  * Store the original quantity with a **conversion factor** in the fact table or product dimension, and calculate the standard quantity in a view or semantic layer.
* Do not let users add quantities in different units. Prefer providing the standardized measure as the default.
* Document the standard unit for each measure in the data dictionary.

---

## 10. Year-to-Date Facts

**Avoid storing year-to-date (YTD) or other cumulative measures in fact tables.**

Reasons:

* A YTD value is **non-additive across time** (adding January YTD and February YTD double counts).
* It is easy to make wrong by summing across periods, and it locks the table to a single definition of the year (calendar versus fiscal).
* It breaks the rule that every fact is true to the grain.
* It is hard to correct when late-arriving data or adjustments arrive.

Instead:

* Store the **period or transaction-level** values.
* Calculate YTD, QTD, and running totals **at query time** or in the BI/OLAP/semantic layer, using the date dimension (fiscal year, month number, and so on).

```sql
SELECT d.calendar_month, SUM(f.net_sales_amount) AS month_sales,
       SUM(SUM(f.net_sales_amount)) OVER (ORDER BY d.calendar_month_number) AS ytd_sales
FROM   fact_sales f JOIN dim_date d ON f.date_key = d.date_key
WHERE  d.calendar_year = 2026
GROUP  BY d.calendar_month, d.calendar_month_number;
```

---

## 11. Heterogeneous Products: Supertype and Subtype Fact Tables

Businesses such as banks, insurers, and telecoms sell **different types of products** that share some measures but have their own **unique facts and attributes** (for example, checking, savings, mortgage, and credit card accounts). Putting them all in one wide table would leave many null columns, and separate tables make cross-product analysis difficult.

The Kimball pattern uses both:

| Table | Purpose |
|---|---|
| **Supertype fact table** | Contains the **facts common to all** product types (for example, balance and transaction amount). Supports analysis across all products. |
| **Subtype fact tables** (one per product type) | Contain the **custom facts** specific to the type (for example, mortgage: principal, interest, escrow). Used for detailed analysis within the type. |

Dimensions follow the same pattern:

* A **supertype dimension** (such as Account or Product) with attributes common to all types.
* **Subtype dimensions** with attributes specific to each type, joined through the same key.

Guidelines:

* Use the supertype table when users want to compare across the whole product range, and the subtype tables when analyzing a single product type in depth.
* Keep the grain consistent between the supertype and subtype tables, so rows can be tied back together.
* ETL must load both sets of tables, and they must reconcile with each other.
* This avoids wide tables with hundreds of mostly null columns.

---

## 12. Real-Time Fact Table Partitions

Business users sometimes need data within minutes, while the standard batch process loads fact tables once a day. The Kimball approach is to **add a real-time partition** to the fact table.

* A **static partition** holds historical data, loaded by the regular batch ETL. It is fully indexed, aggregated, and optimized for queries.
* A **real-time (hot) partition** holds only the **current period's** data (for example, today's rows). It is loaded continuously (or in micro-batches) by a streaming or trickle-feed process.
  * It has **minimal or no indexes** and **no aggregates**, so that loading is fast and it can handle a continuous flow of inserts.
  * It is often kept in memory, or on fast storage.
  * It is often presented to users with the static partition through a **view** that unions the two, so queries see a single logical fact table.
* At the end of the period (usually each night), the real-time rows are **merged into the static partition** by the regular ETL process, which also builds the indexes and aggregates, and the real-time partition is emptied.

Considerations:

* **Dimensions:** real-time rows need dimension keys. New dimension members may arrive before the full dimension update, and are handled with **inferred members**. Changes to Type 2 attributes during the day may be handled only in the nightly process.
* **Consistency:** real-time data may later be **corrected** by the batch load. Users should be told that the current-day data is provisional.
* **Aggregates and cubes** typically exclude the real-time partition until the nightly merge.
* The additional complexity and cost should be justified by a **real business need** for low latency, since most analytic questions do not need it.
* Modern streaming and cloud platforms implement the same idea, with micro-batching or change data capture into incremental tables.

---

## 13. Common Mistakes

* Mixing grains in one fact table (for example, line-level and header-level amounts).
* Not declaring the grain, or defining it in terms of dimensions rather than business events.
* Storing ratios, percentages, or unit prices as the only facts.
* Using `SUM` on semi-additive facts across time.
* Using null foreign keys instead of default dimension rows.
* Replacing null measures with zeros.
* Repeating header-level facts on every line without allocation.
* Storing YTD or cumulative measures.
* Looking up the **current** dimension version for historical or late-arriving facts.
* Putting text descriptions or flags into the fact table instead of dimensions.
* Using an accumulating snapshot for a process with no clear end or with loops.
* Replacing atomic data with aggregates.
* Adding currencies together without conversion.

---

## 14. Self-Check Questions

1. What are the four kinds of columns in a fact table?
2. Why is the grain declared before the facts are chosen?
3. Compare transaction, periodic snapshot, and accumulating snapshot fact tables: grain, row creation, updates, and typical measures.
4. Why does an accumulating snapshot have several date keys, and what do unreached milestones point to?
5. What are the two uses of a factless fact table? How would you find products that were on promotion but did not sell?
6. When would you build a consolidated fact table, and what are the risks?
7. What makes an aggregate fact table valid, and what is an aggregate navigator?
8. Give an example of an additive, a semi-additive, and a non-additive fact. How do you summarize each?
9. Why is the average of percentages wrong, and what should you store instead?
10. Why must fact table foreign keys never be null, and what do you use instead?
11. Why is a null measure treated differently from a null foreign key?
12. When is a fact table surrogate key useful, and why is it not used for joins?
13. How should a late-arriving fact choose its dimension row?
14. How do you handle a header-level shipping charge in a line-item fact table?
15. How should a fact table represent amounts in multiple currencies?
16. Why avoid YTD facts, and how do you calculate YTD instead?
17. Why use supertype and subtype fact tables for heterogeneous products?
18. How does a real-time partition work, and what are its trade-offs?

---

## Quick Reference

| Topic | Rule of thumb |
|---|---|
| Grain | Declare it first; use the lowest atomic level available |
| Keys | Foreign keys never null; use default dimension rows |
| Measures | Prefer additive; store components of ratios |
| Balances | Semi-additive: do not sum across time |
| Transactions | Insert-only, sparse |
| Periodic snapshots | Dense, one row per entity per period |
| Accumulating snapshots | One row per process instance, updated at milestones |
| Header facts | Allocate to line grain, or keep at header grain separately |
| Currency | Store local and standard amounts, plus a rate table |
| YTD | Calculate at query time, do not store |
| Late facts | Look up the dimension version effective at event time |
| Aggregates | Supplement, never replace, atomic data |
