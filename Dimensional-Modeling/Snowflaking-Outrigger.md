## Snowflaking
Snowflaking is the process of normalizing dimension tables by splitting low-cardinality attributes into separate lookup tables.

* **Example**: A product dimension links to a separate category dimension instead of embedding category names directly.
* **When to prefer**:
    - Use when dimension tables are extremely large and storage space is at a premium.
    - Use when attributes are shared across multiple dimensions to enforce data consistency.
    - Use when you need to strictly enforce data hierarchy and normalization rules.
* **When not to prefer**: Avoid in standard data warehouses where user query performance is the top priority. Do not use when business users write ad-hoc SQL, as extra joins cause confusion. Avoid if your database engine handles wide, flat tables efficiently.
* **Best practices**: Keep dimensions star-schema compliant by default. Limit snowflake chains to one level deep. Use database views to flatten tables for end-users.
<br></br>
## Outrigger Dimension
An outrigger is a dimension table that joins to another dimension table instead of joining directly to the fact table.

* **Example**: A customer dimension links to a standard date dimension to track the customer's initial sign-up date.
* **When to prefer**:
    - Use when a high-cardinality attribute has its own complex set of details that change at a different rate.
    - Use to reuse an existing, standard dimension (like a Date dimension) inside another dimension (like an Account Creation Date).
    - Use when it significantly reduces data redundancy without over-complicating the overall schema.
* **When not to prefer**: Do not use if the attributes can be easily flattened into the main dimension table. Avoid if it creates deep, nested chains of dimensions that degrade query performance. Do not use if users expect a simple, single-join path to all descriptive attributes.
* **Best practices**: Reserve outriggers for universally standard dimensions. Align slowly changing dimension histories between tables. Clearly label foreign keys to make user join paths obvious.
<br></br>
## Centipede Fact Table
A centipede fact table is an anti-pattern where a fact table has too many foreign keys joining to tiny, single-attribute dimensions.

* **Example**: A sales fact table links to 40 separate lookup tables for age group, gender, and payment method flags.
* **When to prefer**: Generally, you should never prefer this design, as it represents poor dimensional modeling. The only exception is when the database technology strictly requires highly normalized structures for specific index compression.
* **When not to prefer**: Avoid in almost all standard data warehousing scenarios. Do not use because it creates a "centipede" of joins that severely slows down query execution. Do not use when you can instead group those related attributes into a single junk dimension or flat dimension.
* **Best practices**: Combine low-cardinality indicators into a single junk dimension. Keep total foreign keys in a fact table under 15–20. Leave unique operational IDs directly in the fact table as degenerate dimensions.
