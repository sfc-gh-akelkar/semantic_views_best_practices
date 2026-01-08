# Snowflake Semantic Views: Consolidated Best Practices Guide

> **Prepared by: Snowflake Solution Engineering**  
> A comprehensive guide for data engineering and data science professionals building semantic views on Snowflake.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Getting Started](#getting-started)
3. [Best Practices for Creating Semantic Views](#best-practices-for-creating-semantic-views)
4. [Ownership and Data Access](#ownership-and-data-access)
5. [Defining Facts, Dimensions, and Metrics](#defining-facts-dimensions-and-metrics)
6. [Options for Creating and Managing Semantic Views](#options-for-creating-and-managing-semantic-views)
7. [YAML Conversion and Interoperability](#yaml-conversion-and-interoperability)
8. [Automated Deployment with CI/CD](#automated-deployment-with-cicd)
9. [Integration with dbt Projects](#integration-with-dbt-projects)
10. [Integration with BI Tools](#integration-with-bi-tools)
11. [Validation Rules](#validation-rules)
12. [Performance and Sizing Guidelines](#performance-and-sizing-guidelines)
13. [Troubleshooting](#troubleshooting)
14. [Additional Resources](#additional-resources)

---

## Introduction

Semantic views are Snowflake metadata objects that bridge the gap between business terminology and database schemas. They enable:

- **Business-friendly data access**: Map technical column names (like `amt_ttl_pre_dsc`) to business concepts (like `gross_revenue`)
- **Consistent metric definitions**: Ensure a single authoritative definition for calculations like `Net Revenue = SUM(gross_revenue * (1 - discount))`
- **Natural language querying**: Power Cortex Analyst for self-serve analytics
- **Unified data perspective**: Shift from querying specific data sources to focusing on business use cases

**Key Benefit**: When a user asks for "Net Revenue by Region," the semantic view knows to aggregate at the appropriate level, eliminating inconsistent calculations across reports and applications.

---

## Getting Started

Before building your semantic view, design your business data model by answering:

### 1. Identify Business Entities
- What business entities exist in your data? (customers, products, orders, etc.)
- How do these entities relate to each other?

### 2. Define Metrics and Dimensions
- What metrics are important to your business?
- What dimensions do you use to analyze these metrics?

### 3. Map to Physical Data
- Which tables contain the data you need?
- How will you join these tables?
- What calculations are needed to derive your metrics?

### 4. Recommended Starting Point
- **Start with a simple star schema**
- Begin with **3 core tables** to keep things manageable
- Start with core tables and metrics, then expand iteratively

---

## Best Practices for Creating Semantic Views

### Provide Clear Descriptions

| Guideline | Details |
|-----------|---------|
| **Use business terminology** | All names and descriptions should use familiar business language |
| **Be detailed** | Make descriptions detailed enough for non-technical users to understand |
| **Add comments** | Include helpful context in description fields for someone querying the dataset for the first time |
| **Document special cases** | For example, note the timezone of a DATETIME column |

### Include Representative User Questions

- Include questions that help the model generator better understand your intent
- Include variations of how questions might be asked
- Tailor questions based on the target audience, expected KPIs, and required data

### Review Generated Suggestions Carefully

- Ensure generated questions are relevant for the use case
- Verify suggested relationships match your business understanding
- Check that synonyms and sample values are appropriate

### Test with Real Questions

- After creating your semantic view, test it with actual business questions
- Refine your semantic view based on how well it supports these questions
- Use Cortex Analyst to validate natural language query accuracy

### Iterate on Development

```
┌──────────────────────────────────────────────────────────────┐
│  1. Start Simple → 2. Test → 3. Get Feedback → 4. Refine   │
│                        ↑                           │         │
│                        └───────────────────────────┘         │
└──────────────────────────────────────────────────────────────┘
```

- Start with a simple star schema
- Begin with core tables and metrics, then expand
- Get feedback from business users and refine

---

## Ownership and Data Access

### Shared Ownership Model

Semantic views require collaboration between:
- **Business teams**: Expertise in business cases and terminology
- **Data engineering teams**: Understanding of data access from tables and views

Both teams need to share ownership of the semantic model.

### Role-Based Access Control (RBAC)

Use RBAC to grant appropriate privileges to semantic views and their dependent objects.

#### Key Privileges for Semantic Views

| Privilege | Usage |
|-----------|-------|
| `SELECT` | Query a semantic view; also enables `DESCRIBE SEMANTIC VIEW` |
| `REFERENCES` | View structure (but not data) via Information Schema, DESCRIBE, or SHOW commands |
| `MONITOR` | View details about the semantic view and Cortex Analyst monitoring data |
| `OWNERSHIP` | Full control over the semantic view |

#### Recommended Grant Pattern

Create objects within the same database schema and use a specific custom role for access:

```sql
-- Set variables for the specified role, database, and schema
SET my_role = 'snowflake_intelligence_sales_analysis_role';
SET my_db = 'sales';
SET my_schema = 'sales_analysis';
SET my_full_schema = $my_db || '.' || $my_schema;

-- Grant usage on the database and schema
GRANT USAGE ON DATABASE IDENTIFIER($my_db) TO ROLE IDENTIFIER($my_role);
GRANT USAGE ON SCHEMA IDENTIFIER($my_full_schema) TO ROLE IDENTIFIER($my_role);

-- Grant privileges on future objects within the schema
GRANT SELECT ON FUTURE TABLES IN SCHEMA IDENTIFIER($my_full_schema) TO ROLE IDENTIFIER($my_role);
GRANT SELECT ON FUTURE VIEWS IN SCHEMA IDENTIFIER($my_full_schema) TO ROLE IDENTIFIER($my_role);
GRANT SELECT ON FUTURE SEMANTIC VIEWS IN SCHEMA IDENTIFIER($my_full_schema) TO ROLE IDENTIFIER($my_role);

-- For other object types, USAGE is the correct privilege
GRANT USAGE ON FUTURE FUNCTIONS IN SCHEMA IDENTIFIER($my_full_schema) TO ROLE IDENTIFIER($my_role);
GRANT USAGE ON FUTURE PROCEDURES IN SCHEMA IDENTIFIER($my_full_schema) TO ROLE IDENTIFIER($my_role);
GRANT USAGE ON FUTURE STAGES IN SCHEMA IDENTIFIER($my_full_schema) TO ROLE IDENTIFIER($my_role);
GRANT USAGE ON FUTURE CORTEX SEARCH SERVICES IN SCHEMA IDENTIFIER($my_full_schema) TO ROLE IDENTIFIER($my_role);
```

#### Required Privileges for Creating Semantic Views

| Privilege | Object | Notes |
|-----------|--------|-------|
| `CREATE SEMANTIC VIEW` | Schema | Required to create a new semantic view |
| `SELECT` | Table, View | Required on any tables/views used in the definition |
| `OWNERSHIP` | Existing semantic view | Required to replace an existing semantic view |

---

## Defining Facts, Dimensions, and Metrics

### Understanding the Three Elements

| Element | Role | Description |
|---------|------|-------------|
| **Facts** | Row-level numerical data | Underlying raw values before aggregation |
| **Dimensions** | Viewing perspectives | How you slice and analyze the data |
| **Metrics** | Actionable insights | Aggregated calculations from facts |

> **Important**: You must define at least one dimension or metric in the semantic view.

### Example Definitions

```sql
FACTS (
  line_items.line_item_id AS CONCAT(l_orderkey, '-', l_linenumber),
  orders.count_line_items AS COUNT(line_items.line_item_id),
  line_items.discounted_price AS l_extendedprice * (1 - l_discount)
    COMMENT = 'Extended price after discount'
)

DIMENSIONS (
  customers.customer_name AS customers.c_name
    WITH SYNONYMS = ('customer name')
    COMMENT = 'Name of the customer',
  orders.order_date AS o_orderdate
    COMMENT = 'Date when the order was placed',
  orders.order_year AS YEAR(o_orderdate)
    COMMENT = 'Year when the order was placed'
)

METRICS (
  customers.customer_count AS COUNT(c_custkey)
    COMMENT = 'Count of number of customers',
  orders.order_average_value AS AVG(orders.o_totalprice)
    COMMENT = 'Average order value across all orders',
  orders.average_line_items_per_order AS AVG(orders.count_line_items)
    COMMENT = 'Average number of line items per order'
)
```

### Tips for Modeling

| Tip | Explanation |
|-----|-------------|
| **Use wide tables instead of long tables** | If you have a table with columns like "metric" and "value", flatten the table so each metric is a column. This provides more semantic information. |
| **Capture complex calculations** | Incorporate difficult or business-specific queries into expressions |
| **Use synonyms** | Add familiar terms that users might use when querying |
| **Private metrics** | Set `access_modifier` to `private_access` for internal-only metrics |

### Allowed References in Expressions

Expressions for dimensions, facts, or metrics can reference:
- Physical columns from their own base table
- Logical columns within the same logical table
- Logical columns from other logical tables in the semantic model

**Derived metrics** can reference:
- Aggregations of dimensions and facts from any logical table
- Scalar expressions of metrics from any logical table
- Other derived metrics

> **Note**: Expressions cannot reference physical columns from other physical tables.

---

## Options for Creating and Managing Semantic Views

### Three Main Approaches

| Method | Best For | Details |
|--------|----------|---------|
| **Snowsight Wizard** | Initial setup | Automatic creation of synonyms, sample values, and column descriptions |
| **SQL DDL** | Programmatic control | Use `CREATE OR REPLACE SEMANTIC VIEW` statement |
| **YAML Upload** | Migration from semantic models | Upload existing YAML specifications |

### Recommendations

1. **Start with semantic views** (not semantic models) — they are Snowflake metadata objects with:
   - RBAC support
   - Usage statistics
   - Direct integration with Cortex Analyst and Snowflake Intelligence

2. **Use the Snowsight wizard for initial setup** — it provides helpful automation

3. **Switch to SQL DDL for production** — especially if your environment requires SERVICE user roles

### Interfaces for Management

Once created, semantic views can be managed using:
- Standard `SHOW` and `DESCRIBE` commands
- SQL `SELECT` statements
- Cortex Analyst user interface
- Streamlit or custom applications using the Cortex Analyst API
- Cortex Agents

---

## YAML Conversion and Interoperability

### Converting YAML Models to Semantic Views

Use the `SYSTEM$CREATE_SEMANTIC_VIEW_FROM_YAML` stored procedure:

```sql
-- First, verify the YAML can create a valid semantic view
CALL SYSTEM$CREATE_SEMANTIC_VIEW_FROM_YAML(
  'my_db.my_schema',
  $$
  name: TPCH_REV_ANALYSIS
  description: Semantic view for revenue analysis
  tables:
    - name: CUSTOMERS
      description: Main table for customer data
      base_table:
        database: SNOWFLAKE_SAMPLE_DATA
        schema: TPCH_SF1
        table: CUSTOMER
      primary_key:
        columns:
          - C_CUSTKEY
      dimensions:
        - name: CUSTOMER_NAME
          synonyms:
            - customer name
          description: Name of the customer
          expr: customers.c_name
          data_type: VARCHAR(25)
      metrics:
        - name: CUSTOMER_COUNT
          description: Count of number of customers
          expr: COUNT(c_custkey)
  $$,
  TRUE  -- verify_only mode
);
```

### Exporting Semantic Views to YAML

Use `SYSTEM$READ_YAML_FROM_SEMANTIC_VIEW` to:
- Enable automated post-processing
- Support round-tripping
- Serialize into version control

### Bulk Conversion

> **Note**: Snowflake does not currently support bulk conversion. You must convert YAML files one at a time. For CI/CD integration, script the conversions in a series.

---

## Automated Deployment with CI/CD

### Best Practices

| Practice | Details |
|----------|---------|
| **Version control** | Store semantic view YAML or SQL DDL in Git for version control, peer review, history, and rollback |
| **Export regularly** | If using Snowsight, export/download the YAML model regularly and commit to Git |
| **Automated triggers** | Trigger CI/CD pipelines on Git changes to run tests and accuracy checks |
| **Rollback capability** | Roll back by redeploying the previous known-good YAML or DDL from Git |

### Recommended Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│  1. Work in IDE (VS Code) → 2. Push to Git → 3. CI/CD Trigger  │
│                                                      ↓          │
│                     4. Semantic view materialized in Snowflake  │
└─────────────────────────────────────────────────────────────────┘
```

### Environment Promotion

- **Cloning**: Semantic views are cloned when schemas are cloned — a good alternative for promoting across environments in the same account
- **Sharing**: Semantic views can be shared via Snowflake Marketplace and data sharing

> **Note**: Replication is not yet supported for semantic views.

---

## Integration with dbt Projects

### Installation

Install the `dbt_semantic_view` package from Snowflake Labs:

**1. Add to `packages.yml`:**

```yaml
packages:
  - package: Snowflake-Labs/dbt_semantic_view
    version: 1.0.3  # Use latest version
```

**2. Install the package:**

```bash
dbt deps
```

### Creating Semantic View Models

Create a model in the `models` directory:

```sql
{{ config(materialized='semantic_view') }}

TABLES(
  t1 AS {{ ref('base_table') }}, 
  t2 AS {{ source('seed_sources', 'base_table2') }}
)

DIMENSIONS(
  t1.count AS value, 
  t2.volume AS value
)

METRICS(
  t1.total_rows AS SUM(t1.count), 
  t2.max_volume AS MAX(t2.volume)
)

COMMENT='test semantic view'
```

### Snowflake Connection Configuration

Configure in `profiles.yml`:

```yaml
semantic_project:
  target: snowflake
  outputs:
    snowflake:
      type: "snowflake"
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      authenticator: "{{ env_var('SNOWFLAKE_AUTHENTICATOR') }}"
      role: "{{ env_var('SNOWFLAKE_ROLE') }}"
      database: "{{ env_var('SNOWFLAKE_DATABASE') }}"
      warehouse: "{{ env_var('SNOWFLAKE_WAREHOUSE') }}"
      schema: "{{ env_var('SNOWFLAKE_SCHEMA') }}"
      threads: 4
```

### Running dbt Build

```bash
dbt build --target snowflake --select semantic_view_basic+
```

> **Note**: The code samples in Snowflake Labs are intended for reference, testing, and educational purposes and are not covered by any Service-Level Agreement.

**Resources**:
- [dbt_semantic_view package](https://hub.getdbt.com/Snowflake-Labs/dbt_semantic_view/latest/)
- [Snowflake Engineering Blog: dbt Semantic View Package](https://www.snowflake.com/en/engineering-blog/dbt-semantic-view-package/)

---

## Integration with BI Tools

Several BI tool vendors offer integrations with Snowflake semantic views:

| Tool | Integration Link |
|------|------------------|
| **Sigma** | [Snowflake Semantic Views Launch](https://www.sigmacomputing.com/blog/snowflake-semantic-views-launch) |
| **Omni** | [Omni + Snowflake](https://omni.co/snowflake) |
| **Honeydew** | [Honeydew and Snowflake Semantic Views](https://honeydew.ai/blog/honeydew-and-snowflake-semantic-views/) |
| **Hex** | [Snowflake Semantic Sync & AISQL](https://hex.tech/blog/introducing-snowflake-semantic-sync-aisql/) |

Contact your BI tool account teams for specific integration details.

---

## Validation Rules

Snowflake validates semantic views when you define them. Key rules include:

### General Validation Rules

| Rule | Description |
|------|-------------|
| **Required elements** | Must define at least one dimension or metric |
| **Primary/foreign keys** | Must use physical base table columns or expressions that directly refer to base table columns |

### Relationship Validation

- Relationships must match the actual data structure
- Foreign keys must reference valid primary keys

### Expression Validation

| Expression Type | Rules |
|-----------------|-------|
| **Row-level (dimensions, facts)** | Can reference physical columns from own base table, logical columns within same table, logical columns from other tables |
| **Aggregate-level (metrics)** | Must use proper aggregation functions |
| **Window functions** | Must include dimensions specified in `PARTITION BY` and `ORDER BY` clauses |

---

## Performance and Sizing Guidelines

### Column Limits

> **Practical guideline**: Semantic views should have no more than **50-100 columns** total across all tables.

This applies to both native semantic views and YAML-based models. The limit is primarily due to context window limits in AI components such as Cortex Analyst.

**Impact of exceeding**: May lead to latency or quality degradation (not a hard technical boundary).

### Performance Tips

| Tip | Details |
|-----|---------|
| **Keep table count low** | Start with 3 tables, expand as needed |
| **Limit columns** | Only include columns needed to answer expected questions |
| **Use star schema** | Simplifies relationships and improves query performance |

---

## Troubleshooting

### Common Issues and Solutions

| Issue | Solution |
|-------|----------|
| **Semantic view not listed** | Refresh the list of models (not the page itself) |
| **Relationship errors** | Ensure relationships match the actual data structure |
| **Slow queries** | Reduce the number of tables or columns |
| **Unexpected Cortex Analyst results** | Review facts, dimensions, and metrics definitions |
| **"Table/search service/stage does not exist" errors** | Verify privileges are set correctly (see below) |

### Privilege Verification Checklist

For each semantic view:
- [ ] User's default role has `USAGE` on the database and schema
- [ ] User's default role has `REFERENCES` on the semantic view (or `SELECT` if querying)
- [ ] User's default role has `SELECT` on each underlying table

For each Cortex Search service:
- [ ] User's default role has `USAGE` on the database and schema
- [ ] User has `USAGE` on the Cortex Search service

---

## Additional Resources

### Official Documentation

- [Overview of Semantic Views](https://docs.snowflake.com/en/user-guide/views-semantic/)
- [Best Practices for Semantic Views (Development)](https://docs.snowflake.com/en/user-guide/views-semantic/best-practices-dev)
- [Using Snowsight to Create Semantic Views](https://docs.snowflake.com/en/user-guide/views-semantic/ui)
- [Using SQL Commands for Semantic Views](https://docs.snowflake.com/en/user-guide/views-semantic/sql)
- [Validation Rules](https://docs.snowflake.com/en/user-guide/views-semantic/validation-rules)
- [Querying Semantic Views](https://docs.snowflake.com/en/user-guide/views-semantic/querying)

### Related Technologies

- [Cortex Analyst Overview](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst)
- [Cortex Analyst REST API](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst-api)
- [Cortex Search Service](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search)

### Community Resources

- [dbt_semantic_view Package on dbt Hub](https://hub.getdbt.com/Snowflake-Labs/dbt_semantic_view/latest/)
- [Snowflake Engineering Blog](https://www.snowflake.com/en/engineering-blog/)

---

## Summary: Quick Reference Checklist

### Before You Start
- [ ] Design business data model with entities and relationships
- [ ] Define key metrics and dimensions
- [ ] Map business concepts to physical tables
- [ ] Plan for star schema (recommend starting with 3 tables)

### During Development
- [ ] Use business terminology in names and descriptions
- [ ] Include representative user questions
- [ ] Add synonyms for common terms
- [ ] Keep total columns under 50-100 across all tables
- [ ] Test with real business questions

### Security & Access
- [ ] Set up RBAC with custom roles
- [ ] Grant appropriate privileges on future objects
- [ ] Verify privileges for all dependent objects

### Deployment
- [ ] Store definitions in Git for version control
- [ ] Set up CI/CD pipelines for automated deployment
- [ ] Export YAML regularly for backup
- [ ] Use schema cloning for environment promotion

### Integration
- [ ] Configure dbt package if using dbt workflows
- [ ] Connect BI tools via supported integrations
- [ ] Integrate with Cortex Analyst for natural language queries

---

*Document Version: 1.0*  
*Last Updated: January 2026*  
*Sources: [Snowflake Documentation](https://docs.snowflake.com/en/user-guide/views-semantic/)*

