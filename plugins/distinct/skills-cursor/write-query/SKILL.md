---
description: Write a validated SQL query to a file in the current project. Use when the user wants to create or save a query, generate a SQL file, write a report query, or add a new query to the project.
argument-hint: '<description of query to write>'
---

# /write-query - Write a SQL Query to the Project

Explore the data warehouse to fully understand the data, write a validated SQL query, and save it as a `.sql` file in the right location within the current project.

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

### 2. Find the Right Location and Naming Convention

Before writing any files, explore the project to understand where SQL queries live and how they are named:

1. Use Glob to find existing `.sql` files in the project (e.g. `**/*.sql`)
2. Read a few of them to understand naming conventions, folder structure, and any common patterns (CTEs, aliases, comment headers)
3. Identify the most appropriate directory for the new file
4. Choose a filename consistent with the existing convention

If the project has no existing `.sql` files, default to a `queries/` folder at the project root and use `snake_case` filenames.

### 3. Maximize Context Understanding

**This is the most important step.** You create value by deeply understanding the data before writing a single line of SQL — not by guessing from schema alone. Most relevant context is hidden inside the actual data and in the real-world meaning of columns. Take great pride in this process.

**Read project files directly first** — the `.sql` files you found in step 2 are the most reliable source of truth for how tables are actually queried in this project. Read the most relevant ones before reaching for MCP tools. Then combine with `search_references` for semantic context (prior chat conversations, warehouse knowledge snippets) **and** run exploratory queries with `run_query` until your choices are based on verified facts.

#### Things to always investigate

- **Column reality**: For columns you plan to use, inspect actual values — format, null rate, outliers, cardinality. Column names often mislead.
- **Historical changes**: Whether definitions or join keys changed over time. Example: "before 2025, `ui_name` was non-null while `ui_id` was null — now it is the other way around." Your filters and joins must match the target period.
- **Cross-column dependencies**: Rules like "whenever `market` is UK, `vat_amount` is null" that affect whether a column is valid for all slices.
- **Coverage**: That there is actually data for the date ranges and filters the query will use.
- **Choosing between similar tables**: If more than one table could hold the same fact, run comparison queries (row counts, overlap, key distributions) and choose deliberately. Example: both `transactions` and `purchases` exist — understand how they differ before writing the query.
- **Joins**: Confirm join keys match in practice. Trim/case/format mismatches are common. If row counts explode or collapse unexpectedly after a join, debug it before proceeding.
- **Cross-table coherence**: Sanity-check volumes across tables you combine. Example: if `sessions` has ~2000 rows/day but `purchases` has ~6000/day, something is off — resolve it before writing the final query.

#### Context sources to use

- **Project files (read directly)**: Existing `.sql` files in the project — start here. They show real table names, join patterns, and conventions used by the team.
- **Previous chat conversations**: Prior investigations and insights from the user accessible via `search_references` — can save you from re-discovering things.
- **Knowledge snippets**: Curated documentation about the warehouse (join relationships, caveats, metric definitions) accessible via `search_references` — treat these as fact.

#### Relevant tools
- `get_table_overview`
- `search_references`
- `run_query`
- `grep_table_names`
- `grep_column_names`
- `grep_references`

### 4. Write and Validate the Query

Once you have sufficient context:

1. Write the SQL targeting the correct grain, columns, and filters
2. Execute it with `run_query` to verify results look correct
3. Check row counts, null rates, and value ranges
4. If results look unexpected, debug before finalizing
5. Write the validated query to the chosen `.sql` file path using the Write tool

### 5. Output

Your final output must be:

1. **Write the file** to the correct path in the project
2. **A short explanation** of what the query does, what data it returns, and where the file was saved
3. **The assumptions list** — see format below

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
