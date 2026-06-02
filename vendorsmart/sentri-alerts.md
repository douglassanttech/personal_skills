# Skill: Sentry Management Company - Invoice Workflow Plan

## Purpose
This skill is used whenever the user asks about:
- Sentry management company onboarding or implementation
- Sentry invoice workflow design, steps, rules, or alerts
- Mapping Sentry legacy alerts to VendorSmart workflow features
- Planning Sentry waves (Wave 1, 2, 3)
- Configuring tags, rules, thresholds, or approval steps for Sentry

---

## Context

**Sentry** is the largest management company in the US being onboarded to VendorSmart. Their legacy system uses an alert-based model with 52 alerts across multiple review queues. These alerts need to be mapped to VendorSmart's workflow system (steps, tags, rules, validations, and vendor/bank configurations).

---

## Alert Classification Summary

The 52 legacy alerts are classified into:

| Category | Count | Description |
|----------|-------|-------------|
| Workflow Steps | 8 core + 3 optional | Distinct approval stages with dedicated roles |
| Tags + Rules | 18 | Conditions resolved within existing steps via tags and rule configs |
| System Validations / Config | 9 | Automated checks on vendor, bank, or association data |
| Not Used / Discarded | 8 | Legacy-only features not migrated |

---

## WORKFLOW STEPS (actual approval stages)

### Core Steps (Wave 1)

| # | Step Name | Who Approves | Threshold | approvers | Section Permissions |
|---|-----------|-------------|-----------|-----------|-------------------|
| 1 | **Data Entry** | AP Clerk | — | 0 | EDIT: Invoice Data, Distribution |
| 2 | **Coding** | AP Clerk | — | 0 | EDIT: Distribution, Bank Info |
| 3 | **AP Review** | AP Supervisor | — | 1 | VIEW: all (review only) |
| 4 | **Manager Approval** | Property Manager | — | 1 | VIEW: all |
| 5 | **VP Approval** | Vice President | $10,000 | 1 | VIEW: all |
| 6 | **SVP Approval** | Sr. Vice President | $25,000 | 1 | VIEW: all |
| 7 | **EVP Approval** | Exec. Vice President | $100,000 | 1 | VIEW: all |
| 8 | **President Approval** | President | $250,000 | 1 | VIEW: all |

**Threshold behavior:** Steps 5-8 are auto-skipped (`SKIPPED_BY_THRESHOLD`) if the invoice amount is below the threshold. A $5,000 invoice goes: Data Entry → Coding → AP Review → Manager Approval → DONE. A $30,000 invoice adds SVP. A $150,000 adds EVP. A $300,000 goes all the way to President.

### Optional Steps (Wave 2+)

| # | Step Name | When Triggered | Wave |
|---|-----------|---------------|------|
| 9 | **Banking Review** | Bank account issues (closing, fraud, info required) | Wave 1-2 |
| 10 | **Cash Mgmt Approval** | Cash Poor / Manager Hold scenarios | Wave 1 |
| 11 | **Audit Review** | Random selection for additional audit | Wave 2 |

---

## Excessive Amount Thresholds (Invoice vs Transfer)

The legacy system has separate thresholds for regular invoices and transfer vouchers:

### Invoice Amount Thresholds
| Approval Level | Range |
|---------------|-------|
| VP | $10,000 – $24,999 |
| SVP | $25,000 – $99,999 |
| EVP | $100,000 – $249,999 |
| President | $250,000+ |

### Transfer Amount Thresholds (higher limits)
| Approval Level | Range |
|---------------|-------|
| VP | $100,000 – $199,999 |
| SVP | $200,000 – $249,999 |
| EVP | $250,000 – $499,999 |
| President | $500,000+ |

**Implementation:** Use workflow step config rules with `attribute: "invoiceType"` and `operation: "EQUAL"` to differentiate. Transfer vouchers get higher thresholds via separate rule configs per step.

---

## TAGS + RULES (conditions within steps, NOT separate steps)

### Wave 1 Tags

