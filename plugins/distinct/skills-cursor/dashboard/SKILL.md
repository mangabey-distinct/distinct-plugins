---
description: Build an interactive dashboard with charts, filters, and tables
argument-hint: ' <description of dashboard>'
---

# /build-dashboard - Build Interactive Dashboards

Build an interactive dashboard with charts, filters, tables, and KPIs using the Distinct dashboard format. Dashboards are defined as JSON and rendered automatically by the platform -- you do not generate HTML, CSS, or JavaScript.

## Usage

```
/build-dashboard <description of dashboard>
```

## Note on tool names
The actual tool names may be prefixed with a namespacing string. We will refer to the tool names without any prefix. Please take care to understand what tool is referenced.

## Workflow

### 1. Understand the Dashboard Requirements

Determine:

- **Purpose**: Executive overview, operational monitoring, deep-dive analysis, team reporting
- **Audience**: Who will use this dashboard?
- **Key metrics**: What numbers matter most?
- **Dimensions**: What should users be able to filter or slice by?
- **Data source**: Which tables and columns are needed?

If the request is ambiguous or you are unsure about time ranges, metrics, or dimensions, **ask the user clarifying questions before building**. It is better to ask than to guess wrong.

#### Relevant tools

- `grep_table_names`
- `grep_references`
- `search_references`

### 2. Explore and Test Queries

**Before entering any SQL into the dashboard definition, run the queries first** using `run_query` to verify they return the expected columns and data. This avoids broken dashboards.

**Maximize context understanding.** Most of the value you add is in discovering how the data actually behaves, not in guessing from schema alone. **Read project files directly first** — use Grep and Glob to find existing `.sql` files, dbt models, and related code in the current project, then read them. Combine this with `search_references` / `grep_references` for semantic context (prior chats, warehouse knowledge) **and** run exploratory queries so dashboard SQL is based on what you have verified, not assumptions.

Concrete things to check with `run_query` (and small follow-up queries) before locking queries into the dashboard:

- **Column reality**: For columns you will chart or filter on, inspect actual values—format, null rate, outliers, and cardinality. Names and types on paper often miss how the field is really populated.
- **Time and history**: Whether definitions or join keys changed over time. Example pattern: before a given year one identifier column was populated while another was null, and afterward the reverse—your filters and joins must match the period you are showing.
- **Cross-column rules**: Dependencies such as “whenever `market` is UK, `vat_amount` is null” that affect whether a metric or breakdown is valid for all slices.
- **Coverage**: That there is data for the date ranges and parameter values the dashboard will expose; empty charts often mean the query never got a reality check on the same bounds.
- **Choosing between similar tables**: If more than one table could hold the same fact, run comparison queries (row counts, overlap, key distributions) and pick deliberately. Example: both `transactions` and `purchases` exist—understand how they differ before wiring the dashboard to one of them.
- **Joins**: Confirm join keys match in practice (trim/case/format mismatches are common). If row counts explode or collapse unexpectedly, debug the join with counts and sample mismatches before baking it into a chart.
- **Coherence across sources**: Sanity-check volumes and relationships across tables you combine. Example: if `sessions` is on the order of thousands per day but `purchases` is multiples higher without a clear story, you may be on the wrong grain or table—resolve that before building KPIs that depend on both.

When you hand off the dashboard (or summarize for the user), be explicit about how you validated the important choices (which table, which join, which time rules)—same spirit as “informed choices backed by investigation,” not silent guesses.

1. Explore the schema to find relevant tables and columns
2. Write SQL queries for the needed data
3. Execute each query and verify the results look correct
4. If a query fails or results look unexpected, debug and fix before proceeding

#### Relevant tools

- `get_table_overview`
- `search_references`
- `run_query` -- use this to test every query before adding it to the dashboard
- `grep_table_names`
- `grep_column_names`
- `grep_references`

### 3. Design the Dashboard Layout

The dashboard uses a **12-column grid**. Plan which charts go where using `col_start` (1-based) and `col_span`.

**Adapt the layout to the content:**

- Prefer to put KPI cards at the top, and align them horizontally.
- Avoid using more than 2 charts on one row, and prefer to stack your content horizontally.
- It is suggested to add a detailed data table at the end of the dashboard.
- Use parameters a lot.

### 4. Build the Dashboard Definition

Construct a `DashboardDefinition` JSON object with these sections:

