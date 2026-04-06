---
description: Run queries against the warehouse and return analysis, insights, and findings. Use when the user asks a data question, wants to understand trends, investigate a metric, diagnose a change, or needs a data-driven answer or report.
argument-hint: '<question or analysis request>'
---

# /analyze - Answer Data Questions with Analysis

Run queries, interpret the results, and deliver findings. The output is analysis and insights — not just query files.

## Note on tool names
The actual tool names may be prefixed with a namespacing string. We will refer to the tool names without any prefix. Please take care to understand what tool is referenced.

## Workflow

### 1. Clarify the Question

Parse the request and determine:

- **Scope**: Quick lookup, multi-dimensional analysis, or formal report?
  - *Quick*: Single metric or factual lookup (e.g., "How many users signed up last week?")
  - *Analysis*: Trend, comparison, or root cause (e.g., "What is driving the drop in conversion rate?")
  - *Report*: Comprehensive investigation with methodology, caveats, and recommendations
- **Key metrics and dimensions**: What numbers matter, what to slice by, what time range?
- **Output form**: Number, table, narrative, or combination?

You are allowed to ask the user for clarifications — especially about time ranges or relevant segments. If there are known caveats in the data that affect certain periods, mention them and agree on scope before diving in.

#### Relevant tools
- `grep_table_names`
- `grep_references`
- `search_references`

### 2. Maximize Context Understanding

**This is the most important step.** You create value by deeply understanding the data before writing a single line of SQL — not by guessing from schema alone. Most relevant context is hidden inside the actual data and in the real-world meaning of columns. Take great pride in this process.

You can never base analysis on queries written by just looking at table schemas. Use `search_references` for semantic context (existing SQL files, prior chat conversations, warehouse knowledge snippets) **and** run exploratory queries with `run_query` until your choices are based on verified facts.

#### Things to always investigate

- **Column reality**: For columns you plan to use, inspect actual values — format, null rate, outliers, cardinality. Column names often mislead.
- **Historical changes**: Whether definitions or join keys changed over time. Example: "before 2025, `ui_name` was non-null while `ui_id` was null — now it is the other way around." Your filters and joins must match the target period.
- **Cross-column dependencies**: Rules like "whenever `market` is UK, `vat_amount` is null" that affect whether a metric is valid for all slices.
- **Coverage**: That there is actually data for the date ranges and dimensions you will analyze.
- **Choosing between similar tables**: If more than one table could hold the same fact, run comparison queries (row counts, overlap, key distributions) and choose deliberately. Example: both `transactions` and `purchases` exist — understand how they differ before basing analysis on one of them.
- **Joins**: Confirm join keys match in practice. Trim/case/format mismatches are common. If row counts explode or collapse unexpectedly after a join, debug it before proceeding.
- **Cross-table coherence**: Sanity-check volumes across tables you combine. Example: if `sessions` has ~2000 rows/day but `purchases` has ~6000/day, something is off — resolve it before drawing conclusions.

#### Types of references available

- **Repository files**: `.sql` files, code, and other project files — often contain existing queries and patterns to build on.
- **Previous chat conversations**: Prior investigations and insights from the user that can save you from re-discovering things.
- **Knowledge snippets**: Curated documentation about the warehouse (join relationships, caveats, metric definitions) — treat these as fact.

#### Sub-agents for parallel exploration

When there are multiple independent aspects of the data to investigate, create sub-agents using the Task tool so they can run in parallel.

#### Relevant tools
- `get_table_overview`
- `search_references`
- `run_query`
- `grep_table_names`
- `grep_column_names`
- `grep_references`

### 3. Run Queries and Retrieve Data

Execute queries to get the data needed for analysis:

1. Write SQL targeting the correct grain, columns, and filters
2. Run with `run_query` and inspect the results
3. If results look unexpected, debug before drawing any conclusions
4. For complex analyses, break the problem into sub-questions and run separate queries for each

### 4. Analyze

With data in hand:

- Calculate relevant metrics, aggregations, and comparisons
- Identify patterns, trends, outliers, and anomalies
- Compare across dimensions (time periods, segments, categories)
- For complex analyses, synthesize findings across sub-questions

### 5. Present Findings

**Quick answer:**
- State the answer directly with relevant context
- Include the query used (in a collapsed section or code block) for reproducibility

**Analysis:**
- Lead with the key finding or insight
- Support with data tables
- Note methodology and any caveats
- Suggest natural follow-up questions

**Formal report:**
- Executive summary with key takeaways
- Methodology: approach and data sources used
- Detailed findings with supporting data
- Caveats, limitations, and data quality notes
- Recommendations and suggested next steps

Restate important intermediate results clearly — not everything is well-presented in the tool output, and the user needs to follow the logic.

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

**Confidence scale:** 3 = directly validated or user-stated fact · 2 = very likely, well-investigated · 1 = qualified guess (avoid basing final analysis on these)

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
