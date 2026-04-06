---
name: learn-context
description: Research the data warehouse using MCP tools and write structured knowledge context items via write_knowledge. Use when the user wants to capture, document, or save knowledge about tables, metrics, joins, or business context in the data warehouse. Also use when asked to "learn", "save this as knowledge", "document this", or "write a knowledge item".
---

# Learn Knowledge

Research the data warehouse using MCP tools, then write well-founded knowledge objects that future AI agents can use as context when writing queries.

## Note on tool names
The actual tool names may be prefixed with a namespacing string. We will refer to the tool names without any prefix. Please take care to understand what tool is referenced.

## Tools Available

| Tool | When to use |
|------|-------------|
| `get_table_overview` | First step when working with any table — schema, grain, row count, mandatory context |
| `grep_table_names` | Discover tables by name pattern |
| `grep_references` | Find exact text in existing knowledge and references |
| `search_references` | Find existing knowledge by meaning (semantic search) |
| `run_query` | Inspect actual data — formats, nulls, distributions, join validity |
| `write_knowledge` | Persist the final knowledge objects |

## Workflow

### 1. Search for existing knowledge first
Before writing anything, search for existing items that cover the same topic. If found, update them rather than creating duplicates. Pass `knowledge_id` to `write_knowledge` to edit an existing item.

```
search_references("how is [metric/table/concept] defined")
grep_references("[column or table name]")
```

### 2. Research before writing
Never write knowledge based only on schema names and guesses. Run queries and search references to validate your understanding. The value is in what is non-obvious — unexpected formats, gotchas, cross-table relationships.

Things to investigate for any table or column:
- Actual data formats and value ranges (not just types)
- Null rates and whether they are meaningful
- Changes over time (e.g. "before May 2024 this column was always null")
- Dependencies between columns (e.g. "when market = 'UK', vat_amount is null")
- When multiple similar tables exist, run queries to understand the difference before choosing

When investigating joins, always verify they work as expected — check for match rates and watch for format mismatches between join keys.

### 3. Write knowledge objects
Call `write_knowledge` once per knowledge object. Keep each object focused on a single concept. Split long items into multiple focused ones — they embed better for semantic search.

After writing, give the user a short summary of what was written and the key findings behind each item.

### 4. State your assumptions
List every assumption made, with reasoning and a confidence score:

```markdown
## Assumptions Made

- **[statement]**
  - **Confidence:** 3/3
  - **Reasoning:** Observed in data: no nulls for 3 years of records.

- **[statement]**
  - **Confidence:** 2/3
  - **Reasoning:** Column name strongly suggests it, amounts are in plausible range, no other candidate column exists.

- **[statement]**
  - **Confidence:** 1/3
  - **Reasoning:** Inferred from naming convention, not verified in data.
```

**Confidence scale:** 3 = directly validated or user-stated fact · 2 = very likely, well-investigated · 1 = qualified guess (avoid basing final knowledge on these)

---

## Knowledge Types

### `metric_definition`
How to calculate a business metric or KPI.

*Example:* "Monthly Active Users is defined as distinct `user_id` values in `acme.events.sessions` within a calendar month, where `session_type != 'bot'`."

### `business_context`
Business facts, rules, or conventions behind the data.

*Example:* "The `market` column uses ISO 3166-1 alpha-2 country codes throughout the warehouse."

### `join_path`
How to join two or more tables. Requires `metadata`:

```json
{
  "tables": [
    {"table_id": "project.dataset.table_a", "columns": ["user_id"]},
    {"table_id": "project.dataset.table_b", "columns": ["user_id"]}
  ]
}
```

Use the `content` field for caveats (fanout risks, NULL handling, required filters). Leave `tagged_entities` empty — entities are derived from metadata.

### `ask_the_user`
Instructs future agents to ask the user a specific question before using a table.

*Example:* "This table contains multiple `transaction_type` values. Always ask the user which types to include before writing a query."

### `other`
Anything that doesn't fit the above.

---

## What Makes Good Knowledge

Good knowledge is **self-contained**, **specific**, and **non-obvious**.

| Bad | Good |
|-----|------|
| "The `amount` column contains the transaction amount." | "The `amount` column appears to store amounts multiplied by 100 (range 1090–14900), likely to avoid floats. Divide by 100 for display." |
| "40% of `name` values are null." | "The `name` column has been non-null since 2024-03-01; before that date it is always null, indicating a pipeline change." |
| "There is a correlation between `session_length` and `ltv`." | *(Don't write this — it's analysis, not data format context.)* |

Focus on facts about **data format, structure, business rules, and join behaviour** — not statistical insights or trends.

Each object should be a couple of sentences max. Longer content = split into multiple objects.

---

## Examples of Good Process

**User:** "I want to know how VAT is tracked in the transactions table."

1. `search_references("VAT transactions")` — check existing knowledge
2. `grep_table_names("transaction")` — find the right table
3. `get_table_overview("project.dataset.transactions")` — understand schema
4. Run queries to inspect VAT-related columns: formats, nulls, value ranges, changes over time
5. `write_knowledge` with findings — one object per concept (VAT column format, currency column, rate field added in 2024, etc.)
6. State assumptions with confidence scores

**User:** "Save this as knowledge: the `orders` table uses `order_status = 'completed'` for fulfilled orders."

1. `search_references("orders order_status")` — check for duplicates
2. Quick verification query: `SELECT DISTINCT order_status FROM ... LIMIT 20`
3. `write_knowledge` with the fact + the user's statement noted as confidence=3 source
4. State assumption: user-provided, confidence=3
