# Skill: VendorSmart Invoice Workflow

## Purpose
This skill is used whenever the user asks to:
- Define, design, or plan an invoice workflow for a management company
- Understand how invoice workflows work in VendorSmart
- Configure workflow steps, rules, approvals, or thresholds
- Troubleshoot workflow behavior or step transitions
- Compare workflows across management companies
- Onboard a new management company and define their invoice approval process

---

## Invoice Workflow Architecture

### Core Concepts

Workflows are **management company-scoped** templates that define the sequence of steps an invoice goes through from receipt to approval/rejection.

- Each **management company** has one or more **workflow templates**
- Each **workflow** contains an **ordered list of steps**
- Each **association** within a management company is **assigned** one active workflow via a **WorkflowAssignment**
- When an invoice is created, a **WorkflowExecution** is instantiated from the assigned workflow

---

## Workflow Steps

### Step Types

| Type | `approvers` value | Behavior |
|------|-------------------|----------|
| **DATA_ENTRY** | `0` | No approval needed. User fills in data and moves to next step. |
| **APPROVAL_STEP** | `> 0` | Requires N approvers to approve before progressing. |

### Step Properties

| Property | Description |
|----------|-------------|
| `order` | Position in the workflow (1, 2, 3...) |
| `name` | Display name (e.g., "Data Entry", "Coding", "Board Approval") |
| `approvers` | Number of required approvals (0 = data entry step) |
| `threshold_amount` | If invoice total < this amount, step is auto-skipped |

### Step Statuses

```
PENDING                 - Step not yet reached
DONE                    - Data entry step completed
APPROVED                - Approval step approved by required approvers
REJECTED                - Step was rejected
SKIPPED_BY_THRESHOLD    - Auto-skipped because invoice amount < threshold
```

---

## Section Permissions (per step)

Each step controls which invoice sections are editable or view-only:

| Section | Description |
|---------|-------------|
| `INVOICE_DATA_SECTION` | Invoice header data (vendor, amount, date, etc.) |
| `DISTRIBUTION_SECTION` | GL coding / cost distribution |
| `BANK_INFO_SECTION` | Vendor bank account information |
| `INVOICE_PREFERENCES_SECTION` | Payment preferences and settings |

Permissions: `EDIT` or `VIEW`

---

## Step Configs (Conditional Rules)

Each step can have configs for **APPROVAL** and **REJECTION** events. Configs define what happens after a step action.

### Config Properties

| Property | Description |
|----------|-------------|
| `configType` | `APPROVAL` or `REJECTION` |
| `reasonRequired` | Whether user must provide a reason |
| `targetStep` | Jump to a specific step (instead of next in order) |
| `targetInvoiceStatus` | Set the invoice to a specific status |
| `targetInvoiceTags` | Tags to add to the invoice |
| `tagActionType` | `ADD_TAG` |
| `rules` | Conditions that must ALL match for this config to apply |

### Rule Operations

| Operation | Description |
|-----------|-------------|
| `EQUAL` | Exact match |
| `NOT_EQUAL` | Not equal |
| `GREATER_THAN` | Numeric greater than |
| `LESS_THAN` | Numeric less than |
| `GREATER_THAN_OR_EQUAL` | >= |
| `LESS_THAN_OR_EQUAL` | <= |
| `ANY_OF` | Value matches any in comma-separated list |
| `NOT_ANY_OF` | Value matches none in comma-separated list |
| `ALL_OF` | All items in list must be present |
| `DIFFERENT_FROM` | Value differs from specified |

Rules use `attribute`, `operation`, and `value` to match against invoice properties. **ALL rules in a config must match** for the config to apply.

---

## Invoice Statuses

```
UNASSIGNED   - No workflow assigned
PROCESSING   - Currently going through workflow
REJECTED     - Rejected during workflow
APPROVED     - Workflow completed successfully
ERRORED      - Error during processing
VOIDED       - Invoice voided
```

---

## Workflow Assignment

Workflow assignments connect a workflow template to a specific association:

- **One active assignment per association** at a time
- Old assignments are marked `INACTIVE` when a new one is created
- **Responsibles (approvers) are configured per step per assignment** - same workflow template can have different approvers for different associations
- Step configs (rules) can also be customized per assignment

---

## Step Transitions

| Transition Type | Description |
|----------------|-------------|
| **Sequential** | Move to next step in order |
| **Conditional** | Jump to a specific step if config rules match |
| **Threshold-based** | Auto-skip step if invoice amount < threshold |

### Workflow Completion
A workflow is considered complete when:
- The last step is APPROVED or SKIPPED_BY_THRESHOLD
- No rejection occurred

---

## Example Workflow Templates (existing in system)

