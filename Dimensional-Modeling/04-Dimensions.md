# Dimension Tables

Dimension tables hold the descriptive context for the measurements in a fact table: the who, what, where, when, why, and how. They are the main source of the labels used for filtering, grouping, and drilling, and they are what makes a dimensional model understandable to business users. The quality of the dimensions determines the usability of the whole warehouse.

## Contents

1. Dimension anatomy: grain, surrogate/natural/durable keys, and attributes
2. Descriptive, textual, denormalized attribute design
3. Attribute quality: decoding and avoiding cryptic flags
4. Fixed-depth hierarchies and flattening
5. The date/calendar dimension
6. Time-of-day dimension vs. timestamp attributes
7. Role-playing dimensions
8. Degenerate dimensions
9. Junk dimensions
10. Conformed dimensions
11. Shrunken (rollup) dimensions
12. Outrigger and snowflaked dimensions
13. Mini-dimensions
14. Swappable dimensions
15. Heterogeneous supertype/subtype dimensions (and the abstract generic dimension anti-pattern)
16. Dimension-to-dimension relationships
17. Unknown, not applicable, and null values
18. Audit dimension
19. Common mistakes
20. Self-check questions

---

## 1. Dimension Anatomy: Grain, Keys, and Attributes

A dimension table is **wide and shallow**: often 50 to 100 columns, but far fewer rows than the fact table. Each row describes one member of the dimension (a product, a customer, a store).

| Element | Purpose |
|---|---|
| **Surrogate key** | The primary key; a meaningless integer assigned by the warehouse |
| **Natural (business) key** | The identifier from the source system, kept as an attribute |
| **Durable key** | A permanent, warehouse-assigned identifier for the entity that never changes |
| **Descriptive attributes** | The text and categories used for filtering, grouping, and labelling |
| **Change-tracking columns** | Row effective date, expiry date, and current flag (for Type 2 changes, see Module 5) |
| **Audit columns** | Source system, load batch, and similar metadata |

### Surrogate keys

* A **simple integer**, generated sequentially by the ETL process, with **no business meaning**.
* The fact table's foreign key points to the dimension's surrogate key, never to the source system key.
* Why use them:
  * **Isolation from source changes.** Source keys can be reused, reformatted, or reassigned, or can differ across systems.
  * **Support for history.** Multiple rows (versions) can exist for the same natural key, each with its own surrogate key (Type 2).
  * **Integration.** The same entity from several source systems can be mapped to one surrogate key.
  * **Default rows.** Special rows for "Unknown" and "Not applicable" can be given reserved keys.
  * **Performance.** Small integer keys are compact and join efficiently.
* **Exception:** the date dimension may use a meaningful key such as `20261008` (see Section 5). It is the only common case.

### Natural (business) keys

* Keep the source system's key in the dimension as an ordinary attribute, to support ETL lookups and to let users find rows by familiar codes.
* Natural keys are **not** used as the primary key, and are not placed in the fact table (except as degenerate dimensions).
* With multiple source systems, there can be several natural keys per row (and a "source system" column).

### Durable keys (supernatural keys)

* A **durable key** is a persistent identifier created and managed by the warehouse for an entity, which **never changes** even if the source natural key changes, is reassigned, or the entity is merged from several sources.
* All Type 2 versions of one entity share the same durable key, but each has its own surrogate key.
* Use cases: tracking an entity over time when the business key is not stable (for example, an employee ID that is reissued after rehire), linking versions of the same entity, and counting distinct entities across versions.

Example:

| customer_key (surrogate) | customer_durable_key | customer_id (natural) | customer_name | city | row_effective | row_expiry | is_current |
|---|---|---|---|---|---|---|---|
| 1001 | 5001 | C-77 | Anita Rao | Pune | 2022-01-01 | 2025-05-31 | N |
| 2417 | 5001 | C-77 | Anita Rao | Mumbai | 2025-06-01 | 9999-12-31 | Y |

* Some designs also store the durable key in the fact table, next to the surrogate key (a **Type 7** design), so a query can report facts using either the attributes as they were at the time or the entity's current attributes. See Module 5.

### Dimension grain

Every dimension also has a **grain**: one row per member of the entity it describes, such as one row per product, per store, or per day (or per **version** of that entity, when Type 2 history is kept).

