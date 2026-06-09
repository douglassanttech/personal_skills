# Skill: Invoice Categories — Create for Management Company

## Purpose

Use this skill whenever the user asks to:
- Add invoice categories to a management company (manco)
- Create/insert invoice categories in QA or PROD
- Set up invoice category options for a manco

---

## Background

Invoice categories are stored in `dbo.invoice_category` in the `vsp_payments` database. They are fetched by the frontend via `GET /management-company/{mancoId}` and displayed only when `mancoConfig.invoiceCategories.length > 0`. No frontend or backend code changes are needed — it's a pure DB operation.

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

### 2 — Check existing categories (avoid duplicates)

```sql
SELECT id, name
FROM dbo.invoice_category
WHERE management_company_id = '<mancoId>'
  AND deleted_at IS NULL;
```

### 3 — Insert new categories

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

Adapt the category names to what the user specifies. Always run step 2 first to avoid inserting duplicates.

### 4 — Verify

```sql
SELECT id, name, created_at
FROM dbo.invoice_category
WHERE management_company_id = '<mancoId>'
  AND deleted_at IS NULL
ORDER BY created_at;
```

---

## Notes

- `id` → `uniqueidentifier`, use `NEWID()`
- `created_by` → default `'system'`
- `updated_by` → nullable, can be omitted
- `deleted_at` → soft-delete column; always filter `deleted_at IS NULL` when querying
- For PROD inserts, confirm with the user before executing
