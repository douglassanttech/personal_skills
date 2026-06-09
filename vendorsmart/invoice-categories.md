# Skill: Invoice Categories & Delivery Instructions — Create for Management Company

## Purpose

Use this skill whenever the user asks to:
- Add invoice categories to a management company (manco)
- Add invoice delivery instructions to a management company (manco)
- Create/insert either of the above in QA or PROD
- Set up invoice category or delivery instruction options for a manco

---

## Background

Both `invoice_category` and `invoice_delivery_instruction` live in `dbo` schema of the `vsp_payments` database. They are fetched by the frontend via `GET /management-company/{mancoId}` and displayed only when the respective array is non-empty in the manco config. No frontend or backend code changes are needed — pure DB operations.

---

## DB Credentials

Load from env files at the repo root:

- **QA:** `source /Users/douglas/vendorsmart/PAYABLES_QA.env` → host `qa-vendorsmart.c0t1sxrdk6yr.us-east-1.rds.amazonaws.com`, DB `vsp_payments`
- **PROD:** `source /Users/douglas/vendorsmart/PAYABLES_PROD.env` → DB `vsp_payments`

---

## Steps

### 1 — Verify manco exists

```bash
source /Users/douglas/vendorsmart/PAYABLES_QA.env
sqlcmd -S "$DB_HOST" -U "$DB_USER_NAME" -P "$DB_PASSWORD" -d "vsp_payments" \
  -Q "SELECT id FROM dbo.management_companies WHERE id = '<mancoId>'"
```

### 2 — Check existing records (avoid duplicates)

**Categories:**
```sql
SELECT id, name FROM dbo.invoice_category
WHERE management_company_id = '<mancoId>' AND deleted_at IS NULL;
```

**Delivery instructions:**
```sql
SELECT id, name FROM dbo.invoice_delivery_instruction
WHERE management_company_id = '<mancoId>' AND deleted_at IS NULL;
```

### 3 — Insert

**Categories:**
```sql
INSERT INTO dbo.invoice_category (id, name, management_company_id, created_at, updated_at, created_by)
VALUES
  (NEWID(), 'Transfer',      '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'Reimbursement', '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'Sentry Charge', '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'Payroll',       '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'Purchase Only', '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'Manual Credit', '<mancoId>', GETDATE(), GETDATE(), 'system');
```

**Delivery instructions:**
```sql
INSERT INTO dbo.invoice_delivery_instruction (id, name, management_company_id, created_at, updated_at, created_by)
VALUES
  (NEWID(), 'No special instructions',                           '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'Notice of commencement',                            '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'Return to accounts receivable',                     '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'Return to risk management',                         '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'Return to community manager',                       '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'Return to requestor',                               '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'Return to customer experience',                     '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'Return to CPA roster',                              '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'FEDEXD - FedEx to division charge assn',            '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'FEDEXDD - FedEx to division charge division',       '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'FEDEXV - FedEx to vendor directly charge assn',     '<mancoId>', GETDATE(), GETDATE(), 'system'),
  (NEWID(), 'FEDEXVD - FedEx to vendor directly charge division','<mancoId>', GETDATE(), GETDATE(), 'system');
```

Adapt names to what the user specifies. Always run step 2 first to avoid duplicates.

### 4 — Verify

```sql
SELECT name, created_at FROM dbo.invoice_category
WHERE management_company_id = '<mancoId>' AND deleted_at IS NULL
ORDER BY created_at;

SELECT name, created_at FROM dbo.invoice_delivery_instruction
WHERE management_company_id = '<mancoId>' AND deleted_at IS NULL
ORDER BY created_at;
```

---

## Notes

- `id` → `uniqueidentifier`, use `NEWID()`
- `created_by` → default `'system'`
- `updated_by` → nullable, can be omitted
- `deleted_at` → soft-delete; always filter `deleted_at IS NULL` when querying
- For PROD inserts, confirm with the user before executing