* State the grain and test it, so that no two rows describe the same member at the same time.
* The key design rule is that, for a fact row at the declared fact grain, a dimension must supply **exactly one** row, with one value for each attribute. If a fact row could legitimately relate to several rows of a dimension, the design must change (see multivalued dimensions and bridge tables in Module 6).
* Rows at different grains should not be mixed in one table. A shrunken dimension (Section 11) is the standard way to serve facts captured at a higher grain.

### Attributes

* The **descriptive attributes** are the real value of the dimension. Spend design effort on making them rich, accurate, and understandable.
* More attributes means more ways for users to slice and dice.
* Attributes should be named in business language.

---

## 2. Descriptive, Textual, Denormalized Attribute Design

* **Textual and descriptive.** Dimension attributes are mostly text (names, descriptions, categories), although some are numeric (such as package size).
* **Low-cardinality text repeated on many rows is fine.** Do not normalize it away to save space. Dimension tables are small compared with fact tables, so the storage saving is trivial.
* **Denormalize.** Put attributes from related entities (for example, brand and category for a product) into the same table, and do not split them into separate tables (snowflaking).
* Benefits of denormalization:
  * Simple, fast queries with fewer joins.
  * Easy to understand: users see one table for each business concept.
  * Efficient filtering and grouping on a single table.
* Include **multiple versions of a label** where needed (short name, long name, code).
* Provide **sort-order** columns where alphabetical sorting is not natural (for example, month number for month name, or size order for "Small, Medium, Large").
* Where a numeric value is used mainly to filter or group (such as a standard list price or an age), it may be an attribute (often with a **band**) rather than a fact, and sometimes you keep it in both places.

Example (simplified product dimension):

```sql
CREATE TABLE dim_product (
    product_key            INT           PRIMARY KEY,   -- surrogate key
    product_durable_key    INT           NOT NULL,
    sku_number             VARCHAR(20)   NOT NULL,      -- natural key
    product_name           VARCHAR(100)  NOT NULL,
    brand_name             VARCHAR(50)   NOT NULL,
    subcategory_name       VARCHAR(50)   NOT NULL,
    category_name          VARCHAR(50)   NOT NULL,
    department_name        VARCHAR(50)   NOT NULL,
    package_type           VARCHAR(30)   NOT NULL,
    package_size           VARCHAR(20)   NOT NULL,
    diet_type              VARCHAR(30)   NOT NULL,
    row_effective_date     DATE          NOT NULL,
    row_expiry_date        DATE          NOT NULL,
    is_current             CHAR(1)       NOT NULL
);
```

---

## 3. Attribute Quality: Decoding and Avoiding Cryptic Flags

Users see attribute values in report headings, filters, and labels, so every value must make sense on its own.

* **Replace codes with decoded descriptions.** The warehouse should present "Frozen Foods", not `FZ`. Keep the original code as a separate column if people use it.
* **Avoid cryptic flags and indicators** such as `Y/N`, `0/1`, or `T/F` as the only content. Use self-explanatory values.

| Poor | Better |
|---|---|
| `holiday_flag = 'Y'` | `holiday_indicator = 'Holiday'` / `'Non-holiday'` |
| `status = 3` | `order_status = 'Shipped'` |
| `wkend = 1` | `weekday_indicator = 'Weekend'` / `'Weekday'` |
| `gender = 'U'` | `gender = 'Unknown'` |

* **Standardize and clean the values.** Consistent capitalization, spelling, abbreviations, and formats ("Street" vs. "St." vs. "Str"), with duplicates and typos resolved in ETL.
* **Make the content consistent across sources.** If two systems code the same concept differently, map both to one standard description.
* **Do not leave nulls or blanks** (see Section 17).
* **Document each attribute**: its definition, its source, valid values, and how it is derived.
* **Name columns in business terms,** consistently across dimensions, since the same name should mean the same thing everywhere.

---

## 4. Fixed-Depth Hierarchies and Flattening

Many dimensions have natural hierarchies, such as product → brand → category → department, or date → month → quarter → year.

* In a **fixed-depth hierarchy**, each member has exactly one parent at each level, and there is a known number of levels.
* **Flatten** the hierarchy into the dimension: each row carries the attribute for every level, so higher-level values are repeated across all the rows below them.
* This supports **drill-down and roll-up** with no extra joins: users simply add or remove an attribute from the report.
* The relationship at each level must be **many-to-one** (each product belongs to one brand, and each brand to one category).

Example:

| product_name | brand_name | category_name | department_name |
|---|---|---|---|
| Crunchy Oat Cereal 500g | SunGrain | Cereal | Grocery |
| Crunchy Oat Cereal 1kg | SunGrain | Cereal | Grocery |
| Instant Noodles Masala | QuickBite | Noodles | Grocery |

