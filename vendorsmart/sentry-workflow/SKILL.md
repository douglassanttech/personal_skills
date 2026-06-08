# Skill: Sentry Workflow — Setup, Debug & Prod Operations

## Purpose

Use this skill when:
- Creating or recreating the Sentry invoice workflow in any environment
- Debugging workflow errors for Sentry (mc 18991)
- Fixing section permissions or step configurations in QA/PROD DB
- Planning Sentry wave rollout and alert mapping

---

## Environment IDs

| Env | Management Company ID | QA Workflow ID |
|-----|----------------------|----------------|
| QA | `18991` | `a1b2c3d4-e5f6-7890-abcd-ef1234567890` (title: "Sentry Workflow") |
| PROD | TBD — not yet deployed | — |

**DB connections:**
- QA: `PAYABLES_QA.env` → `qa-vendorsmart.c0t1sxrdk6yr.us-east-1.rds.amazonaws.com`, DB `payables`
- PROD: `PAYABLES_PROD.env` → `prod-vendorsmart.c0t1sxrdk6yr.us-east-1.rds.amazonaws.com`, DB `payables`

---

## Workflow Design (10 Steps)

```
IN → 1.Data Entry → 2.Coding → 3.Manager Approval → 4.AP Review
   → 5.VP → 6.SVP → 7.EVP → 8.President → 9.Alert Review → 10.Cash Management → ✓ Approved
```

| # | Step Name | Role | approvers | Threshold | Notes |
|---|-----------|------|-----------|-----------|-------|
| 1 | Data Entry | AP Clerk | 0 | — | Entry step, no approval required |
| 2 | Coding | AP Clerk | 0 | — | GL distribution + bank info |
| 3 | Manager Approval | Property Manager | 1 | — | |
| 4 | AP Review | AP Supervisor | 1 | — | |
| 5 | VP Approval | Vice President | 1 | $10,000 | Auto-skip below threshold |
| 6 | SVP Approval | Sr. Vice President | 1 | $25,000 | Auto-skip below threshold |
| 7 | EVP Approval | Exec. Vice President | 1 | $100,000 | Auto-skip below threshold |
| 8 | President Approval | President | 1 | $250,000 | Auto-skip below threshold |
| 9 | Alert Review | System (auto) | 0 | — | Blocking gate — auto-advances when all work queues cleared |
| 10 | Cash Management | Cash Manager | 1 | — | |

**Step 9 (Alert Review):** No manual approval. System auto-advances when all active work queue tags are resolved. Blocking gate between President and Cash Management.

**Rejection:** Any step → back to Step 2 (Coding), `reasonRequired: true`.

---

## Section Permissions per Step

This is the **correct configuration** after the 2025-06-08 fix. `INVOICE_PREFERENCES_SECTION` must be **VIEW only** (not EDIT) in all steps until Payment Instructions feature is shipped.

| Step | INVOICE_DATA | DISTRIBUTION | BANK_INFO | INVOICE_PREFERENCES |
|------|-------------|--------------|-----------|---------------------|
| 1 Data Entry | EDIT | EDIT | VIEW | VIEW |
| 2 Coding | VIEW | EDIT | EDIT | VIEW |
| 3 Manager Approval | VIEW | VIEW | VIEW | VIEW |
| 4 AP Review | VIEW | VIEW | VIEW | VIEW |
| 5 VP Approval | VIEW | VIEW | VIEW | VIEW |
| 6 SVP Approval | VIEW | VIEW | VIEW | VIEW |
| 7 EVP Approval | VIEW | VIEW | VIEW | VIEW |
| 8 President Approval | VIEW | VIEW | VIEW | VIEW |
| 9 Alert Review | VIEW | VIEW | VIEW | VIEW |
| 10 Cash Management | VIEW | VIEW | VIEW | VIEW |

> **Note:** `INVOICE_PREFERENCES_SECTION` does not yet have a UI component or event mapping in the backend (`_getSectionsFilled` in `workflow-execution.ts:283`). Setting it to EDIT in any step will immediately block the workflow with a 400 error. Do NOT set EDIT on this section until Payment Instructions feature is implemented.

---

## Known Bug & Fix (2025-06-08)

