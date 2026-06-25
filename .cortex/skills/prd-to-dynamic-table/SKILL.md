---
name: prd-to-dynamic-table
description: "Parse PRD/requirements files (XLSX) and produce a structured implementation plan for a Snowflake Dynamic Table. Use when: onboarding new sources to a Silver-layer DT, reviewing column mappings from business stakeholders, or translating business rules into SQL transforms. Triggers: PRD, requirements, source onboarding, column mapping, dynamic table plan."
---

# PRD to Dynamic Table Plan

## When to Use

Use this skill when a user provides a PRD-style document (XLSX workbook) that describes:
- New source systems to onboard into an existing or new Dynamic Table
- Column-level mappings from source to target
- Business rules (status normalization, deduplication, data quality, etc.)

The goal is to produce a structured, reviewable implementation plan — NOT to execute DDL directly.

## Inputs

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prd_path` | Yes | Absolute path to the XLSX requirements file |
| `target_dynamic_table` | Yes | Fully-qualified target table name (e.g., `DB.SCHEMA.SILVER_AP_INVOICES`) |

## Workflow

### Step 1: Parse the PRD

Run the parser script to extract structured data from the XLSX workbook:

```bash
uv run .cortex/skills/prd-to-dynamic-table/scripts/parse_prd.py --input "<prd_path>"
```

The script reads all sheets and classifies them into three categories based on header patterns:
- **Source Onboarding** — new systems, contacts, delivery mechanisms
- **Column Mappings** — source-to-target field definitions with transforms
- **Business Rules** — normalization logic, dedup rules, data quality checks

If parsing fails or a sheet is unrecognizable, report the issue to the user rather than guessing structure.

**STOP**: Confirm with the user that parsing captured the expected number of sources, columns, and rules. Show a brief summary (source count, column count, rule count).

### Step 2: Catalog Lookup

Inspect the current state of the target Dynamic Table:

```bash
cortex search table-details "<target_dynamic_table>"
```

If the table exists, capture its current columns and types. If it doesn't exist yet, note that this is a greenfield build.

### Step 3: Analyze Changes

Compare the parsed PRD against the current table state. For each item, classify as:

| Classification | Meaning |
|----------------|---------|
| **Confirmed** | PRD explicitly states the mapping/rule and it is marked confirmed or approved |
| **Open** | PRD flags this as needing a decision (e.g., "NEEDS DECISION", "Open", "TBD") |
| **Assumption** | Not explicitly stated in the PRD — you are inferring from context |

Produce:
1. List of new source systems with delivery details
2. Column-level diff (new columns, type changes, new transforms)
3. Business rules that affect the Silver DT SQL (CASE statements, QUALIFY, WHERE clauses)
4. Items that affect layers OTHER than Silver (Gold normalization, DMFs, etc.) — note but exclude from the DT plan

### Step 4: Surface Open Questions

Present ALL items classified as "Open" or "Assumption" to the user via `ask_user_question`. Group them:

- **Decisions needed from business**: items the PRD flagged as open
- **Assumptions made by this analysis**: inferences not explicitly confirmed

Do NOT proceed to Step 5 until the user resolves or explicitly defers each item.

**STOP**: Wait for user decisions on open questions.

### Step 5: Produce Output

Generate the final structured output (see Output section below). Include a DDL sketch for the Dynamic Table showing the complete UNION ALL structure with all source branches.

## Best Practices: Surface Assumptions, Don't Guess

These rules are non-negotiable:

1. **If a column mapping says "Open" or "Needs Decision"** — surface it; do not pick a default.
2. **If a business rule contradicts another rule** — surface both; do not resolve the conflict.
3. **If a transform is ambiguous** (e.g., "Normalize" with no target format) — surface it with the specific question: "What is the target format?"
4. **If a source column has no explicit Silver target** — do NOT assume it should be dropped. Ask.
5. **If data types between sources conflict for the same target column** — surface the conflict with both types.

When in doubt: **stop and ask**. A wrong assumption silently baked into a DT is far more expensive than a 30-second clarification.

## Stopping Points

- After Step 1: Confirm parsing results with user
- After Step 4: User resolves open questions before DDL sketch is produced

## Output

The skill ALWAYS produces these sections, in this order:

### 1. New Source Systems

| Source System | Platform | Region | Delivery | Status |
|---------------|----------|--------|----------|--------|

### 2. Column Mapping Changes

| Target Column | Type | Source(s) | Transform | Status |
|---------------|------|-----------|-----------|--------|

Include only NEW or CHANGED columns relative to the current DT. Mark each as Confirmed/Open/Assumption.

### 3. Business Rules Affecting Silver

| Rule ID | Category | SQL Impact | Status |
|---------|----------|------------|--------|

Only rules that translate into SQL logic in the DT definition. Exclude rules that apply to Gold, DMFs, or downstream only.

### 4. Assumptions

Bulleted list of every inference made that is NOT explicitly confirmed in the PRD.

### 5. Open Questions

Bulleted list of every item that requires a human decision before implementation.

### 6. DDL Sketch

```sql
CREATE OR REPLACE DYNAMIC TABLE <target_dynamic_table>
  TARGET_LAG = ...
  WAREHOUSE = ...
AS
SELECT ...
FROM ...
UNION ALL
SELECT ...
FROM ...
```

This is a sketch, not production DDL. It shows structure and transforms but may need refinement after open questions are resolved.

## Example Usage

**User prompt:**
> Here's the updated PRD for AP invoices: `/workspaces/project/docs/ap_invoices_prd_v3.xlsx`
> Target table is `COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES`.

**Skill invocation:**
1. Parse → finds 2 new sources (Baan, Workday), 30 column mappings, 10 business rules
2. Catalog lookup → DT exists with SAP + Oracle branches (17 columns)
3. Analyze → 4 new columns needed, 2 status mappings, 1 dedup rule, 2 open questions
4. Surface → asks user about payment terms normalization layer and cost center format handling
5. Output → full structured plan with DDL sketch showing 4-branch UNION ALL

## Notes

- The parser script supports XLSX only (via openpyxl). CSV fallback is manual — read the files directly.
- The DDL sketch uses `TARGET_LAG = DOWNSTREAM` unless the PRD specifies otherwise.
- Surrogate key pattern: `MD5(SOURCE_SYSTEM || '|' || INVOICE_ID)` — consistent with existing Silver tables.