Additional points:

* A dimension can contain **several hierarchies** at once (for example, product by brand and category, and also by package type), all as attributes of the same table.
* Hierarchies need not be perfectly clean. Users also slice by **non-hierarchical attributes** (for example, color or size).
* Name each level clearly and keep the level values consistent.
* **Variable-depth (ragged) hierarchies**, such as an organization chart, cannot be flattened this way and use bridge tables (see Module 6).
* Make level values **unambiguous**. A city name such as "Springfield" exists in several states, and a category such as "Other" can appear under several parents. Keep the parent attributes alongside the level, or make the label unique, so users do not unintentionally combine different entities when grouping on one level alone.
* Do not snowflake the hierarchy into separate tables for each level.

---

## 5. The Date/Calendar Dimension

Almost every fact table has a date dimension. It is built **in advance**, not derived from the fact tables, and is the one dimension that exists in virtually every dimensional model.

### Design

* **Grain:** one row per calendar day.
* **Size:** small. Ten to twenty years is only 3,650 to 7,300 rows, and it can be extended in advance.
* **Key:** the surrogate key may be a meaningful integer like `YYYYMMDD` (the standard exception to the "meaningless key" rule), or a sequential number. A meaningful key makes it easy to partition the fact table by date. It is still important to have a **separate full date column** and not to rely on parsing the key.
* **Pre-load** it with a script, and add attributes with business rules, so that the calculations are not repeated in every query.

### Typical attributes

| Group | Examples |
|---|---|
| Date | full date, date label, day of week name and number, day number in month/year |
| Week | week number in year, week start date, ISO week |
| Month | month name, month number, month-year label, last-day-of-month indicator |
| Quarter and year | quarter number, quarter label, calendar year |
| Fiscal | fiscal week, fiscal month, fiscal quarter, fiscal year |
| Indicators | weekday indicator, holiday indicator, holiday name, selling-season, special event |
| Relative | current-day, current-month, and current-year indicators, or offsets such as "months ago", refreshed by ETL every day |

* Use **descriptive text** for indicators ("Holiday"/"Non-Holiday"), and include **sort columns** for names.
* Do not rely on users or BI tools to calculate these attributes from a date column. Calculate them once in the dimension.

### Calendar vs. fiscal

* Keep **both** calendar and fiscal attributes in the same dimension, with clear names (for example, `calendar_year` and `fiscal_year`).
* If different parts of the business use different fiscal calendars (or retail calendars such as 4-4-5), either:
  * add a set of attributes for each calendar, or
  * create a separate date dimension per calendar, or one with a **calendar identifier** attribute, paired with a row for each calendar/date combination.
* Fiscal periods that don't align with calendar months (for example, 4-4-5 weeks) are a strong reason to keep the logic in the dimension.

### Holidays and special days

* Store a **holiday indicator** and the **holiday name** (and perhaps region-specific flags if the business operates in several countries).
* Holiday data is supplied from a business-maintained source, and is added to the dimension when the year is created.

### Unknown and not-applicable dates

* Fact tables must never have null date keys. Add **special rows** for "Unknown", "Not applicable", and "Not yet occurred" (also used for incomplete accumulating snapshot milestones).
* These rows usually have a distinct **key** and a **far-future date** (such as `9999-12-31`) in the date column, with descriptive text in all the other attributes.

Example generation (PostgreSQL syntax, simplified):

```sql
INSERT INTO dim_date (date_key, full_date, day_name, month_number, month_name,
                      calendar_quarter, calendar_year, weekday_indicator)
SELECT TO_CHAR(d, 'YYYYMMDD')::INT,
       d,
       TO_CHAR(d, 'FMDay'),
       EXTRACT(MONTH FROM d),
       TO_CHAR(d, 'FMMonth'),
       'Q' || EXTRACT(QUARTER FROM d),
       EXTRACT(YEAR FROM d),
       CASE WHEN EXTRACT(ISODOW FROM d) IN (6,7) THEN 'Weekend' ELSE 'Weekday' END
FROM   (SELECT g.d::date AS d
        FROM   generate_series('2015-01-01'::date, '2035-12-31'::date, '1 day') AS g(d)) AS days;
```

---

## 6. Time-of-Day Dimension vs. Timestamp Attributes

When analysis needs the time of day (for example, peak hours, shifts, or call-center loads):