**Bug:** `400 — "The following sections are required: INVOICE_PREFERENCES_SECTION"` during Data Entry step advancement.

**Root cause:**
1. `workflow-execution.ts:283` — `_getSectionsFilled()` maps domain events → filled sections. `INVOICE_PREFERENCES_SECTION` is not in the map (no event fires for it yet).
2. The Sentry workflow in QA had `INVOICE_PREFERENCES_SECTION` with `permission = 'EDIT'` on Data Entry and Coding steps.
3. `getRequiredSections()` returns all sections with `EDIT` permission → backend blocks because the section is never filled.

**Fix applied to QA:**
```sql
UPDATE section_permissions
SET permission = 'VIEW', updated_at = GETUTCDATE()
WHERE workflow_step_id IN (
  'a1b2c3d4-e5f6-7890-abcd-ef1234567891',  -- Data Entry
  'a1b2c3d4-e5f6-7890-abcd-ef1234567892'   -- Coding
)
  AND section = 'INVOICE_PREFERENCES_SECTION'
  AND permission = 'EDIT';
```

**If this happens in PROD**, run same query replacing step IDs with the prod step IDs for Data Entry and Coding:
```sql
-- Find prod step IDs first:
SELECT ws.id, ws.name, ws.[order]
FROM workflow_steps ws
JOIN workflows w ON ws.workflow_id = w.id
WHERE w.management_company_id = '18991'
  AND ws.[order] IN (1, 2)
ORDER BY ws.[order];

-- Then fix:
UPDATE section_permissions
SET permission = 'VIEW', updated_at = GETUTCDATE()
WHERE workflow_step_id IN ('<data_entry_id>', '<coding_id>')
  AND section = 'INVOICE_PREFERENCES_SECTION'
  AND permission = 'EDIT';
```

---

## DB Schema (relevant tables)

```sql
workflows          — id, management_company_id, title
workflow_steps     — id, workflow_id, name, [order], approvers, threshold_amount
section_permissions — id, workflow_step_id, section, permission
workflow_step_configs — step configs (approval rules, rejection rules)
workflow_step_config_rules — conditions for configs
```

**section values:** `INVOICE_DATA_SECTION`, `DISTRIBUTION_SECTION`, `BANK_INFO_SECTION`, `INVOICE_PREFERENCES_SECTION`
**permission values:** `EDIT`, `VIEW`

---

## Work Queues (21 total — tag-triggered tasks that block step progression)

### Step 3 — Manager Approval (2 queues)
| Queue | Tag | Origin |
|-------|-----|--------|
| Owner Reimbursement | `Owner Reimbursement` | C3 — vendor type = reimbursement |
| Borrow From Reserves | `Borrow From Reserves` | MANUAL — PM adds tag when approving |

### Step 4 — AP Review (7 queues)
| Queue | Tag | Origin |
|-------|-----|--------|
| Duplicate | `Duplicate` | AUTO — same invoice# for same vendor+community |
| Credit Account | `Credit Account` | MANUAL — AP Clerk at data entry (non-ACH credit) |
| Credit ACH | `Credit ACH` | AUTO — voucher amount = $0 |
| DE Level 1 Review | `DE Level 1 Review` | AUTO — data entry user is Level 1 role |
| IRS Invoice Hold | `IRS Hold` | C3 — vendor IRS hold flag |
| Restricted Vendor | `Restricted Vendor` | C3 — vendor restricted flag |
| Canceled Property | `Canceled Property` | AUTO — community status = canceled |