### 1. Basic (3 steps)
```
Step 1: Data Entry       (approvers: 0)
Step 2: Coding           (approvers: 0)
Step 3: Final Approval   (approvers: 1)
```

### 2. With Board Approval (4 steps)
```
Step 1: Data Entry       (approvers: 0)
Step 2: Coding           (approvers: 0)
Step 3: Board Approval   (approvers: 1, threshold: $X)
Step 4: Manager Approval (approvers: 1)
```

### 3. Complex (6 steps)
```
Step 1: Data Entry       (approvers: 0)
Step 2: Coding           (approvers: 0)
Step 3: Approval Level 1 (approvers: 1, threshold: $X)
Step 4: Approval Level 2 (approvers: 1, threshold: $Y)
Step 5: Board Approval   (approvers: 1, threshold: $Z)
Step 6: Final Approval   (approvers: 1)
```

### 4. Coding-first (3 steps)
```
Step 1: Coding           (approvers: 0)
Step 2: Board Approval   (approvers: 1)
Step 3: Manager Approval (approvers: 1)
```

---

## Key Files in Codebase

### Backend (payables-api)

| Category | Path |
|----------|------|
| **Entities** | `src/invoices/models/repositories/entities/workflow*.entity.ts`, `section-permission.entity.ts` |
| **Domain Models** | `src/invoices/models/workflow*.ts`, `workflow-step*.ts` |
| **Repositories** | `src/invoices/models/repositories/workflow*.repository.ts` |
| **Controllers** | `src/invoices/controllers/workflows.controller.ts`, `association-workflow.controller.ts` |
| **Use Cases** | `src/invoices/use-cases/workflow/` (create, update, move-to-next-step, edit-config) |
| **Enums** | `src/invoices/models/enums/config-type.enum.ts`, `operation.enum.ts` |
| **Migrations** | `src/database/migrations/` (workflow setup SQL) |

### Frontend (vs-web)

| Category | Path |
|----------|------|
| **Models** | `association-workflow.model.ts`, `workflow-step-config-response.ts` |
| **Enums** | `config-type.enum.ts`, `step-status.enum.ts`, `invoice.enum.ts` |
| **Services** | `association-workflow.service.ts`, `workflow-assignment.service.ts` |
| **Components** | workflow config, workflow list, step display, batch approval components |

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/workflows` | List workflows for management company |
| `POST` | `/workflow-assignments` | Create assignment (workflow -> association) |
| `PATCH` | `/workflow-assignments/:id` | Update assignment |
| `GET` | `/workflow-assignments` | List assignments |
| `GET` | `/workflow-assignments/:id` | Get assignment details |
| `POST` | `/workflow-assignments/:id/step/:stepId/config` | Configure step behavior |
| `GET` | `/workflow-assignments/:id/step/:stepId/config` | Get step config |

---

## Database Relationships

```
Workflow (template, scoped to management company)
  └── WorkflowStep[] (ordered steps)
       ├── SectionPermission[] (edit/view per section)
       ├── WorkflowStepConfig[] (approval/rejection rules)
       │    └── WorkflowStepConfigRule[] (conditions)
       └── WorkflowStepResponsible[] (approvers per assignment)

WorkflowAssignment (links workflow to association)
  ├── workflow -> Workflow
  ├── associationId -> Association
  ├── status: ACTIVE | INACTIVE
  └── WorkflowStepConfig[] (assignment-specific overrides)

WorkflowExecution (running instance per invoice)
  ├── workflow -> Workflow
  ├── currentStep -> WorkflowStep
  └── WorkflowExecutionStep[] (step execution history)
       └── WorkflowStepConfigExecution[] (matched rules log)
```

---

## Instructions for Claude

When helping define a new workflow:

1. **Gather requirements**:
   - Management company name and size
   - Number of associations
   - Approval hierarchy (who approves what)
   - Threshold amounts for auto-skip
   - Any special rules (vendor-specific, amount-based routing, etc.)

2. **Design the workflow**:
   - Define steps in order with type (data entry vs approval)
   - Set threshold amounts per step
   - Define section permissions per step
   - Define conditional rules (step configs) if needed

3. **Document the workflow** as a clear table showing:
   - Step order, name, type, approvers count, threshold
   - Section permissions per step
   - Rejection/approval rules and target steps

4. **Reference existing workflows** as starting points when appropriate

5. **Consider common patterns**:
   - Data Entry is almost always Step 1 (or Coding if data comes pre-filled)
   - Coding (GL distribution) usually comes before approvals
   - Board approval often has a high threshold (only triggered for large invoices)
   - Final approval (manager) is usually the last step
   - Rejection configs often send back to an earlier step (e.g., back to Coding)