* **Do not combine date and time into one dimension.** A row for every minute of every day for ten years would be over five million rows.
* Instead, use two parts:
  1. The **date dimension** (one row per day).
  2. A separate **time-of-day dimension** with **one row per minute** (1,440 rows) or **per second** (86,400 rows), with attributes such as hour, minute, AM/PM, day part (morning, afternoon, evening), shift, and peak/off-peak indicator.
* The fact table then has **two foreign keys**: `date_key` and `time_of_day_key`.
* Also store the **full date-time stamp** as a **column in the fact table**. This is needed for precise calculations (such as the time elapsed between events), which cannot be done accurately from the two dimension keys.

| Need | Use |
|---|---|
| Group or filter by hour, shift, or day part | Time-of-day dimension |
| Compute durations or exact sequencing | Timestamp column in the fact table |
| Only the date matters | Date dimension alone |

* If the business operates across time zones, decide whether to store local time, UTC, or both. See Module 6 (multiple time zones).

---

## 7. Role-Playing Dimensions

A **role-playing dimension** is a single physical dimension that is referenced **several times** by one fact table, with each foreign key representing a different role.

* Examples:
  * Date in an order fact table: order date, requested ship date, actual ship date, delivery date.
  * Airport in a flight fact table: origin airport and destination airport.
  * Employee: salesperson, approver, and handler.
* Implementation:
  * Keep **one physical table** (a single copy of the data to maintain).
  * Create a **separate view** (or alias) for each role, with **column names renamed** to include the role ("Order Date Month", "Ship Date Month"), so the roles are not confused in reports.
  * The fact table has **one foreign key column per role**, each pointing to the same dimension.
* Roles are independent. A report can filter by order date and group by ship date.

```sql
CREATE VIEW dim_order_date AS
SELECT date_key AS order_date_key, full_date AS order_date,
       month_name AS order_month, calendar_year AS order_year
FROM   dim_date;

CREATE VIEW dim_ship_date AS
SELECT date_key AS ship_date_key, full_date AS ship_date,
       month_name AS ship_month, calendar_year AS ship_year
FROM   dim_date;
```

---

## 8. Degenerate Dimensions

A **degenerate dimension** is a dimension key in the fact table that has **no corresponding dimension table**, because all its useful attributes have already been placed in other dimensions.

* Typical examples: order number, invoice number, receipt number, ticket number, and bill of lading number.
* They are:
  * **Kept in the fact table** as a column, usually next to the foreign keys.
  * Used to **group line items** into the parent transaction, and to link to source-system documents.
  * Often part of the fact table's **primary key** (for example, order number + line number).
* They are **not facts**. They are identifiers, not measurements to be summed.
* If the identifier **has its own descriptive attributes** (such as order type or sales channel), those attributes should go into a dimension (often a junk dimension), while the number can remain in the fact table.
* Common in transaction fact tables and header/line item designs.

---

## 9. Junk Dimensions

A **junk dimension** combines several unrelated, **low-cardinality** flags, indicators, and text attributes into a **single dimension**, in place of leaving them in the fact table or making a separate dimension for each.

### Problem it solves

A transaction may carry many small attributes, such as payment type, order channel, gift-wrap indicator, and rush-order flag. Options:

| Option | Problem |
|---|---|
| Leave in the fact table as text columns | The fact table becomes wide and bloated, with the same text repeated across millions or billions of rows |
| Create a dimension for each | Too many foreign keys, and many tiny tables |
| Drop them | Business loses useful analysis |
| **Junk dimension** | One foreign key, one small table |

### Design

* Each row represents a **unique combination** of the attribute values.
* Two ways to populate it:
  * **Cartesian product** of all possible values (suitable if the number of combinations is small, such as 2 × 3 × 4 = 24 rows).
  * **Only combinations that occur**, added by ETL when a new combination is first seen (suitable if the full product would be very large).
* The fact table holds one `junk_key` instead of many columns.
* Keep **related attributes together** and split into **several junk dimensions** if the number of combinations would otherwise be huge (for example, more than tens of thousands of rows), or if the attributes have different update patterns.
* Give each attribute a descriptive name, and descriptive values, as in any dimension.

Example:

| order_profile_key | payment_type | order_channel | gift_wrap | rush_order |
|---|---|---|---|---|
| 1 | Credit card | Web | Gift wrapped | Standard |
| 2 | Credit card | Web | No gift wrap | Rush |
| 3 | Cash on delivery | Phone | No gift wrap | Standard |

---

## 10. Conformed Dimensions

