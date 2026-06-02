# Skill: Setup Workflow for Management Company (Manco)

## Purpose

Use this skill whenever the user asks to:
- Set up a new management company in VendorSmart
- Onboard a new manco with C3 integration
- Configure the invoice email for a new manco
- Run the default user group script for a new manco
- Understand the full manco setup checklist

---

## ⚠️ Critical Warning

**You MUST run the `APDefaultGroupForMancos.sql` script on PROD as the last step.**
If you skip it, users accessing Business (Payables/Payments pages) will get a 4XX error and an "Insufficient Permissions" toast.

---

## Full Setup Checklist

Execute the steps **in this order**:

### Step 1 — Call the Setup Endpoint

Use `POST /management-company/setup` on `vsp-payments-api` (Swagger: `https://apiv2.vendorsmarttest.com/payments-api/docs#/Management%20Companies/setup`).

This endpoint performs steps 1–4 automatically:
- Inserts into `management_companies` and `accounting_system` tables in the `vsp-payments` DB
- Creates a VS AIS UUID for `accounting_system`
- Triggers the C3 sync for the manco

**Request body example:**
```json
{
  "managementCompanyId": "18991",
  "managementCompanyName": "Lone star",
  "tawnSqId": "123e4567-e89b-12d3-a456-426614174000",
  "accountingSystem": {
    "type": "C3",
    "c3Id": "100"
  },
  "hasPaymentEnablementFlag": false,
  "hasSetAccountingPaymentDateFeature": false,
  "paymentProvider": "PAYABLI",
  "enablementExpirationTime": "2880",
  "triggerC3Sync": true
}
```

> All field values (managementCompanyId, c3Id, etc.) are provided by the implementation team (Holy/Max) via ticket.

> If the manco already exists, the endpoint won't duplicate data — it will only trigger the C3 sync again.

---

### Step 2 — Create the Workflow (direct DB insert)

The setup endpoint does **not** create workflows. Must be inserted directly into the DB.

Use the script below as a base — paste it into ChatGPT with the ticket details to generate the specific version:

```sql
-- Pre configs
DECLARE @workflowId uniqueidentifier = NEWID();
DECLARE @workflowName nvarchar(MAX) = 'Adams Farm Community Association workflow';
DECLARE @mancoId nvarchar(MAX) = '75905';
DECLARE @workflowStepId uniqueidentifier = NEWID();

-- Insert into workflows table
INSERT INTO workflows (id, title, description, management_company_id)
VALUES (@workflowId, @workflowName, @workflowName, @mancoId);

-- Insert into workflow_steps table
INSERT INTO workflow_steps (id, workflow_id, [order], name, threshold_amount, approvers)
VALUES (@workflowStepId, @workflowId, 1, 'Invoice Coding', 0, 0);

-- Insert into section_permissions table
INSERT INTO section_permissions (workflow_step_id, section, permission)
VALUES
  (@workflowStepId, 'INVOICE_DATA_SECTION', 'EDIT'),
  (@workflowStepId, 'DISTRIBUTION_SECTION', 'VIEW'),
  (@workflowStepId, 'DISTRIBUTION_SECTION', 'EDIT'),
  (@workflowStepId, 'BANK_INFO_SECTION', 'VIEW'),
  (@workflowStepId, 'BANK_INFO_SECTION', 'EDIT');
```

The workflow structure (steps, approvers, thresholds) will be defined in the ticket.

---

### Step 3 — Update C3 Connector Parameter Store (tech debt)

The `/vsp-c3-connector/branches` parameter on AWS Parameter Store maps our mancos to C3 mancos.

**This is tech debt** — C3 connector should no longer use it, but until fixed:

1. Update `/vsp-c3-connector/branches` on Parameter Store with the new manco
2. Redeploy `vsp-c3-connector`
3. Trigger `POST /management-company/setup` again (won't duplicate, will re-invoke C3 sync)

---

### Step 4 — Create Invoice Upload Email (CDK)

The invoice upload email address must be created via CDK:

1. Add the new manco entry to `managmentCompanyMappingPerEnvironment` in `vs-aws-resources/src/stacks/vsp-invoice-email-processor.construct.ts`:

```typescript
[`newmanco@${invoicesDomain}`]: {
  id: <mancoId>,
  qrCodePlacement: 'AFTER',
}
```

2. Deploy the CDK stack to prod
3. Redeploy `vsp-invoice-email-processor`

---

### Step 5 — Run APDefaultGroupForMancos.sql on PROD ⚠️

**This is the last step and is mandatory.**

**Why:** When users access Business, they are auto-created on the VendorSmart side and assigned to a default user group. If this group doesn't exist, users can't access Payables (Invoices/Payments).

The script:
- Creates the default user group for the manco
- Enables all required permissions

**File:** `APDefaultGroupForMancos (1).sql` (ask the team for the latest version)

---

### Step 6 — Verify the Full Flow

After all steps above:

1. Create **two invoices**: one regular and one prepaid
2. Run **batch approval + generate checks** to confirm end-to-end flow
3. Check the **Open Payables report** (`Business > Reporting > All reports > Search for "Payable" > Accounts Payable - Open Items Report by Association - Open Periods`)
   - If invoices are **not** listed there → C3 integration is working correctly
4. Add a **DEMO association** so the team can test flows
   - Check if DEMO association has vendors; if not, ask C3 to add vendors for it

---

## Summary Table

| # | Step | Where | Notes |
|---|------|--------|-------|
| 1 | Call setup endpoint | `vsp-payments-api` | Inserts manco + accounting_system + triggers C3 sync |
| 2 | Insert workflow via SQL | `payables` DB (MSSQL) | Endpoint doesn't handle this — use base script + ChatGPT |
| 3 | Update Parameter Store `/vsp-c3-connector/branches` | AWS SSM | Tech debt; redeploy c3-connector after |
| 4 | Create invoice email via CDK | `vs-aws-resources` | Deploy to prod + redeploy `vsp-invoice-email-processor` |
| 5 | Run `APDefaultGroupForMancos.sql` on PROD | `payables` DB | **LAST STEP — mandatory** |
| 6 | Verify full flow | Business UI | 2 invoices + batch approval + Open Payables report + DEMO assoc |
