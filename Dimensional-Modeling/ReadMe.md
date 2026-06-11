# Dimensional Modeling

The dimensional model of a business process is made up of two components: measurements and their context, known as facts and dimensions. These two simple concepts—measurement and context—are the foundation of dimensional design.

Implemented in a relational database, the dimensional model is called a **star schema**. Implemented in a multidimensional database, it is known as a **cube**.

## Information Systems Categories

Information systems fall into two major categories:
- Those that **support the execution of business processes** (Operational Systems)
- Those that support the **analysis of business processes** (Analytical Systems)

An **operational system** supports the execution of a business process, while an **analytic system** supports the evaluation of the process. An operational system manages day-to-day business transactions, while an analytical system evaluates historical data to drive long-term strategic decisions.

## Comparison: Operational vs Analytical Systems

| Feature | Operational System | Analytical System |
|---|---|---|
| **Core Purpose** | Run everyday business transactions smoothly. | Support strategic business decision-making. |
| **Technology Class** | OLTP (Online Transaction Processing). | OLAP (Online Analytical Processing). |
| **Primary Users** | Frontline staff, automated systems, customers. | Executives, data analysts, data scientists. |
| **Data Nature** | Current, granular, dynamic, volatile real-time data. | Historical, aggregated, stable, summarized data. |
| **Data Structure** | Highly normalized to eliminate data redundancy. | Highly denormalized for rapid text searching. |
| **Storage Structure** | Row-oriented (optimized for frequent writes). | Columnar storage (optimized for scanning rows). |
| **Operation Focus** | Fast inserts, updates, and deletes. | Complex reads and bulk data aggregations. |
| **Query Speed** | Milliseconds (low latency, high concurrency). | Seconds to hours (high data volume processing). |
| **Data Sources** | Direct application entries, forms, purchases. | Extracted operational data via ETL pipelines. |
| **Common Platforms** | PostgreSQL, MySQL, Oracle DB, SQL Server. | Snowflake, Google BigQuery, Amazon Redshift. |
| **Core Metric** | Transaction speed and total system uptime. | Query response time and insight accuracy. |
| **Business Value** | Keeps business operations running smoothly. | Fuels long-term competitive business growth. |