A **conformed dimension** is a dimension that has the **same meaning and content** wherever it is used, across different fact tables and data marts. Conformed dimensions are the **foundation of integration** in the Kimball architecture.

### What conformed means

* Either **identical** tables (same keys, attributes, definitions, and values), or one is a **subset** of the other (a shrunken dimension, see Section 11).
* The **attribute names, definitions, and values** must be consistent. A "Region" must mean the same thing for sales and for inventory.
* Conformance is judged at the **attribute level**: two dimensions conform when the attributes they share have the same column names and domain contents, even if one dimension has fewer attributes or rows (see Section 11). Dimensions that merely overlap, with similar but differently defined attributes, are **not** conformed, and drilling across them will mislead.
* Dimension keys and rows must be consistent so the same entity gets the same key across processes.

### Why they matter

* They enable **drill-across**: querying separate fact tables, grouped by the same dimension attributes, then merging the result sets. This gives "sales and inventory by product" without a single combined fact table.
* They give the enterprise **consistent labels and one version of the truth** for key business entities.
* They allow the warehouse to be built **incrementally**. New business processes reuse the existing conformed dimensions.
* They are the columns of the **enterprise bus matrix**.

### How to get them

* The business must agree on **common definitions**, led by **data stewards** and a governance process. This is as much an organizational task as a technical one.
* A central team usually builds and publishes the conformed dimension once (the **dimension manager**), and the other processes use the published copy.
* Conformed dimensions should also be **conformed in the semantic layer** and BI tools.
* When two sources describe the same entity differently, reconcile them in the ETL conforming step, and handle the differences by agreement (or by building a separate, non-conformed attribute).

---

## 11. Shrunken (Rollup) Dimensions

A **shrunken dimension** is a **conformed subset** of a base dimension, used when facts are captured at a **higher level of detail** than the base dimension.

There are three common forms:

1. **Rollup:** fewer rows, at a higher grain.
   * Example: a **Month** dimension derived from the Date dimension, for monthly budgets or forecasts. A **Brand** dimension derived from Product for facts captured only by brand.
2. **Attribute (column) subset:** the same rows, but only some attributes, because the fact table only relates to those.
   * Example: a Customer dimension with only a few columns for a specific mart, where those columns match the base definition exactly.
3. **Row subset:** the same grain as the base, but only some of its rows, for a fact table that covers only some members.
   * Example: a Product dimension limited to one product line for a subtype fact table, or an Employee dimension limited to sales representatives.

Rules:

* The attributes it contains must be **identical in name, definition, and values** to those in the base dimension. This keeps it conformed and allows drill-across.
* It usually has its **own surrogate key** (for example, a month key).
* It is also used for **aggregate fact tables**, where the aggregated fact table is linked to the shrunken dimensions.
* Create it from the base dimension by ETL, so it stays in sync.

Example:

* `fact_sales` (daily grain) joins to `dim_date`.
* `fact_sales_budget` (monthly grain) joins to `dim_month` (a shrunken dimension of `dim_date`).
* The report can drill across both tables by month, because `calendar_month` and `calendar_year` have the same values in both.

---

## 12. Outrigger and Snowflaked Dimensions

An **outrigger dimension** is a secondary dimension table that is **attached to another dimension** table (and not to the fact table), by a foreign key in the primary dimension.

* Examples:
  * A **Product** dimension with a "Product introduction date" key pointing to the **Date** dimension (the date acts as an outrigger).
  * A **Customer** dimension linked to a **Demographics** (county-level census data) table with dozens of attributes shared by many customers.
* It is a limited form of **snowflaking**.

Guidance:

* Kimball allows outriggers but recommends using them **sparingly**, because they:
  * Add joins and complexity.
  * Can slow queries.
  * Make the model harder for users to understand.
* Reasonable uses:
  * A **cluster of low-cardinality attributes** with its own natural grouping, reused by several rows or dimensions.
  * A **date** attribute of the dimension that users want to analyze by the full calendar (introduction date, first purchase date).
  * Where the outrigger's attributes come from a **different source or refresh cycle** than the main dimension (for example, externally supplied demographic data).
* Alternatives (usually preferred):
  * **Flatten** the attributes into the main dimension.
  * Attach the second dimension **directly to the fact table** instead of nesting it.
* Do not build an outrigger on top of another outrigger.
* Keep the outrigger hidden or clearly labelled in the BI tool, so users understand the relationship.

### Snowflaked dimensions