### Step 10 — Cash Management (12 queues)
| Queue | Tag | Origin |
|-------|-----|--------|
| Cash Poor | `Cash Poor` | C3 — book balance < minimum balance |
| Manager Hold | `On Hold` | MANUAL |
| Cashier's Check | `Cashier's Check` | MANUAL — AP Clerk at data entry |
| Account Closing | `Account Closing` | C3 — bank account status inactive |
| Additional Banking Info | `Additional Banking Info` | MANUAL |
| Closed Bank COA | `Closed Bank COA` | C3 — bank COA inactivated |
| Fraud Warning | `Fraud Warning` | C3 — fraud flag on bank account |
| W9 Missing | `W9 Missing` | VALIDATION |
| Liability Insurance Missing | `Liability Insurance Missing` | VALIDATION |
| Liability Insurance Expired | `Liability Insurance Expired` | VALIDATION |
| Workers Comp Missing | `Workers Comp Missing` | VALIDATION |
| Workers Comp Expired | `Workers Comp Expired` | VALIDATION |

---

## Routing Rules (7 tags that force steps even below threshold)

| Tag | Forces Step |
|-----|-------------|
| `Employee Reimbursement` | VP Approval (Step 5) |
| `VP Approval Required` | VP Approval (Step 5) |
| `Priority` | VP Approval (Step 5) |
| `One Time Pay` | SVP Approval (Step 6) |
| `SVP Approval Required` | SVP Approval (Step 6) |
| `Executive Hold` | SVP Approval (Step 6) |
| `Back Dated` | AP Review (Step 4) |

---

## Step Config Rules

### Manager Approval (Step 3)
```
Config 1: tag ANY_OF "Borrow From Reserves" → reasonRequired=true
Config 2: tag ANY_OF "Owner Reimbursement,Employee Reimbursement" → reasonRequired=true
```

### AP Review (Step 4)
```
Config 1: tag ANY_OF "Duplicate" → reasonRequired=true
Config 2: tag ANY_OF "Credit Account,Credit ACH" → reasonRequired=true
Config 3: tag ANY_OF "Back Dated" → reasonRequired=true
```

### All Steps — Rejection
```
configType: REJECTION, reasonRequired: true, targetStep: Coding (Step 2), targetInvoiceStatus: PROCESSING
```

---

## Transfer Amount Thresholds (higher limits for transfer vouchers)

| Level | Invoice | Transfer |
|-------|---------|----------|
| VP | $10,000+ | $100,000+ |
| SVP | $25,000+ | $200,000+ |
| EVP | $100,000+ | $250,000+ |
| President | $250,000+ | $500,000+ |

Implementation: workflow step config rule with `attribute="invoiceType"` and `operation="EQUAL"` to apply different thresholds.

---

## Alert Classification (52 Legacy CP2 Alerts)

| Category | Count |
|----------|-------|
| Work Queues | 21 |
| Routing Rules | 7 |
| Threshold Rules | 9 |
| Discarded (Not Used in legacy) | 8 |
| Need Discovery | 6 |

**Discarded (do not migrate):**
Credit Voucher, Bank/Branch/Account Creation Required, Bank COA Does Not Cash Checks, BOD Only Account, Cash Management Creation Required, Transaction Limit Met, Selection Property, COA Creation Required.

**Need Discovery:**
ACH/Utility Back Dated (L.3), Missing PAxtX File (L.10, Wave 3), Partial Pay (L.12, Wave 2), Pay by Phone (L.13, Wave 2), Purchase Only (L.14), Rejected (L.42, core).

---

## Wave Roadmap

### Wave 1 (current)
- 10-step workflow with thresholds
- Work queues: all 21 listed above
- Routing rules: all 7 listed above
- Vendor document validation (W9, insurance, workers comp)
- Cash Poor / bank account validation from C3

### Wave 2
- Transfer amount thresholds (separate rules by invoice type)
- IRS Invoice Hold (vendor flag from C3)
- Restricted Vendor (vendor flag from C3)
- Executive Hold (vendor flag)
- Partial Pay feature
- Pay by Phone → ACH conversion

### Wave 3
- PAxtX file integration (DL4 sync)

---

## Instructions for Claude

1. `INVOICE_PREFERENCES_SECTION` must NEVER be set to EDIT — it will break the workflow until Payment Instructions feature ships
2. Step order is: Manager Approval (3) **before** AP Review (4) — confirmed in discovery
3. Step 9 (Alert Review) has NO manual approval — auto-advances only
4. Rejection always goes back to Coding (Step 2) with reason required
5. When debugging section errors: query `section_permissions` joined to `workflow_steps` to verify EDIT vs VIEW
6. Always verify prod step IDs before running DB fixes — they differ from QA
7. Tag origins: AUTO (system), C3 (synced from C3), VALIDATION (vendor docs), MANUAL (user-applied)
