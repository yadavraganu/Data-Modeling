
---

### 1. Transaction Fact Tables

#### Description

The transaction fact table is the most common and fundamental type of fact table. A single row in this table corresponds to a **discrete space-and-time event** or transaction in the physical world (e.g., a customer scans an item at a register, a user clicks a link, or a payment is processed).

* **Grain:** The most atomic level possible (e.g., "One row per invoice line item" or "One row per ATM withdrawal").
* **Facts:** The measurements are typically fully **additive**, meaning they can be summed up across any and all dimensions in the schema (e.g., `quantity_sold`, `extended_amount`).

#### When to Use It

* When you need to track highly detailed, discrete business events.
* When users need the ability to drill down to the absolute lowest level of operational detail.
* When your business processes naturally produce a steady, linear stream of individual transactions.

---

### 2. Periodic Snapshot Fact Tables

#### Description

Instead of capturing individual events as they happen, a periodic snapshot table captures the **cumulative state of a business process at uniform, regular intervals** (such as daily, weekly, or monthly).

* **Grain:** "One row per period, per entity" (e.g., "One row per account per month" or "One row per store product per day").
* **Facts:** The measurements are typically **semi-additive**. For example, you can safely sum account balances across the `Customer` or `Branch` dimensions, but you cannot sum them across the `Time` dimension (adding your bank balance from January and February together yields a meaningless number).

#### When to Use It

* When you need to monitor performance, trends, or statuses over time (e.g., inventory levels, bank account balances, or monthly sales quotas).
* When query performance would suffer if you tried to calculate running totals on billions of raw rows from a transaction fact table.

---

### 3. Accumulating Snapshot Fact Tables

#### Description

An accumulating snapshot fact table is designed to track a **predictable workflow or lifecycle** that has a well-defined beginning, middle, and end. Instead of inserting a new row for every event, a single row is created when the process starts, and that same row is **updated** as the process passes through multiple milestones.

* **Grain:** "One row per entire lifecycle instance" (e.g., "One row per order fulfillment lifecycle" or "One row per insurance claim").
* **Structure:** It features multiple date foreign keys corresponding to each milestone in the process, alongside duration or lag facts (e.g., `days_between_order_and_shipment`).

#### When to Use It

* When you are analyzing the operational efficiency, throughput, or cycle time of a specific workflow (e.g., order fulfillment, hospital patient admissions, or student university applications).
* When you need to see the current state of a multi-step pipeline at a glance.

---

### 4. Factless Fact Tables

This is a table that records an occurrence or a relationship but contains **no numeric measures (facts)**—only foreign keys.

There are two main types:

1. **Event/Coverage Tables:** Records that an event happened (e.g., "Student attended class on X date"). The "fact" is implicitly just a count of the rows.
2. **Eligibility/Conditions Tables:** Maps what *could* have happened (e.g., "Product X was on promotion in Store Y on Date Z"). This is crucial for calculating what did *not* happen (e.g., finding which promoted products had zero sales).

---

### Comparison

| Feature | Transaction Fact Table | Periodic Snapshot Fact Table | Accumulating Snapshot Fact Table |
| --- | --- | --- | --- |
| **Temporal Concept** | Point in time (discrete event) | Regular, recurring intervals (day/month) | Indeterminate timespan (lifespan of a process) |
| **Grain** | One row per transaction/line | One row per interval per entity | One row per pipeline instance |
| **Row Insertion Strategy** | Insert only | Insert only (at end of period) | Insert on start, **Update** on milestone completion |
| **Table Size** | Very deep (billions of rows, narrow) | Dense and large (grows deterministically) | Highly compact (one row per lifecycle instance) |
| **Date Dimensions** | Usually 1 or 2 date foreign keys | 1 date foreign key (the snapshot date) | **Multiple** date keys (one for each milestone) |
| **Fact Additivity** | Fully Additive | Semi-Additive (cannot sum over time) | Non-additive or semi-additive (often durations/lags) |
| **Core Analytic Value** | Maximum drill-down detail | Trend lines and historical balances | Workflow bottlenecks and velocity tracking |

---