A **snowflaked dimension** has its attributes **normalized into several related tables** (for example, product, brand, and category as three tables, each joined to the next). An outrigger is a deliberate, limited attachment for one justified case, and snowflaking is normalization applied across the whole dimension.

* Kimball generally advises **against** snowflaking in the presentation layer. Users face a harder model, queries need more joins, browsing the values of one dimension becomes slower and more complex, and the storage saved is trivial compared with the fact table.
* Snowflaked structures are normal in source systems and ETL staging, and some BI or OLAP tools expect them.
* If you inherit a snowflake, you can often **hide it behind a view** that joins the tables into one flat dimension.

---

## 13. Mini-Dimensions

A **mini-dimension** is a separate dimension created to hold **rapidly changing attributes** of a large dimension, so those attributes are not tracked in the main dimension with Type 2 history.

### Problem

* A **large dimension** (such as Customer with millions of rows) has some attributes that **change often**, for example credit score, income band, or number of purchases.
* Using Type 2 for these would create a **new row for every change**, making the dimension grow explosively.

### Solution

* Move the volatile attributes to a **new, small dimension**, and:
  * **Band the values** into ranges (income 50-75K, score 700-749), so that the number of combinations stays small (typically a few hundred to thousands of rows).
  * Each row of the mini-dimension is a **unique combination** of the banded attribute values.
  * Add the **mini-dimension's foreign key to the fact table**, alongside the customer key.
* The fact table row records the customer **and** the customer's profile **at the time of the event**. The relationship over time is preserved by the fact table, so the main dimension doesn't need Type 2 rows for these changes.

### Notes

* Bands should be chosen with the business, because users lose the precise values in the banded attributes. If the exact value matters, keep it as a Type 1 attribute or as a fact where it will not multiply rows.
* The link between a customer and the profile **exists only through fact rows**. If there is no event (no fact row), there is no record of the profile. There are two common fixes: a separate **relationship table** (often a factless table) recording which customer had which profile and when, or a **Type 5** design, which also adds the **current** mini-dimension key to the main dimension as a Type 1 attribute. The mini-dimension itself is the **Type 4** technique.
* The mini-dimension can also be used for **demographic** and **behavior** profiles, and in other large dimensions.
* More detail on the SCD types is in Module 5.

---

## 14. Swappable Dimensions

**Swappable dimensions** (sometimes called hot-swappable dimensions) are **alternative versions of a dimension table** that can be used with the **same fact table**, so different groups of users can see the same facts through different dimensional perspectives.

* The alternatives share the **same keys and grain**, and can differ in:
  * **Attributes** (a subset or a different set of attributes for each user group, or different levels of detail).
  * **Values or labels** (for example, localized language versions, or different naming conventions).
  * **Hierarchies and groupings** (for example, an alternative customer segmentation or product classification).
  * **Access** (a restricted version without sensitive attributes).
* They are implemented as **views or separate physical tables** that use the same surrogate keys, and the BI tool or user chooses which one to use.

Cautions:

* Because the alternatives share keys, the fact table doesn't change, which keeps the design simple.
* Drill-across and enterprise consistency depend on **conformance**. A swappable version must not redefine conformed attributes in a conflicting way.
* Maintain the alternatives in **ETL** (or as views) so they stay in sync with the base dimension.
* Kimball lists *hot swappable dimensions* among its advanced dimension techniques. The description above is the general idea, so check *The Data Warehouse Toolkit* for the full treatment.

---

## 15. Heterogeneous Supertype/Subtype Dimensions (and an Anti-Pattern)

Some businesses (banks, insurers, telecom, and retailers with very different product lines) have entities of **different types**, each with attributes that exist only for that type.

Example: an insurance policy, a bank account, or a product that can be a physical item, a service, or a subscription.

Options:

| Approach | When to use |
|---|---|
| **Single dimension with all attributes** | Few types, few type-specific attributes. Irrelevant attributes are filled with "Not applicable" |
| **Supertype dimension + subtype dimensions** | Many types, many type-specific attributes |

The supertype/subtype approach:

* **Supertype dimension:** contains the **attributes common to all types** and one row per entity. It is used with the supertype fact table, for analysis across all types.
* **Subtype dimensions:** one per type, with the **extra attributes** unique to that type, and the same key as the supertype. They are used with subtype fact tables for detailed analysis within a type.
* The dimension design mirrors the fact table design (see Module 3, Section 11).

Benefits:

* Avoids wide tables with hundreds of mostly empty columns.
* Allows cross-type analysis and type-specific analysis at the same time.