| Alert | Tag Name | Rule Behavior | Relevant Steps |
|-------|----------|--------------|----------------|
| **Duplicate Invoice Number** | `Duplicate` | System auto-tags → routes to AP Review queue | Data Entry, Manager, Payable |
| **Owner Reimbursement** | `Owner Reimbursement` | Consume vendor type from C3, apply tag → extra approval check at Manager step | Manager, Paid |
| **Purchase Only** | `Purchase Only` | Tag only, no special step needed | Data Entry, Manager |
| **Borrow From Reserves** | `Borrow From Reserves` | Manager manually adds tag → sends to separate approval queue | Manager, Paid |
| **Credit Account (non-ACH)** | `Credit Account` | Tag → AP Review. Credit memo feature needed (link invoices) | Data Entry, Manager |
| **Credit Account/ACH** | `Credit ACH` | $0 voucher amount + information added as notes | Data Entry, Manager, Paid |
| **ACH/Utility Back Dated** | `Back Dated` | Prepaid + insert payment date. Back date and future date supported. If on hold is prepaid, will process on release. Post on payment date | Data Entry, Manager, Paid |
| **Request for Cashier's Check** | `Cashier's Check` | Tag → requires Manager + Payable review | Manager, Payable, Paid |
| **Priority Invoice** | `Priority` | Tag → VP approval triggered | Paid |
| **One Time Pay** | `One Time Pay` | Tag → SVP approval (vendor missing insurance) | Manager |
| **Data Entry Level 1** | `DE Level 1 Review` | Rule: if keyed by Data Entry Level 1 user → additional Review queue | Data Entry, Manager, Payable |
| **Employee Reimbursement** | `Employee Reimbursement` | Tag → VP approval required | Data Entry, Manager |

### Wave 2 Tags

| Alert | Tag Name | Rule Behavior | Relevant Steps |
|-------|----------|--------------|----------------|
| **IRS Invoice Hold** | `IRS Hold` | Vendor record flag in C3 → auto-tag → AP Review. All payments must be approved before processing | Data Entry, Manager, Payable |
| **Restricted Vendor** | `Restricted Vendor` | Vendor flag from C3 → tag → review needed | Data Entry, Manager |
| **Vendor on Executive Hold** | `Executive Hold` | Vendor flag → tag → SVP/President review required | Data Entry, Manager, Payable |
| **Partial Pay** | `Partial Pay` | Feature: partially pay already-approved invoice. W1 workaround: void and recreate | Paid |
| **Pay by Phone** | `Pay by Phone` | Pay by phone → change to ACH after paid → system auto-updates to paid. "Set as paid" option | Paid |

### Status/Flow Tags (no wave, core behavior)

| Alert | Implementation | Behavior |
|-------|---------------|----------|
| **Rejected** | Invoice status = `REJECTED` | Rejection at any step → return to earlier step (Coding) |
| **Manager Hold** | Tag `On Hold` | Blocks workflow progression until released |

---

## Step Config Rules

### AP Review (Step 3) — Approval Configs

```
Config 1: Duplicate handling
  Rule: attribute="tag", operation="ANY_OF", value="Duplicate"
  Action: reasonRequired=true

Config 2: Credit account
  Rule: attribute="tag", operation="ANY_OF", value="Credit Account,Credit ACH"
  Action: reasonRequired=true

Config 3: Back dated
  Rule: attribute="tag", operation="ANY_OF", value="Back Dated"
  Action: reasonRequired=true
```

### Manager Approval (Step 4) — Approval Configs

```
Config 1: Borrow from reserves
  Rule: attribute="tag", operation="ANY_OF", value="Borrow From Reserves"
  Action: reasonRequired=true

Config 2: Owner/Employee reimbursement
  Rule: attribute="tag", operation="ANY_OF", value="Owner Reimbursement,Employee Reimbursement"
  Action: reasonRequired=true
```

### All Steps — Rejection Configs

```
Config: Rejection sends back to Coding
  configType: REJECTION
  reasonRequired: true
  targetStep: Step 2 (Coding)
  targetInvoiceStatus: PROCESSING
```

---

## SYSTEM VALIDATIONS / VENDOR & BANK CONFIG (not workflow)

These are NOT workflow steps. They are automated checks or configuration on master data.

### Vendor Validations (block or warn during invoice processing)

| Alert | Implementation | Behavior |
|-------|---------------|----------|
| **W9 Missing** | Vendor document validation | Warning displayed. Dismissed by uploading valid W9 |
| **Liability Insurance Missing** | Vendor document validation | Warning. Dismissed by uploading document |
| **Liability Insurance Expired** | Vendor document validation | Warning. Dismissed by uploading updated document |
| **Workers Comp Missing** | Vendor document validation | Warning. Dismissed by uploading document |
| **Workers Comp Expired** | Vendor document validation | Warning. Dismissed by uploading updated document |

### Bank Account Validations

| Alert | Implementation | Behavior |
|-------|---------------|----------|
| **Account Closing** | Bank account status = inactive | Don't display inactive accounts in dropdown (Wave 1) |
| **Fraud Warning** | Flag on bank account in C3 | Manual deactivation by Sentry. Visual alert (Wave 2) |
| **Closed Bank COA** | Bank account status check | Block usage of closed/inactive bank accounts |
| **Transaction Limit Met** | Bank account transaction limit validation | Block if limit exceeded. Allow selecting another account or future date |
| **Cash Poor** | Book balance < minimum balance (from C3) | Display minimum, block bank account from being used (Wave 1) |

### Association Validations

| Alert | Implementation | Behavior |
|-------|---------------|----------|
| **Canceled Property** | Association status check | Block or flag invoices for canceled properties |

---

## NOT USED / DISCARDED

These alerts are marked as "Not Used" in the legacy system and will NOT be migrated:

| Alert | Type | Reason |
|-------|------|--------|
| Credit Voucher | AP | Not Used — replaced by Credit Account alerts |
| Bank/Branch/Account Creation Required | Banking | Not Used |
| BOD Only Account | Banking | Not Used |
| Cash Management Creation Required | Banking | Not Used |
| Bank COA Does not cash checks | Banking | Not Used |
| COA Creation Required | GL | Not Used |
| Selection Property | Cash Mgmt | Not Used |
| Missing PAxtX File | AP | DL4 legacy system, Wave 3 at earliest |

---

## Proposed Workflow Template — Sentry Wave 1

```
Workflow: "Sentry Standard Workflow"
Management Company: Sentry

