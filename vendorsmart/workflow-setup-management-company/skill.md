# Skill: Setup Workflow for Management Company

## Purpose
Use this skill when the user asks to:
- Create a complete workflow SQL for a new management company
- Define steps, thresholds, section permissions, and step configs for a workflow template
- Onboard a management company with an invoice approval workflow
- Understand what tables need to be populated for a workflow to work end-to-end

---

## Full Workflow Setup Checklist

A complete workflow for a management company requires populating these tables in order:

| # | Table | What goes here |
|---|-------|---------------|
| 1 | `workflows` | One row: the workflow template |
| 2 | `workflow_steps` | One row per step (ordered) |
| 3 | `section_permissions` | Which invoice sections are VIEW/EDIT per step |
| 4 | `workflow_step_configs` | Rejection/approval rules at template level (optional but recommended) |
| 5 | `workflow_step_config_rules` | Conditions for each config (tag-based, amount-based, etc.) |

After running the workflow SQL, each association still needs a `workflow_assignment` + `workflow_step_responsibles` (see skill: `workflow-assign-association`).

---

## Table Schemas

### workflows
| Column | Type | Notes |
|--------|------|-------|
| `id` | uniqueidentifier | Static UUID — reuse in step configs |
| `title` | nvarchar | Display name |
| `description` | nvarchar | Short description |
| `management_company_id` | nvarchar(50) | FK to management company |
| `has_cash_management` | bit | 1 if MC uses cash management step logic |

### workflow_steps
| Column | Type | Notes |
|--------|------|-------|
| `id` | uniqueidentifier | Static UUID — reuse in section_permissions and configs |
| `workflow_id` | uniqueidentifier | FK to `workflows.id` |
| `order` | int | 1-based, sequential |
| `name` | nvarchar | Display name |
| `threshold_amount` | decimal | Auto-skip if invoice ≤ this. 0 = no skip |
| `approvers` | int | 0 = data entry step, ≥1 = approval step |

### section_permissions
| Column | Type | Notes |
|--------|------|-------|
| `workflow_step_id` | uniqueidentifier | FK to `workflow_steps.id` |
| `section` | nvarchar | `INVOICE_DATA_SECTION`, `DISTRIBUTION_SECTION`, `BANK_INFO_SECTION`, `INVOICE_PREFERENCES_SECTION` |
| `permission` | nvarchar | `VIEW` or `EDIT` |

Data entry steps typically get both VIEW+EDIT on all sections. Approval steps get VIEW only.

### workflow_step_configs
| Column | Type | Notes |
|--------|------|-------|
| `id` | uniqueidentifier | Static UUID |
| `workflow_step_id` | uniqueidentifier | FK to `workflow_steps.id` |
| `workflow_assignment_id` | uniqueidentifier | NULL = template-level (applies to all assignments) |
| `order` | int | Evaluation order within the step |
| `config_type` | nvarchar | `REJECTION` or `APPROVAL` |
| `reason_required` | bit | 1 = user must type a reason |
| `target_step_id` | uniqueidentifier | Where to route (for rejections: usually Coding step) |
| `target_invoice_status` | nvarchar | e.g. `PROCESSING`, `REJECTED` |
| `target_invoice_tags` | nvarchar | Tags to apply (comma-separated) |
| `tag_action_type` | nvarchar | `ADD_TAG` |

### workflow_step_config_rules
| Column | Type | Notes |
|--------|------|-------|
| `id` | uniqueidentifier | Static UUID |
| `workflow_step_config_id` | uniqueidentifier | FK to `workflow_step_configs.id` |
| `attribute` | nvarchar | `tag`, `amount`, `invoiceType`, `vendorId`, `associationId`, `glAccount` |
| `operation` | nvarchar | `ANY_OF`, `NOT_ANY_OF`, `ALL_OF`, `EQUAL`, `NOT_EQUAL`, `GREATER_THAN`, `LESS_THAN`, `GREATER_THAN_OR_EQUAL`, `LESS_THAN_OR_EQUAL`, `DIFFERENT_FROM` |
| `value` | nvarchar | For `ANY_OF`: comma-separated tag names. For amounts: numeric string. |

**All rules in a config must match** for the config to apply (AND logic between rules).

---

## Common Patterns

### Step types
- **Data Entry**: `approvers = 0`, threshold = 0, full EDIT permissions
- **Approval Step**: `approvers = 1` (usually), threshold = 0 or amount, VIEW-only permissions
- **Threshold Step**: approval step with `threshold_amount > 0` — auto-skipped when invoice ≤ threshold