- **`id`**: A URL-safe slug (e.g., `"sales-overview"`)
- **`title`**: Human-readable dashboard title
- **`parameters`** (optional): Runtime filters the user can change in the UI. Parameters are injected into queries and transforms using `{{param_name}}` syntax.
- **`queries`**: Map of query IDs to SQL. Queries can reference parameters with `{{param_name}}`. Each query runs against the data warehouse and produces a table of results.
- **`transforms`** (optional): Map of transform IDs to pandas code. Each transform takes a query (or another transform) as input and produces a new dataset. Useful when a single query can feed multiple elements with different aggregations.
- **`ui_elements`**: List of UI element definitions (charts, tables, KPIs). Each element references a query or transform as its `data_source`. Chart elements have `type`, `x`, and `y`. Table elements have `type: "table"` and optional `y` for column selection. KPI elements have `type: "kpi"`, `column`, optional `aggregate` (sum, avg, min, max, mode, or none), and optional `description` (~max 10 words). If `aggregate` is none or omitted, the source data must have exactly 1 row.
- **`layout`**: Grid placement for each element using `element_id`, `col_start` (1-12), and `col_span` (1-12).

Then call `write_dashboard` to save it.

If you are **updating** an existing dashboard, always call `read_dashboard` first to get the current state and preserve any elements you are not changing.

#### Relevant tools

- `write_dashboard` -- saves the dashboard and returns a URL
- `read_dashboard` -- read before modifying an existing dashboard
- `list_dashboards` -- lists the dashboards the user has access to edit

### 5. Present the Result

After `write_dashboard` succeeds, it returns a response according to the following PyDantic format:

```python
class DashboardWriteResponse(BaseModel):
    """Response from saving a dashboard."""

    dashboard_id: str = Field(..., description="ID of the saved dashboard.")
    title: str = Field(..., description="Display title of the dashboard.")
    url: str = Field(
        ...,
        description="Deep-link URL (distinct://...) to open in the app.",
    )
    url_browser: str = Field(
        ...,
        description="Browser URL (http[s]://[domain]/desktop/...) that will in turn redirect to the deep link. "
        "Use this in browser-based environments where deep links cannot be opened directly.",
    )
    message: str = Field(..., description="Human-readable success message.")
```

Always present one of the URLs to the user after a successful write. Prefer `url` (the deep link) since Cursor can open it directly. Use `url_browser` if you have reason to believe the environment cannot handle deep links. Style it as a markdown link with the dashboard name as the text.

**Always** present the link when creating a new dashboard so the user can navigate to it immediately. On subsequent writes it is less critical since the dashboard auto-updates if the user already has it open.
The dashboard content auto updates for the user if they have it open.

## Performance Guidelines

- **Under 10,000 data points**: Aggregation in pandas transforms is fine. This is especially useful when a single query feeds multiple charts via different transforms.
- **Over 10,000 data points**: Prefer aggregating in the data warehouse (SQL) rather than pulling raw rows and aggregating in transforms. Large row counts slow down the dashboard rendering pipeline.
- When in doubt, aggregate in SQL. It is always safe and often faster.

## Element Types

- **Charts** (`line`, `bar`, `area`, `pie`, `scatter`): Use `x` and `y` columns. `line` for time series, `bar` for category comparisons, `pie` for composition (best with fewer than 6 categories).
- **`table`**: Detailed tabular data. Optional `y` to specify which columns to show; if empty or omitted, all columns are shown.
- **`kpi`**: Single-value KPI card. Requires `column` (the column name). Use `aggregate` (sum, avg, min, max, mode, or none) when the source has multiple rows. Use `description` for a short label (~max 10 words). If `aggregate` is none or omitted, the source must have exactly 1 row.

## Complete Example

> A sales dashboard with a date parameter, two queries (one reused via transforms for two different charts), KPI cards, trend chart, category breakdown, and a detail table.

