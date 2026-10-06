# The Four-Step Dimensional Design Process

The Four-Step Dimensional Design Process is the core design method in Ralph Kimball's dimensional modelling approach (*The Data Warehouse Toolkit*). It is a framework for designing star schemas that are easy for business users to understand and fast to query.

The four steps are made in order, because each decision constrains the next: the process determines the grain, the grain determines the dimensions, and the grain plus the process determine the facts. In practice the work is iterative, and you may revisit earlier steps as you learn more from stakeholders and source data. The one firm rule is that the grain must be settled before dimensions and facts are finalized.

| Step | Decision | Guiding question |
|---|---|---|
| 1 | Business process | What is the business doing that we want to measure? |
| 2 | Grain | What does exactly one row in the fact table represent? |
| 3 | Dimensions | How do business users describe, filter, and group this event? |
| 4 | Facts | What numeric measurements does this event produce? |

## Step 1: Select the Business Process

Identify the operational business process you want to model. A business process is an activity the organization performs, usually named with a verb, that is supported by a source system and produces performance measurements. It is not a department or a single report.

* **Examples:** Retail checkout (sales), processing an insurance claim, registering a student for a class, invoicing a customer, shipping an order.
* **Why it matters:** Modelling the process, rather than a department's reports, lets the model answer many questions about that activity and lets different departments share the same data. A process-based model also avoids duplicated, inconsistent copies of the same data across departments.
* **How to choose:** Use business requirements (interviews and workshops) together with a data realities check of the source systems. A process is only a good candidate if the data to support it actually exists and is of usable quality. Prioritize by business value and feasibility.
* **Output:** Each process becomes a row in the enterprise **bus matrix**, which is the roadmap for building the warehouse one process at a time.

## Step 2: Declare the Grain

Define exactly what a single row in the fact table represents, in business terms. The grain declaration is the pivotal step in the process, and it must be made **before** choosing dimensions and facts, because it determines which of both are valid.

* **Examples:** "One row per individual item scanned at the checkout register," "one row per boarding pass scanned at the gate," "one row per account per day (end-of-day balance)."
* **Grain types:** Every grain falls into one of the three fact table types:
  * **Transaction:** one row per event at a point in time (the POS and boarding pass examples).
  * **Periodic snapshot:** one row per entity per regular period (the daily balance example).
  * **Accumulating snapshot:** one row per instance of a process with a defined start and end, updated as it moves through milestones (for example, one row per order, updated through order, ship, deliver, and pay).
* **Why it matters:** An ambiguous grain is the most common cause of faulty fact tables, including double counting, mixed levels of detail in one table, and incorrect totals.
* **Best practice:** Model at the lowest, most atomic grain available. Atomic data can be rolled up in any direction, so it withstands unexpected queries. Aggregated data limits the questions that can be answered. Summaries can be added later as aggregate tables, but detail cannot be recovered from a summary.
* **State the grain in words,** not as a list of dimensions. "One row per line item on a sales receipt" is a grain declaration; "date, store, and product" is not.

## Step 3: Identify the Dimensions

Dimensions provide the context around a business process event: the "who, what, where, when, why, and how." They hold the descriptive attributes used to filter, group, and label the facts.

* **Examples:** Date, Product, Customer, Store, Employee, Promotion.
* **Rule of thumb:** Each dimension must take exactly **one value** for each fact row at the declared grain. If a dimension can have several values for one fact row (for example, multiple diagnoses on one hospital visit), the grain, the dimension, or a bridge table design must be revisited.
* **Design points:**
  * Every dimension is joined to the fact table by a **surrogate key**; the source system's natural/business key is kept as an attribute.
  * Almost every fact table has a **Date** dimension.
  * Attributes should be descriptive text, not cryptic codes, and are denormalized into one flat table per dimension.
  * The same dimension can play several roles in one fact table (for example, order date, ship date, and delivery date), which is known as a **role-playing dimension**.
  * Identifiers with no attributes of their own, such as an order number or invoice number, are stored in the fact table as **degenerate dimensions**.
  * Dimensions shared across processes should be **conformed** so that results from different fact tables can be compared.
* **Why it matters:** Rich, well-labelled dimensions are what make a dimensional model usable. If the grain is declared correctly in Step 2, identifying the dimensions is a matter of asking how the business will want to slice and dice the data.

## Step 4: Identify the Facts

Facts are the numeric measurements produced by the business process event. The question to ask is "what is the process measuring?"

* **Examples:** Quantity sold, sales amount, discount amount, tax amount, cost amount, transaction duration.
* **Additivity:** Classify each fact:
  * **Additive:** can be summed across all dimensions (quantity sold, sales amount).
  * **Semi-additive:** can be summed across some dimensions but not time (account balance, inventory on hand). These are typical in periodic snapshots and are usually averaged or taken at a point in time across the time dimension.
  * **Non-additive:** cannot be meaningfully summed (unit price, ratios, percentages). Store the additive components (for example, numerator and denominator) and calculate the ratio in the reporting layer.
* **Grain rule:** **Every fact in the table must be true to the grain declared in Step 2.** A metric at a different level of detail (for example, a monthly total or an order-level amount in a line-item table) belongs in a separate fact table, or must be allocated down to the grain.
* **Facts versus KPIs:** Facts are the raw measurements. KPIs are usually calculated from facts, often by combining them in the BI or semantic layer.
* **Special cases:** A process that records events but has no numeric measurements is modelled as a **factless fact table**, and a count of rows is then the measure.

## Worked Example: Retail Point-of-Sale

| Step | Decision |
|---|---|
| 1. Process | Retail sales at the checkout register |
| 2. Grain | One row per product line item on a sales receipt |
| 3. Dimensions | Date, Product, Store, Promotion, Cashier; receipt number as a degenerate dimension |
| 4. Facts | Sales quantity, regular unit price, discount amount, net sales amount, cost amount |

## Validation Checklist

Ask these four questions in order to check the model:

1. **Process:** What is the business doing, and is the data available?
2. **Grain:** What does exactly one row represent, and is it as atomic as possible?
3. **Dimensions:** Does each dimension have a single value per fact row, and how will users filter and group the data?
4. **Facts:** Is every measure true to the grain, and is its additivity understood?

## Common Pitfalls

* Designing around a department or a specific report instead of a business process.
* Leaving the grain vague, or mixing several grains in one fact table.
* Storing only summarized data and discarding the atomic detail.
* Putting facts of a different grain in the same fact table.
* Storing ratios or percentages as facts instead of their additive components.
* Allowing a dimension to hold more than one value per fact row without a deliberate design (such as a bridge table).
* Choosing a process for which source data does not exist or is unreliable.