### Standard rejection config (all steps)
```sql
-- No rules = applies unconditionally on every rejection
-- Returns to Coding step, reason required, status = PROCESSING
INSERT INTO workflow_step_configs (id, [order], workflow_step_id, config_type, reason_required, target_step_id, target_invoice_status)
VALUES (@rejCfgId, 1, @stepId, 'REJECTION', 1, @codingStepId, 'PROCESSING');
```

### Tag-based approval config
```sql
-- Config: requires reason when tag matches
INSERT INTO workflow_step_configs (id, [order], workflow_step_id, config_type, reason_required)
VALUES (@aprCfgId, 1, @stepId, 'APPROVAL', 1);

-- Rule: tag is any of the listed values
INSERT INTO workflow_step_config_rules (id, workflow_step_config_id, attribute, operation, value)
VALUES (@ruleId, @aprCfgId, 'tag', 'ANY_OF', 'Tag Name A,Tag Name B');
```

### Routing rules (force steps below threshold)
The `workflow_step_configs` schema only supports `APPROVAL`/`REJECTION` types — there is no "force step" type. Tag-based routing rules that force a step to not be skipped must be configured **per assignment via the API** after the assignment is created. Document them as comments in the SQL.

---

## SQL File Structure

```sql
-- Header comment: MC name, step summary, notes on routing rules
DECLARE @managementCompanyId ...
DECLARE @workflowId          ...
DECLARE @step1Id .. @stepNId ...
DECLARE @rejCfg1Id .. @rejCfgNId ...
DECLARE @aprCfgXId ...
DECLARE @ruleXId ...

BEGIN TRANSACTION;
BEGIN TRY
  -- 1. INSERT INTO workflows
  -- 2. INSERT INTO workflow_steps
  -- 3. INSERT INTO section_permissions (per step)
  -- 4. INSERT INTO workflow_step_configs (rejections, then approval configs)
  -- 5. INSERT INTO workflow_step_config_rules
  COMMIT TRANSACTION;
END TRY
BEGIN CATCH
  ROLLBACK TRANSACTION;
  PRINT ERROR_MESSAGE();
END CATCH;

-- ROLLBACK SCRIPT (commented out):
-- DELETE workflow_step_config_rules → workflow_step_configs → section_permissions → workflow_steps → workflows
```

---

## Townsquare Integration — UAT aponta para PROD

O ambiente UAT da Townsquare **aponta para o backend de produção do VendorSmart**. Isso significa que qualquer MC configurada no UAT da Townsquare precisa existir e estar corretamente configurada no PROD do VendorSmart.

### Workaround: `accounting_system_credentials`

Para que a integração funcione no UAT da Townsquare, é necessário setar a coluna `accounting_system_credentials` na tabela `management_company.management_companies` (banco do `vsp-payments-api`) com o path SSM das credenciais do sistema contábil da MC.

```sql
UPDATE management_company.management_companies
SET accounting_system_credentials = 'CAMINHO_SSM_DAS_CREDENCIAIS'
WHERE id = 'MANAGEMENT_COMPANY_ID';
```

> **Atenção**: como o UAT da Townsquare bate em PROD, execute esse UPDATE no banco de **PROD** do `vsp-payments-api`.

---

## Reference: Sentry Workflow

The Sentry 8-step workflow is the most complete example in the codebase:
- **File**: `payables-api/src/database/migrations/sentry-workflow.sql`
- Steps: Data Entry, Coding, Manager Approval, AP Review, VP ($10k), SVP ($25k), EVP ($100k), President ($250k)
- Has rejection configs on all 8 steps
- Has approval configs with tag rules on steps 3 (Manager) and 4 (AP Review)
- `has_cash_management = 1`

Use it as a template when creating a new workflow with similar structure.

---

## Useful Queries

```sql
-- Verify workflow was created correctly
SELECT w.id, w.title, ws.[order], ws.name, ws.threshold_amount, ws.approvers
FROM workflows w
JOIN workflow_steps ws ON ws.workflow_id = w.id
WHERE w.management_company_id = 'MC_ID'
ORDER BY ws.[order];

-- Verify section permissions
SELECT ws.name AS step, sp.section, sp.permission
FROM section_permissions sp
JOIN workflow_steps ws ON sp.workflow_step_id = ws.id
WHERE ws.workflow_id = 'WORKFLOW_ID'
ORDER BY ws.[order], sp.section;

-- Verify step configs and rules
SELECT ws.name AS step, wsc.config_type, wsc.reason_required, wsc.target_invoice_status,
       wscr.attribute, wscr.operation, wscr.value
FROM workflow_step_configs wsc
JOIN workflow_steps ws ON wsc.workflow_step_id = ws.id
LEFT JOIN workflow_step_config_rules wscr ON wscr.workflow_step_config_id = wsc.id
WHERE ws.workflow_id = 'WORKFLOW_ID'
ORDER BY ws.[order], wsc.[order];
```