### Anti-pattern: abstract generic dimensions

An **abstract generic dimension** is one table meant to hold **every kind of entity** (customers, products, locations, employees) using generic columns such as `type`, `name`, `description`, and `attribute_1` to `attribute_n`, or a key-value structure.

* It looks flexible, because new entity types need no new tables.
* Kimball advises against it. Attributes lose their business meaning, the table is full of columns that apply to only some types, queries are hard to write and slow, values cannot be consistently constrained, and users cannot browse it naturally.
* Build **separate, business-specific dimensions** for each entity, and use the supertype/subtype approach where types share attributes.
* Generic structures can be acceptable in a source system or staging area, but not in the presentation area.

---

## 16. Dimension-to-Dimension Relationships

In a dimensional model, **dimensions normally relate to each other through the fact table**, not by direct links between dimension tables.

When you need to represent a relationship between two dimensions (for example, customer and sales representative, or store and region manager):

| Situation | Approach |
|---|---|
| The relationship exists because of events (sales) | It is recorded in the **fact table**. No separate structure is required |
| The relationship is **stable** and descriptive | Copy the attributes into the dimension (**denormalize**), or add an **outrigger** |
| The relationship **changes over time** and has no event | Use a **factless fact table** (such as customer-to-representative assignment by date) |
| The relationship is **many-to-many** | Use a **bridge table** (see Module 6) |

Guidelines:

