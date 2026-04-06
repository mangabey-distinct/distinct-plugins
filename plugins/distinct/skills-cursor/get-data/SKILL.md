---
description: Fetch data from the warehouse by writing validated SQL query files. Use when the user wants to extract data, pull a report, write or generate a query, or get records from the data warehouse.
argument-hint: '<description of data to fetch>'
---

# /get-data - Fetch Data from the Warehouse

Write validated SQL queries to `.sql` files that extract the data the user needs. The final output is query files — not raw data results — unless the user explicitly asks to see them.

## Note on tool names
The actual tool names may be prefixed with a namespacing string. We will refer to the tool names without any prefix. Please take care to understand what tool is referenced.

## Workflow

### 1. Clarify the Expected Query Structure

Before exploring the data, nail down the shape of the SQL to write:

- **Grain**: What does one row represent? (e.g., one transaction, one user per day, one session)
- **Columns**: What fields should the query return?
- **Filters**: Date range, segment, dimension slices?
- **Form**: Aggregated summary or row-level detail?

If any of these are ambiguous, ask the user before proceeding. It is better to ask once up front than to write the wrong query.

#### Relevant tools
- `grep_table_names`
- `grep_references`
- `search_references`

### 2. Maximize Context Understanding

**This is the most important step.** You create value by deeply understanding the data before writing a single line of SQL — not by guessing from schema alone. Most relevant context is hidden inside the actual data and in the real-world meaning of columns. Take great pride in this process.

You can never write the final query by just looking at table schemas. **Read project files directly first** — use Grep and Glob to find existing `.sql` files, dbt models, and related code in the current project, then read them. This is often the fastest way to understand how the data is actually queried in practice. Combine this with `search_references` for semantic context (prior chat conversations, warehouse knowledge snippets) **and** run exploratory queries with `run_query` until your choices are based on verified facts.

#### Things to always investigate

- **Column reality**: For columns you plan to use, inspect actual values — format, null rate, outliers, cardinality. Column names often mislead.
- **Historical changes**: Whether definitions or join keys changed over time. Example: "before 2025, `ui_name` was non-null while `ui_id` was null — now it is the other way around." Your filters and joins must match the target period.
- **Cross-column dependencies**: Rules like "whenever `market` is UK, `vat_amount` is null" that affect whether a column is valid for all slices.
- **Coverage**: That there is actually data for the date ranges and filters the query will use.
- **Choosing between similar tables**: If more than one table could hold the same fact, run comparison queries (row counts, overlap, key distributions) and choose deliberately. Example: both `transactions` and `purchases` exist — understand how they differ before writing the query.
- **Joins**: Confirm join keys match in practice. Trim/case/format mismatches are common. If row counts explode or collapse unexpectedly after a join, debug it before proceeding.
- **Cross-table coherence**: Sanity-check volumes across tables you combine. Example: if `sessions` has ~2000 rows/day but `purchases` has ~6000/day, something is off — resolve it before writing the final query.

#### Context sources to use

- **Project files (read directly)**: Use Grep and Glob to find `.sql` files, dbt models, and other code in the current project, then read them with the Read tool. Existing queries are the most reliable source of truth for how tables are actually used — start here.
- **Previous chat conversations**: Prior investigations and insights from the user accessible via `search_references` — can save you from re-discovering things.
- **Knowledge snippets**: Curated documentation about the warehouse (join relationships, caveats, metric definitions) accessible via `search_references` — treat these as fact.

#### Relevant tools
- `get_table_overview`
- `search_references`
- `run_query`
- `grep_table_names`
- `grep_column_names`
- `grep_references`

### 3. Write and Validate the Query

Once you have sufficient context:

1. Write the SQL targeting the correct grain, columns, and filters
2. Execute it with `run_query` to verify results look correct
3. Check row counts, null rates, and value ranges
4. If results look unexpected, debug before finalizing
5. Write the validated query to a `.sql` file

### 4. Output

Your final output must be:

1. **Tool calls to write the query/queries to `.sql` files**
2. **A short explanation** of what each query does and what data it returns
3. **The assumptions list** — see format below

Do not return raw data results unless the user explicitly asks for them.

## Assumptions Format

Every choice you made about tables, joins, columns, and filters is an assumption. State them all — transparency is the standard.

```markdown
## Assumptions Made

- **[statement]**
  - **Confidence:** 3/3
  - **Reasoning:** ...

- **[statement]**
  - **Confidence:** 2/3
  - **Reasoning:** ...
```

**Confidence scale:** 3 = directly validated or user-stated fact · 2 = very likely, well-investigated · 1 = qualified guess (avoid basing final queries on these)

**Examples:**

```markdown
## Assumptions Made

- **The table `transactions` is the main table for purchases.**
  - **Confidence:** 3/3
  - **Reasoning:** Provided by the user.

- **The item ID is extracted as `(select value from unnest(transaction_params) where key = 'item_id') as item_id`.**
  - **Confidence:** 2/3
  - **Reasoning:** All transactions for the past 3 years have this id set with no duplicates. All but 4 rows join successfully to dim_items using it.

- **Total revenue is taken from the `sales` column.**
  - **Confidence:** 1/3
  - **Reasoning:** Column name and value range (0.49–9.99 USD) suggest this is revenue. No other column is a candidate.
```
