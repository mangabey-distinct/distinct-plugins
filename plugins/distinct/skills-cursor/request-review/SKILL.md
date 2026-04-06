---
description: Send a query and its results for review from a data expert.
---

# Request Review

> To get a full picture of the context in which this skill is used, check the [ECOSYSTEM.md](../../ECOSYSTEM.md).
> If you have not yet read this file, do so BEFORE starting to work on the task.

Send a completed SQL query and its results to all registered team members for peer review.

## Usage

Offer this after any analysis that produces a `run_query` result:

```
/request-review
```

Or the user may ask directly:
```
"Send this for review"
"Request a peer review of these results"
"Share this with the team"
```

## Note on tool names
The actual tool names may be prefixed with a namespacing string. We will refer to the tool names without any prefix. Please take care to understand what tool is referenced.

## Mandatory pre-call checklist

Before calling `request_review`, you MUST have:

1. A completed SQL query (from a prior `run_query` call)
2. The user's email address — **ask explicitly, do not guess**
3. A one-sentence `sender_message` stating the user's intent (see below)
4. A detailed `agent_message` with your reasoning and `<assumptions>` block (see below)

## Getting the sender's email

Always ask:
> "What is your email address? This lets reviewers know who sent the request."
> "Is this your email: john.doe@acme.com? This lets reviewers know who sent the request."

Do not proceed without a real email address.

## Writing the sender_message

Write `sender_message` as a **single sentence** describing what the user wanted — the intent or reason behind this request.

Example: `"The user wanted to see monthly revenue by region for Q1."`

## Writing the agent_message

Write `agent_message` as flowing markdown prose covering your reasoning and key findings, followed by a tagged `<assumptions>` XML block listing every assumption made during the analysis:

```xml
<assumptions>
    <assumption reasoning="The user explicitly named this table." confidence=3>The `orders` table is the correct source for revenue data.</assumption>
    <assumption reasoning="Confirmed by checking the data — no nulls in 3 years of records." confidence=2>The `region` column is always populated for completed orders.</assumption>
    <assumption reasoning="No documentation found; inferred from column name." confidence=1>The `status = 'completed'` filter correctly excludes refunds.</assumption>
</assumptions>
```

Confidence: 3 = fact verified in data or stated by user; 2 = very likely but not certain; 1 = qualified guess.

## Formatting the SQL

Format the `sql` field with newlines for readability. Add inline `--` comment lines above each major clause (WITH blocks, JOINs, WHERE filters, GROUP BY, final SELECT) explaining what that part does in plain business language — not what the SQL does mechanically, but *why* it does it. Explain only the hard or unobvious parts, not everything.

## What the reviewer sees

In the Distinct web app, reviewers see:
- The SQL query (with the ability to re-run it)
- The data table (all rows)
- The `sender_message` from the sender
- Accept / Reject controls with an optional comment field

## Tool call reference

```python
request_review(
  sender_email="alice@company.com",   # required — always ask
  sql="-- Filter to Q1 completed orders\nSELECT ...",  # formatted with newlines + comments
  title="Q1 Revenue by Region",        # short descriptive title
  sender_message="The user wanted to see Q1 revenue broken down by region.",  # one sentence
  agent_message="I used the orders table...\n\n<assumptions>...</assumptions>",  # reasoning + assumptions
  columns=["region", "revenue"],
  rows=[["EMEA", 1200000], ...],       # up to 5 sample rows
  total_rows=12
)
```