* Prefer **denormalizing** a simple one-to-many relationship (for example, the sales representative's name in the customer dimension) when the attribute is stable, or when it is acceptable to use the Type 1 or Type 2 approach.
* Avoid chains of foreign keys between dimensions, because they produce a snowflake schema.
* Be careful when the relationship changes: decide whether history of the relationship matters, and choose the SCD type or the factless table accordingly.

---

## 17. Unknown, Not Applicable, and Null Values

Dimension tables should be free of nulls and blank cells, because nulls cause confusing report output and inconsistent filtering.

### In dimension attributes

* Replace nulls with a **descriptive value** explaining the absence:
  * **Unknown:** the value exists but was not captured.
  * **Not Applicable:** the attribute does not apply to this row.
  * **Not Provided / Missing:** the source did not supply it.
* Use consistent wording throughout the warehouse.
* For hierarchies, where a level does not apply, either repeat the parent value or use "Not Applicable".
* Numeric attributes are harder, and practice varies between teams. Avoid **magic numbers** (such as -999 or 0) that can be mistaken for real values and distort calculations. If a numeric attribute can be missing, keep it null, but expose a **text version or band** with an "Unknown" label for reporting.
* Handle **mixed-case, trailing spaces, and empty strings** in ETL, so they are not mistaken for valid values.

### Default dimension rows

* Every dimension should have default rows with **reserved surrogate keys** (for example, `-1` for Unknown, `-2` for Not Applicable, or `0`), loaded at creation.
* The fact table points to these rows when the real foreign key is missing, not applicable, or invalid (see Module 3, Section 5).
* Each default row has descriptive text in **every** attribute column.
* The same convention should be used in **all** dimensions.

| customer_key | customer_name | city | segment |
|---|---|---|---|
| -1 | Unknown | Unknown | Unknown |
| -2 | Not Applicable | Not Applicable | Not Applicable |

---

## 18. Audit Dimension

The **audit dimension** captures **metadata about how and when each fact row was created**, and the **quality of the data**, so that users and ETL teams can trace and filter the data. It is created by the ETL process (the audit dimension assembler is one of the Kimball ETL subsystems).

* **Grain:** typically one row per ETL batch/run for a given fact table, or per unique combination of audit attributes.
* The fact table has an **audit key** foreign key to this dimension.
* Typical attributes:

| Group | Examples |
|---|---|
| Lineage | ETL batch ID, job name, ETL version, source system, extract date |
| Timing | Load start and end timestamps, processing date |
| Data quality | Overall quality indicator (OK, Warning, Error), number of rows with quality events, data screen results |
| Treatment | Flags such as "Imputed value", "Estimated", "Late-arriving", "Corrected" |
| Business rules | Version of the allocation rule, currency rate version, calculation method |

* Benefits:
  * **Lineage:** shows where each fact row came from and which process loaded it.
  * **Quality filtering:** lets users **include or exclude** low-quality or estimated rows, or understand how much of a total is based on questionable data.
  * **Troubleshooting and reload:** allows ETL teams to find and replace the rows from a failed or incorrect batch.
  * **Compliance and trust:** supports reconciliation and audits.
* Dimension tables can carry their own audit columns (source, batch ID, load date), without needing a separate audit dimension.
* The audit dimension is usually **hidden** from general users, and exposed to analysts, stewards, and ETL developers.

---

## 19. Common Mistakes

* Using the source natural key as the primary key of a dimension.
* Skipping the durable key when natural keys are unstable.
* Leaving codes undecoded, or cryptic flags like `Y/N` as the only value.
* Snowflaking hierarchies into separate tables without a strong reason.
* Allowing nulls and blanks in attributes, or having no default "Unknown" rows.
* Building the date dimension from the fact table dates, or leaving out fiscal attributes, holidays, and a "Not applicable" date row.
* Combining date and time into one huge dimension.
* Creating a physically separate copy of the date table for each role, instead of using views.
* Placing the order number into a dimension table with no attributes.
* Putting many flags in the fact table, or creating many tiny dimensions, instead of a junk dimension.
* Using different definitions of the "same" dimension in different marts (non-conformed).
* Creating a shrunken dimension with attributes whose values differ from the base dimension.
* Overusing outriggers, or chaining them.
* Tracking rapidly changing attributes in a huge dimension with Type 2, instead of a mini-dimension.
* Using one wide dimension with hundreds of empty columns, instead of supertype/subtype dimensions.
* Creating an abstract, generic dimension for all entity types.
* Using ambiguous level values (the same city name in different states) in flattened hierarchies.
* Using the same attribute name in different marts for differently defined content.
* Omitting any audit information and quality indicators.

---

## 20. Self-Check Questions

1. Why does a dimension use a surrogate key rather than the source system key?
2. What is a durable key, and how does it differ from a surrogate key and a natural key?
3. Why is denormalization acceptable in dimension tables?
4. How would you improve a column that stores `Y`/`N` and a status code `3`?
5. How do you model a fixed-depth hierarchy, and why are the relationships many-to-one?
6. What are the main attributes of a date dimension, and how do you handle fiscal calendars and holidays?
7. How do you represent a date that is unknown or has not yet occurred?
8. Why separate the date and time-of-day dimensions, and why also keep a full timestamp in the fact table?
9. What is a role-playing dimension, and how is it implemented?
10. What is a degenerate dimension, and why is it not a fact?
11. What problem does a junk dimension solve, and how do you populate it?
12. What makes a dimension conformed, and why is it needed for drill-across?
13. What is a shrunken dimension, and when would you build one?
14. What is an outrigger, and why should it be used sparingly?
15. What problem does a mini-dimension solve, and what is its limitation?
16. What are swappable dimensions used for?
17. When would you use supertype and subtype dimensions?
18. How do you represent a relationship between two dimensions?
19. How do you handle missing values in a dimension attribute and in a fact table foreign key?
20. What does an audit dimension contain, and how do users and ETL teams use it?
21. What is the grain of a dimension table, and why must a dimension supply exactly one row per fact row?
22. What are the three forms of shrunken dimension, and what must they share with the base dimension?
23. What is the difference between an outrigger and a snowflaked dimension, and why is snowflaking discouraged in the presentation layer?
24. Why is an abstract generic dimension a poor design, and what should you build instead?

---

## Quick Reference

| Pattern | Use it when |
|---|---|
| Surrogate key | Always, for every dimension (date may use a meaningful key) |
| Durable key | The natural key can change, be reassigned, or merge across systems |
| Flattened hierarchy | Fixed-depth, many-to-one levels |
| Date dimension | Every fact table with a date, built in advance |
| Time-of-day dimension | Analysis by hour, shift, or day part |
| Role-playing | The same dimension appears several times in one fact table |
| Degenerate | The fact row has an identifier with no other attributes |
| Junk | Many low-cardinality flags and indicators |
| Conformed | The same entity is used across several processes |
| Shrunken | Facts at a higher level than the base dimension, or aggregates |
| Outrigger | A stable cluster of attributes, or a date attribute, used sparingly |
| Mini-dimension | A large dimension with rapidly changing attributes |
| Swappable | Different user groups need different views of the same dimension |
| Supertype/subtype | Entity types with very different attributes |
| Default rows | Missing, unknown, or inapplicable foreign keys |
| Audit dimension | Lineage and data quality of each fact row |
| Snowflaked dimension | Avoid in the presentation layer; flatten it, or hide it behind a view |
| Abstract generic dimension | Avoid; build separate, business-specific dimensions |
