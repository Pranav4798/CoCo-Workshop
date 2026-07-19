# PRD-Driven Update: SILVER_AP_INVOICES — Baan + Workday Onboarding

## Overview

This document captures all artifacts from the PRD-driven update to `COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES`, adding Baan IV and Workday Financial Management as source systems.

---

## 1. Input PRD Files

| File | Purpose |
|------|---------|
| `sample_business_requirements_source_onboarding.csv` | New source system requests (Baan IV, Workday) with contacts, delivery details, and status |
| `sample_business_requirements_column_mapping.csv` | Column-level source-to-Silver mappings with transforms and status flags |
| `sample_business_requirements_business_rules.csv` | Business rules (BR-001 through BR-010) governing normalization, dedup, and data quality |

---

## 2. Decisions Made

| Topic | Decision | Rationale |
|-------|----------|-----------|
| Payment terms normalization (BR-005) | Normalize at Silver | Maintains consistency with existing SAP/Oracle behavior already in the DT |
| TARGET_LAG | Changed from `'1 hour'` to `DOWNSTREAM` | Per BR-009; Silver refreshes on-demand from Gold |

---

## 3. Deployed DDL

```sql
CREATE OR REPLACE DYNAMIC TABLE COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
  TARGET_LAG = DOWNSTREAM
  WAREHOUSE = COCO_WORKSHOP_WH
AS
SELECT
    MD5(SOURCE_SYSTEM || '|' || SOURCE_INVOICE_ID) AS INVOICE_SK,
    SOURCE_SYSTEM,
    SOURCE_INVOICE_ID,
    INVOICE_NUMBER,
    VENDOR_ID,
    VENDOR_NAME,
    INVOICE_DATE,
    DUE_DATE,
    INVOICE_AMOUNT,
    CURRENCY_CODE,
    PAYMENT_TERMS_RAW,
    CASE
        WHEN PAYMENT_TERMS_RAW IN ('NET30', 'N30', 'Net 30') THEN 'NET30'
        WHEN PAYMENT_TERMS_RAW IN ('NET60', 'N60', 'Net 60') THEN 'NET60'
        ELSE PAYMENT_TERMS_RAW
    END AS PAYMENT_TERMS,
    PO_NUMBER,
    LINE_DESCRIPTION,
    GL_ACCOUNT,
    COST_CENTER,
    APPROVAL_STATUS_RAW,
    CASE
        WHEN APPROVAL_STATUS_RAW IN ('APPROVED', 'VALIDATED', 'POSTED', 'Approved') THEN 'APPROVED'
        WHEN APPROVAL_STATUS_RAW IN ('PENDING', 'In Review') THEN 'PENDING'
        ELSE APPROVAL_STATUS_RAW
    END AS APPROVAL_STATUS,
    CREATED_AT
FROM (
    SELECT
        'SAP'                AS SOURCE_SYSTEM,
        INVOICE_ID           AS SOURCE_INVOICE_ID,
        INVOICE_NUMBER,
        VENDOR_ID,
        VENDOR_NAME,
        INVOICE_DATE,
        DUE_DATE,
        INVOICE_AMOUNT,
        CURRENCY_CODE,
        PAYMENT_TERMS        AS PAYMENT_TERMS_RAW,
        PO_NUMBER,
        LINE_DESCRIPTION,
        GL_ACCOUNT,
        COST_CENTER,
        APPROVAL_STATUS      AS APPROVAL_STATUS_RAW,
        CREATED_AT
    FROM COCO_WORKSHOP.SOURCE_DATA.BRONZE_SAP_AP_INVOICES

    UNION ALL

    SELECT
        'ORACLE'             AS SOURCE_SYSTEM,
        INV_ID               AS SOURCE_INVOICE_ID,
        INV_NUM              AS INVOICE_NUMBER,
        SUPPLIER_ID          AS VENDOR_ID,
        SUPPLIER_NAME        AS VENDOR_NAME,
        INV_DATE             AS INVOICE_DATE,
        PAYMENT_DUE_DATE     AS DUE_DATE,
        TOTAL_AMOUNT         AS INVOICE_AMOUNT,
        CURRENCY             AS CURRENCY_CODE,
        TERMS_CODE           AS PAYMENT_TERMS_RAW,
        PURCHASE_ORDER       AS PO_NUMBER,
        DESCRIPTION          AS LINE_DESCRIPTION,
        ACCOUNT_CODE         AS GL_ACCOUNT,
        DEPT_CODE            AS COST_CENTER,
        STATUS               AS APPROVAL_STATUS_RAW,
        CREATION_DATE        AS CREATED_AT
    FROM COCO_WORKSHOP.SOURCE_DATA.BRONZE_ORACLE_AP_INVOICES

    UNION ALL

    SELECT
        'BAAN'               AS SOURCE_SYSTEM,
        BAN_INVOICE_ID       AS SOURCE_INVOICE_ID,
        BAN_INVOICE_REF      AS INVOICE_NUMBER,
        BAN_VENDOR_CODE      AS VENDOR_ID,
        BAN_VENDOR_DESC      AS VENDOR_NAME,
        BAN_INV_DATE         AS INVOICE_DATE,
        BAN_PAY_DATE         AS DUE_DATE,
        BAN_AMOUNT           AS INVOICE_AMOUNT,
        BAN_CURR             AS CURRENCY_CODE,
        BAN_PAY_TERMS        AS PAYMENT_TERMS_RAW,
        BAN_PO_REF           AS PO_NUMBER,
        BAN_LINE_DESC        AS LINE_DESCRIPTION,
        BAN_GL_CODE          AS GL_ACCOUNT,
        BAN_COST_CTR         AS COST_CENTER,
        BAN_STATUS           AS APPROVAL_STATUS_RAW,
        BAN_CREATED          AS CREATED_AT
    FROM COCO_WORKSHOP.SOURCE_DATA.BRONZE_BAAN_AP_INVOICES
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY BAN_INVOICE_REF
        ORDER BY BAN_CREATED DESC
    ) = 1

    UNION ALL

    SELECT
        'WORKDAY'            AS SOURCE_SYSTEM,
        WD_INVOICE_ID        AS SOURCE_INVOICE_ID,
        WD_INVOICE_NUM       AS INVOICE_NUMBER,
        WD_SUPPLIER_ID       AS VENDOR_ID,
        WD_SUPPLIER_NAME     AS VENDOR_NAME,
        WD_INVOICE_DATE      AS INVOICE_DATE,
        WD_DUE_DATE          AS DUE_DATE,
        WD_AMOUNT            AS INVOICE_AMOUNT,
        WD_CURRENCY          AS CURRENCY_CODE,
        WD_PAY_TERMS         AS PAYMENT_TERMS_RAW,
        WD_PO_NUMBER         AS PO_NUMBER,
        WD_MEMO              AS LINE_DESCRIPTION,
        WD_LEDGER_ACCOUNT    AS GL_ACCOUNT,
        WD_COST_CENTER       AS COST_CENTER,
        WD_APPROVAL_STATUS   AS APPROVAL_STATUS_RAW,
        WD_CREATED_DATE      AS CREATED_AT
    FROM COCO_WORKSHOP.SOURCE_DATA.BRONZE_WORKDAY_AP_INVOICES
);
```