```json
{
    "id": "regional-sales-dashboard",
    "title": "Regional Sales Dashboard",
    "parameters": {
        "start_date": {
            "type": "date",
            "label": "Start Date",
            "default": "2025-01-01"
        },
        "region": {
            "type": "select",
            "label": "Region",
            "default": "All",
            "options": ["All", "North", "South", "East", "West"]
        }
    },
    "queries": {
        "order_details": {
            "sql": "SELECT order_date, region, product_category, revenue, order_id FROM `project.dataset.orders` WHERE order_date >= '{{start_date}}' AND (region = '{{region}}' OR '{{region}}' = 'All')"
        },
        "kpi_summary": {
            "sql": "SELECT COUNT(DISTINCT order_id) AS total_orders, SUM(revenue) AS total_revenue, AVG(revenue) AS avg_order_value, COUNT(DISTINCT region) AS active_regions FROM `project.dataset.orders` WHERE order_date >= '{{start_date}}' AND (region = '{{region}}' OR '{{region}}' = 'All')"
        }
    },
    "transforms": {
        "monthly_trend": {
            "source": "order_details",
            "code": "df['order_month'] = pd.to_datetime(df['order_date']).dt.to_period('M').astype(str)\nresult = df.groupby('order_month')['revenue'].sum().reset_index()"
        },
        "category_breakdown": {
            "source": "order_details",
            "code": "result = df.groupby('product_category')['revenue'].sum().reset_index().sort_values('revenue', ascending=False)"
        }
    },
    "ui_elements": [
        {
            "id": "total-orders",
            "type": "kpi",
            "title": "Total Orders",
            "data_source": "kpi_summary",
            "column": "total_orders",
            "description": "Total completed orders"
        },
        {
            "id": "total-revenue",
            "type": "kpi",
            "title": "Total Revenue",
            "data_source": "kpi_summary",
            "column": "total_revenue",
            "description": "Sum of all revenue"
        },
        {
            "id": "avg-order-value",
            "type": "kpi",
            "title": "Avg Order Value",
            "data_source": "kpi_summary",
            "column": "avg_order_value",
            "description": "Average revenue per order"
        },
        {
            "id": "active-regions",
            "type": "kpi",
            "title": "Active Regions",
            "data_source": "kpi_summary",
            "column": "active_regions",
            "description": "Regions with orders"
        },
        {
            "id": "revenue-trend",
            "type": "line",
            "title": "Monthly Revenue Trend",
            "data_source": "monthly_trend",
            "x": "order_month",
            "y": ["revenue"]
        },
        {
            "id": "category-chart",
            "type": "bar",
            "title": "Revenue by Category",
            "data_source": "category_breakdown",
            "x": "product_category",
            "y": ["revenue"]
        },
        {
            "id": "order-table",
            "type": "table",
            "title": "Order Details",
            "data_source": "order_details",
            "y": []
        }
    ],
    "layout": [
        { "element_id": "total-orders", "col_start": 1, "col_span": 3 },
        { "element_id": "total-revenue", "col_start": 4, "col_span": 3 },
        { "element_id": "avg-order-value", "col_start": 7, "col_span": 3 },
        { "element_id": "active-regions", "col_start": 10, "col_span": 3 },
        { "element_id": "revenue-trend", "col_start": 1, "col_span": 6 },
        { "element_id": "category-chart", "col_start": 7, "col_span": 6 },
        { "element_id": "order-table", "col_start": 1, "col_span": 12 }
    ]
}
```

**What this example demonstrates:**

- **Parameters** (`start_date`, `region`) used with `{{...}}` syntax in query SQL to create interactive filters
- **Query reuse** via transforms: `order_details` feeds both `monthly_trend` and `category_breakdown` transforms, avoiding duplicate warehouse queries
- **KPI cards** pulling from a dedicated summary query
- **12-column grid layout**: KPIs across the top (3 cols each), two charts side-by-side (6 cols each), full-width table at the bottom

## Examples

```
/build-dashboard Monthly sales dashboard with revenue trend, top products, and regional breakdown. Data is in the orders table.
```

```
/build-dashboard Build an operational dashboard for support tickets showing volume by priority, response time trends, and resolution rates.
```

```
/build-dashboard Create an executive dashboard showing MRR, churn, new customers, and NPS over the last 12 months.
```

## Tips

- Always **run queries with `run_query` first** before adding them to the dashboard definition to verify they work
- If updating an existing dashboard, **always call `read_dashboard` first** so you don't lose existing content
- After `write_dashboard`, **present the returned URL** to the user so they can view the dashboard immediately
- Use transforms when the same base query can serve multiple charts with different aggregations -- this reduces warehouse load
- For dashboards with user-selectable filters, use parameters and `{{param_name}}` in your SQL
- It is fine to ask the user questions about time ranges, preferred metrics, or dimensions before building
- For recent-period analyses, default to the last 12 months unless the user specifies otherwise
