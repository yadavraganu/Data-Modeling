# Foundations of Kimball Dimensional Modelling

This module covers the ideas that everything else in the Kimball approach builds on: why data warehouses and BI systems exist, how the Kimball architecture is organized, how dimensional models differ from normalized models, and how a DW/BI project is planned and run.

## Contents

1. Purpose of DW/BI and the Kimball, Inmon, and Data Vault perspectives
2. The Kimball DW/BI architecture
3. Dimensional modelling vs. 3NF/ER modelling
4. Star schema, snowflake schema, and OLAP cubes
5. Business process orientation
6. The Kimball Lifecycle
7. Requirements gathering
8. Common myths and misconceptions
9. Self-check questions

---

## 1. Purpose of DW/BI and the Kimball, Inmon, and Data Vault Perspectives

### Why DW/BI exists

Operational systems (order entry, billing, HR, and so on) are built to **record transactions** quickly and reliably, one at a time. They are poor at answering questions that span many transactions, many systems, and long periods of time. A data warehouse/business intelligence (DW/BI) system exists to **support decision making** by delivering consistent, integrated, historical information in a form business people can use.

Kimball's stated goals for a DW/BI system are that it should:

* Make information **easily accessible**: the content is understandable, labelled in business terms, and fast to query.
* Present information **consistently**: data is credible, from different sources is reconciled, and the same name always means the same thing.
* Be **adaptive to change**: new questions, new data, and new business rules should not invalidate existing data or applications.
* Present information in a **timely** way: raw data is turned into usable information within the time the business needs.
* Be a **secure bastion** that protects information assets.
* Serve as the **authoritative and trustworthy foundation** for improved decision making.
* Be **accepted by the business community**. If the business does not use it, it has failed, however good the technology is.

The DW/BI system is not just a database. It is a combination of data, ETL processes, and BI applications.

### Three major perspectives

| | Kimball (dimensional) | Inmon (Corporate Information Factory) | Data Vault (Linstedt) |
|---|---|---|---|
| **Core idea** | The warehouse is the union of dimensional data marts built around business processes and tied together by conformed dimensions | A single, normalized (3NF) enterprise data warehouse is the integrated source; departmental data marts are derived from it | A scalable, auditable integration layer of hubs, links, and satellites that tracks all data and its history |
| **Central data structure** | Star schemas with atomic-grain fact tables | 3NF enterprise warehouse, then dimensional marts | Hubs (business keys), links (relationships), satellites (descriptive history) |
| **Who queries it** | Business users and BI tools directly | Usually the data marts, not the EDW | Usually downstream marts (often dimensional); the vault itself is not designed for end users |
| **Typical build approach** | Incremental, one business process at a time, guided by the bus matrix | More top-down: design the enterprise model, then build marts | Incremental and highly modular; raw vault first, business vault and marts on top |
| **Main strengths** | Business usability, query performance, fast delivery of value | Strong enterprise-wide integration and normalization | Agility, parallel loading, full auditability, and resilience to source change |
| **Typical criticisms** | Needs disciplined conformance work to avoid stovepipes | Heavier up-front effort; value is delayed | More tables and joins, and not directly consumable by business users |

**Important points to remember:**

* These are architectural and philosophical approaches, not strict rivals. Many organizations use a hybrid, such as a Data Vault or a normalized layer for integration and history, with **Kimball-style dimensional marts as the presentation layer**.
* In both Kimball and Inmon approaches, the data that business users query ends up dimensional. The disagreement is mainly about whether a normalized enterprise warehouse must come first and what role it plays.
* The old "bottom-up vs. top-down" labels oversimplify. Kimball's approach is top-down in planning (an enterprise bus matrix and shared conformed dimensions) and incremental in delivery.

---

## 2. The Kimball DW/BI Architecture

The architecture has four separate and distinct components, usually described as a restaurant analogy: the kitchen (back room) and the dining room (front room).

```
Source Systems  -->  ETL System  -->  Presentation Area  -->  BI Applications
(operational)       (back room)       (dimensional models)    (front room)
```

### 2.1 Operational source systems

* Capture business transactions (ERP, CRM, POS, web logs, files, external data).
* Optimized for transaction processing and availability, not for analytics.
* The DW/BI team has little control over their content and format, so the warehouse must be designed around what they can actually provide (see the data realities audit in Section 7).