---

## 4. Validation Queries

Run these after deploying to confirm correctness:

```sql
-- Row count by source (expect SAP:15, Oracle:15, Baan:≤10, Workday:10)
SELECT SOURCE_SYSTEM, COUNT(*) AS CNT
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM
ORDER BY SOURCE_SYSTEM;

-- Payment terms fully normalized (expect only NET30, NET60)
SELECT DISTINCT PAYMENT_TERMS
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES;

-- Approval status fully normalized (expect only APPROVED, PENDING)
SELECT DISTINCT APPROVAL_STATUS
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES;

-- No unexpected approval status values
SELECT APPROVAL_STATUS, COUNT(*)
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
WHERE APPROVAL_STATUS NOT IN ('APPROVED', 'PENDING')
GROUP BY APPROVAL_STATUS;

-- Baan dedup working (expect 0 rows)
SELECT INVOICE_NUMBER, COUNT(*) AS DUPES
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
WHERE SOURCE_SYSTEM = 'BAAN'
GROUP BY INVOICE_NUMBER
HAVING COUNT(*) > 1;

-- No NULL surrogate keys
SELECT COUNT(*) AS NULL_SKS
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
WHERE INVOICE_SK IS NULL;

-- No orphan source system values
SELECT DISTINCT SOURCE_SYSTEM
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
WHERE SOURCE_SYSTEM NOT IN ('SAP', 'ORACLE', 'BAAN', 'WORKDAY');
```

---

## 5. Assumptions Requiring Engineering Review

1. **TARGET_LAG = DOWNSTREAM** — changed from 1 hour. Silver now only refreshes when downstream (Gold) requests it. Confirm this is acceptable for alerting/monitoring use cases.
2. **QUALIFY dedup is Baan-only** — if other sources develop duplicate issues, additional clauses are needed.
3. **Case-sensitive approval status matching** — Workday sends `'Approved'` and `'In Review'` (mixed case). If source casing varies, consider wrapping in `UPPER()`.
4. **Baan cost center mixed formats** — both `BC-XX` and `BC-XXX` pass through unchanged. Downstream consumers must handle both.
5. **Workday legal signoff pending** — DPA-2025-0041 still in review. The branch is live but data is already loaded in Bronze. Remove the branch if premature.

---

## 6. PRD Evaluator Skill (Reuse Instructions)

The project skill lives at:

```
.cortex/skills/prd-to-dynamic-table/
├── SKILL.md              # Workflow, inputs, outputs, stopping points
└── scripts/
    └── parse_prd.py      # XLSX parser (uv run, PEP 723 inline deps)
```

### To reuse for the next PRD update:

1. Prepare an XLSX workbook with sheets for source onboarding, column mappings, and business rules (the parser detects sheets by header patterns — no fixed naming required).
2. Run:
   ```
   Parse <path-to-prd.xlsx> and plan changes to <DB.SCHEMA.TARGET_TABLE>.
   ```
3. The skill will: parse → catalog lookup → analyze → surface open questions → produce DDL sketch.

### For CSV inputs (like this session):

The parser only handles XLSX. For CSV, read the files directly and follow the same 5-step workflow manually (the skill documents the full process in SKILL.md).

---

## 7. Business Rules Reference

| Rule ID | Summary | Applies To |
|---------|---------|------------|
| BR-001 | Status normalization (CASE in Silver) | Silver DT |
| BR-002 | No FX conversion at Silver | Gold only |
| BR-003 | Baan dedup on INVOICE_NUMBER | Silver DT (Baan branch) |
| BR-004 | Flag invoices > $500K | DMF / Guardrails Pack |
| BR-005 | Payment terms normalization | Silver DT (decided) |
| BR-006 | GL codes stored as-is | Silver DT (no action) |
| BR-007 | SOURCE_SYSTEM literal per branch | Silver DT |
| BR-008 | Drop system-specific columns | Silver DT |
| BR-009 | TARGET_LAG = DOWNSTREAM | Silver DT config |
| BR-010 | Data retention / history | Phase 2 backlog |
