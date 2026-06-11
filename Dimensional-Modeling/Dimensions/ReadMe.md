### What are Dimensions?
Dimensions are the contextual elements in a dimensional model that provide the "who, what, where, when, why, and how" for business measurements (facts).
Dimensions are attributes that describe facts. They provide context and meaning to the numeric measurements stored in fact tables.

**Examples of Dimensions:**
- **Time Dimension** - dates, months, quarters, years
- **Customer Dimension** - customer names, segments, geography
- **Product Dimension** - product names, categories, pricing
- **Location Dimension** - stores, regions, countries

### Dimension Characteristics

| Characteristic | Description |
|---|---|
| **Data Type** | Descriptive, textual, categorical |
| **Cardinality** | Can range from low to high |
| **Relationships** | Link to fact tables via foreign keys |
| **Denormalization** | Often denormalized for query performance |
| **Stability** | Relatively stable; changes tracked via slowly changing dimensions |