### 2.2 ETL system (the "kitchen")

The extract, transform, and load (ETL) system is everything between the source systems and the presentation area.

* **Extract:** read and copy data from sources into the warehouse environment.
* **Transform:** clean, correct, deduplicate, conform, and combine data from different sources.
* **Load:** deliver data into dimensional structures, building dimension tables first and then fact tables.
* Includes the **staging area**, which may be flat files or relational tables. Staging is **off-limits to business users**; its data is not cleaned, conformed, or presented for querying.
* Kimball identifies 34 ETL subsystems, covered in a later module.

### 2.3 Presentation area (the "dining room")

This is where data is organized, stored, and made available for direct querying by users, report writers, and BI applications. Kimball's requirements for this area are:

* The data is presented, stored, and accessed in **dimensional schemas** (relational stars or multidimensional OLAP cubes).
* The presentation area contains **atomic-level detail**, not just summaries. Aggregates can be added for performance, but they supplement atomic data and do not replace it.
* The data is organized by **business process** and follows the **enterprise data warehouse bus architecture**: data marts share **conformed dimensions** and **conformed facts**, so they can be used together.

The presentation area is not a set of isolated departmental data marts. It is a coordinated set of process-oriented marts.

### 2.4 Business intelligence applications (the "front room")

Everything business users touch to analyze data:

* Ad hoc query tools, standard reports, dashboards, and analytic applications.
* Data mining and predictive modelling tools.
* Operational BI and embedded analytics.
* A **semantic layer** or metadata that presents friendly names and consistent definitions.

### 2.5 Variations

* **Independent data marts:** each built separately without conformance. This leads to stovepipes and is a mistake Kimball explicitly warns against.
* **Hub-and-spoke (Inmon):** a normalized EDW feeds dimensional marts.
* **Hybrid:** a normalized or Data Vault integration layer feeds the Kimball presentation area.

---

## 3. Dimensional Modelling vs. 3NF/ER Modelling

Both use entities and relationships, but they have different goals.

| | Third normal form (ER) model | Dimensional model |
|---|---|---|
| **Primary goal** | Remove redundancy and protect transaction integrity | Understandability and query performance |
| **Structure** | Many tables, each about a single entity | Few tables: fact tables and denormalized dimension tables |
| **Typical user** | Applications and developers | Business users, analysts, BI tools |
| **Query complexity** | Many joins; hard to navigate without expert help | Simple, predictable star joins |
| **Redundancy** | Minimized | Deliberate in dimensions (denormalization) |
| **Suited to** | OLTP systems, and integration/staging layers | Analytical queries on the presentation layer |
| **Resilience to change** | Changes may break many tables or applications | New attributes, dimensions, and facts can be added without invalidating existing queries |

Key points:

* A dimensional model **contains the same information** as a normalized model, just packaged differently.
* A normalized ER model may have hundreds of tables for one business area. A dimensional model for the same area is much smaller and easier to explain.
* ER modelling is useful in the ETL staging area and in operational systems, but it is not what users should query.
* A 3NF model can be converted to a dimensional model by applying the four-step process: choose a business process, declare the grain, identify dimensions, and identify facts.

---

## 4. Star Schema, Snowflake Schema, and OLAP Cubes

### 4.1 Star schema

* A **fact table** in the center, holding the numeric measurements of a business process event at a declared grain.
* **Dimension tables** around it, each joined to the fact table by a single key, holding descriptive context.
* Called a "star" because the diagram looks like one.
* Dimensions are **denormalized**: related attributes (for example, product, brand, and category) sit together in one flat table.
* Benefits: simple to understand, few joins, predictable query patterns, and good performance on modern databases.

### 4.2 Snowflake schema

* A star in which dimension tables are **normalized** into multiple related tables (for example, Product, then Brand, then Category as separate tables).
* Reduces storage redundancy in dimensions, but this saving is usually insignificant compared with the size of the fact tables.
* Costs: more tables and joins, a more complicated model for users, and potentially slower queries.
* Kimball recommends **avoiding snowflaking by default**. It is acceptable in limited situations, such as outriggers for a clearly separate, reusable set of attributes, or when a BI tool requires it.

### 4.3 OLAP cubes