Step 1: Data Entry           (order: 1, approvers: 0, threshold: 0)
  - EDIT: INVOICE_DATA_SECTION, DISTRIBUTION_SECTION
  - VIEW: BANK_INFO_SECTION

Step 2: Coding               (order: 2, approvers: 0, threshold: 0)
  - EDIT: DISTRIBUTION_SECTION, BANK_INFO_SECTION
  - VIEW: INVOICE_DATA_SECTION

Step 3: AP Review            (order: 3, approvers: 1, threshold: 0)
  - VIEW: all sections
  - Handles: Duplicate, Credit, Back Dated, Cashier's Check tags

Step 4: Manager Approval     (order: 4, approvers: 1, threshold: 0)
  - VIEW: all sections
  - Handles: Borrow From Reserves, Owner Reimbursement tags

Step 5: VP Approval          (order: 5, approvers: 1, threshold: 10000)
  - VIEW: all sections
  - Auto-skip if invoice < $10,000

Step 6: SVP Approval         (order: 6, approvers: 1, threshold: 25000)
  - VIEW: all sections
  - Auto-skip if invoice < $25,000

Step 7: EVP Approval         (order: 7, approvers: 1, threshold: 100000)
  - VIEW: all sections
  - Auto-skip if invoice < $100,000

Step 8: President Approval   (order: 8, approvers: 1, threshold: 250000)
  - VIEW: all sections
  - Auto-skip if invoice < $250,000

Rejection at any step → back to Step 2 (Coding), reasonRequired: true
```

---

## Wave Roadmap

### Wave 1 (MVP)
- Core workflow (8 steps with thresholds)
- Tags: Duplicate, Owner Reimbursement, Purchase Only, Borrow From Reserves, Credit Account, Credit ACH, Back Dated, Cashier's Check
- Bank account validation (inactive accounts hidden)
- Cash Poor validation (minimum balance check from C3)
- Vendor document warnings (W9, insurance)

### Wave 2
- Audit Review step (random selection)
- IRS Invoice Hold (vendor flag from C3)
- Restricted Vendor (vendor flag from C3)
- Executive Hold (vendor flag → SVP/President)
- Fraud Warning (bank account flag)
- Partial Pay feature
- Pay by Phone → ACH conversion
- Transfer amount thresholds (separate rules by invoice type)

### Wave 3
- Banking Review step (if needed as separate step)
- PAxtX file integration (if DL4 sync needed)
- Additional automation and rule refinement

---

## Open Questions

1. **Banking Review & Cash Mgmt** — Are these separate workflow steps for Wave 1, or handled as system validations that block payment?
2. **Transfer vs Invoice thresholds** — Do we need separate workflow templates for transfers, or rules within the same workflow to apply different thresholds?
3. **Credit memo** — Is this a separate invoice type with linking, or just a tag + notes approach for Wave 1?
4. **Audit randomization** — What percentage of invoices are selected? Is this configurable per association?
5. **C3 vendor flags** — Which flags (IRS Hold, Restricted, Executive Hold) are available in C3 API for Wave 1?

---

## Instructions for Claude

When working on Sentry implementation:

1. **Always reference this plan** when discussing workflow design decisions
2. **Check the wave** before suggesting features — don't promise Wave 2/3 items in Wave 1
3. **Map legacy alert names** to VendorSmart concepts (step, tag, rule, validation)
4. **Use threshold-based steps** for the approval hierarchy (VP → SVP → EVP → President)
5. **Tags are the bridge** between legacy alerts and VendorSmart workflow rules
6. **Rejection always goes back to Coding** (Step 2) with reason required
7. **Vendor/bank validations are NOT workflow steps** — they are system-level checks