* **Online analytical processing (OLAP)** stores data in multidimensional structures optimized for fast slicing, dicing, and aggregation.
* Cubes are typically built **from** a dimensional (star) schema and have the same dimensions and measures.
* Types:
  * **ROLAP:** relational OLAP, using a star schema in a relational database.
  * **MOLAP:** multidimensional OLAP, using a dedicated cube engine.
  * **HOLAP:** a hybrid of the two.
* In Kimball's view, cubes are a **performance and analytic layer on top of** a relational dimensional model, not a replacement for it. Atomic data is kept in the star schema.

### 4.4 Quick comparison

| | Star schema | Snowflake schema | OLAP cube |
|---|---|---|---|
| Dimension structure | Flat, denormalized | Normalized into sub-tables | Dimensions with hierarchies and levels |
| Typical storage | Relational | Relational | Multidimensional engine (or in-memory) |
| Ease of use | High | Medium | High (for supported queries) |
| Typical role | Core presentation layer | Exception | Performance and analytics layer |

---

## 5. Business Process Orientation

* Build the warehouse around **business processes** (such as taking orders, shipping, billing, or handling claims), not around departments or individual reports.
* Processes produce the measurement events that become fact tables. Departments consume those measurements.
* Benefits of this orientation:
  * The same data serves many departments instead of being duplicated and defined differently by each.
  * Avoids inconsistent versions of "the truth" across departmental marts.
  * Matches how source systems capture data, which makes ETL more straightforward.
  * Allows incremental delivery: one process at a time, each adding value.
* Processes are connected by **conformed dimensions**, so measures from different processes can be compared in one report (drill-across).
* The **enterprise data warehouse bus matrix** documents this: rows are business processes and columns are the common dimensions.

Example: a retailer's sales, inventory, and purchasing processes all use the Date, Product, and Store dimensions. Because those dimensions are conformed, the retailer can compare sales and inventory by product in the same report.

---

## 6. The Kimball Lifecycle

The Kimball Lifecycle is the project methodology for delivering DW/BI systems. It is built on a few principles: **focus on the business**, **build an appropriate information infrastructure**, **deliver in manageable increments**, and **provide the full solution**, including both data and BI applications.

### Lifecycle stages

```
                    Project / Program Planning
                              |
                    Business Requirements Definition
                              |
       +----------------------+----------------------+
       |                      |                      |
   Technology Track        Data Track          BI Application Track
   - Technical             - Dimensional       - BI application
     architecture design     modelling           specification
   - Product selection     - Physical design   - BI application
     and installation      - ETL design and      development
                             development
       |                      |                      |
       +----------------------+----------------------+
                              |
                         Deployment
                              |
                    Maintenance and Growth

         Project management runs through all stages
```

| Stage | What happens |
|---|---|
| **Program/project planning** | Assess readiness (sponsorship, business motivation, feasibility), define scope, justify the effort, and plan resources and the schedule |
| **Business requirements definition** | Understand the business's key drivers, goals, and analytic needs; identify and prioritize business processes |
| **Technology track** | Design the technical architecture, then select and install products (database platform, ETL tool, BI tools) |
| **Data track** | Dimensional modelling, physical design, and ETL design and development |
| **BI application track** | Specify and build the reports, dashboards, and analytic applications users will actually use |
| **Deployment** | Bring the technology, data, and BI applications together, along with training, support, and communication |
| **Maintenance and growth** | Support users, manage the system, and plan the next iterations |
| **Project management** | Monitor status, track issues, manage scope, and communicate throughout |

Key ideas:

* The three tracks run **in parallel** after requirements and converge at deployment.
* Each pass through the lifecycle delivers **one iteration** (typically one or a few business processes), then the cycle repeats.
* Business requirements **drive** all downstream decisions: architecture, models, and BI applications.
* The lifecycle is a flexible roadmap, not a rigid waterfall, and can be combined with agile practices.

---

## 7. Requirements Gathering

Good requirements are the foundation of a successful dimensional model. The goal is to understand **what the business is trying to accomplish and how it measures it**, not to ask users for lists of data fields.

### 7.1 Sources of requirements

* **Business user interviews or workshops:** focus on their goals, decisions, performance measures, and current analytic frustrations.
* **Interviews with IT and source system experts:** what data exists and how reliable it is.
* **Existing reports and analyses:** what the business already uses, and what it works around.

### 7.2 Interview approach

* Interview **business managers and analysts**, not only executives or IT. Group them by department or function, and use pairs of interviewers (one leads and one documents).
* Ask about **responsibilities, objectives, and success measures**: how do you know whether you are doing well, and what do you track?
* Ask about **how they use data now**: what reports and analyses do they use, what takes too long, and what questions can't they answer?
* Ask about **decisions and actions**: what would you do differently with better information?
* Avoid asking, "What data do you want?" Users typically describe the reports they know, which limits the design.
* Document each session promptly, and confirm the understanding with the interviewee.

### 7.3 Facilitated workshops

* Combine business and IT participants to define and prioritize business processes and measures.
* Useful for validating that interview findings are consistent across groups, and for building consensus on priorities.

### 7.4 Data realities audit

* In parallel with business interviews, **profile the source data**: what is available, how complete and accurate it is, and how it is structured.
* Identifies whether the data needed to support a business requirement actually exists and can be obtained at the required grain.
* Kimball recommends balancing both sides: requirements driven by the business, constrained and informed by what the data can support. A common rule of thumb is that **two thirds of the effort is in understanding the business** and one third in understanding data realities.

### 7.5 Turning requirements into design inputs

* Summarize findings by **business process**, with the related questions, measures, and dimensions.
* Create the initial **enterprise bus matrix** (processes against dimensions).
* **Prioritize** processes using business value and feasibility (data readiness, complexity, sponsorship).
* Agree the scope of the first iteration with the sponsor and stakeholders.
* Document **success criteria**, so that you can show the result delivered what the business needed.

---

## 8. Common Myths and Misconceptions

| Myth | Reality |
|---|---|
| Dimensional models contain only summary data | They should contain **atomic**, detailed data. Summaries are added as aggregates for performance. |
| Dimensional models are only for predictable, departmental reporting | A properly designed model at the atomic grain supports unpredictable, ad hoc queries, and can span the enterprise through the bus architecture. |
| Dimensional models cannot scale | They are used for very large data volumes. Scale depends on the platform and design, not the modelling approach. |
| Dimensional models can't integrate data across the enterprise | Integration is achieved through **conformed dimensions and facts**. |
| Dimensional models are only for data marts, not an enterprise warehouse | In the Kimball approach, the enterprise warehouse is the union of the conformed, process-oriented dimensional models. |
| Normalized models are always more flexible | Normalized models are flexible for transactions, but dimensional models are designed to handle changes to analytic requirements gracefully. |
| Dimensional modelling is only for relational databases | Dimensional design applies to relational and multidimensional (OLAP) implementations, and to modern columnar and cloud platforms. |

---

## 9. Self-Check Questions

1. What are the four components of the Kimball DW/BI architecture, and what is the role of each?
2. Why is the staging area off-limits to business users?
3. What does Kimball require of the presentation area (atomic data, dimensional structure, bus architecture)?
4. How do the Kimball, Inmon, and Data Vault approaches differ, and how can they be combined?
5. Why is a dimensional model easier for business users than a 3NF model, and what is the cost of denormalizing dimensions?
6. When might snowflaking be acceptable, and why is it avoided by default?
7. What is the relationship between a star schema and an OLAP cube?
8. Why model business processes rather than departments?
9. Name the stages of the Kimball Lifecycle and explain which activities run in parallel.
10. What is a data realities audit, and why is it done alongside business requirements gathering?
11. Why should you avoid asking users "what data do you want"?
12. How is an enterprise bus matrix built from requirements?

---

## Key Terms

**DW/BI:** data warehouse/business intelligence system · **ETL:** extract, transform, load · **Staging area:** ETL work area, not accessible to users · **Presentation area:** where dimensional data is stored and queried · **Fact table:** table of numeric measurements at a declared grain · **Dimension table:** table of descriptive context · **Star schema:** fact table surrounded by denormalized dimensions · **Snowflake:** dimensions normalized into sub-tables · **OLAP cube:** multidimensional structure for analysis · **Conformed dimension:** a dimension shared across fact tables with consistent meaning · **Bus matrix:** grid of business processes against dimensions · **Data realities audit:** profiling source data to test what the business requirements can be built on · **Atomic grain:** the most detailed level of data captured
