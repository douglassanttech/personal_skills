# Skill: Castle ETL Payouts

## Purpose
This skill is used whenever the user asks to:
- Understand the Castle Click / Jenark ETL pipeline
- Debug payout or payment issues between Castle Click, Jenark, and payables-api
- Trace an invoice through the ETL flow (from Castle Click DB to payable creation)
- Understand how checks are created on Jenark after batch approval
- Investigate DeltaPayableState transitions or invoice processing failures
- Work on vsp-castle-api ETL code, castle-adapter, or payables-api castle-etl-handler
- Understand the DynamoDB monitoring tables used in the ETL

---

## Architecture Overview

```
Castle Click DB (MSSQL)
        |
        v
vsp-castle-api (NestJS, ECS Fargate)
  ├── castle-etl/        -- ETL orchestration (sync, reprocess, review, conciliate)
  └── castle-adapter/    -- Castle Click integration (vendors, invoices, associations)
        |
        |── vsp-jenark-api    (voucher details, create checks, balance, on-hold)
        |── DynamoDB           (castle-etl-inv, castle-etl-invoice-reference)
        |── SNS                (VSP_PAYMENTS_TOPIC)
        |
        v
payables-api (NestJS, ECS Fargate)
  └── payables/use-cases/castle-etl-handler/   -- SQS consumer, creates payables
        |
        v
  MSSQL (payables DB) + SNS (PayableCreatedEvent)
```

---

## ETL Scheduled Pipelines (vsp-castle-api)

All triggered externally (not internal cron). Endpoints are internal-only.

### 1. Sync Castle Click Invoices
- **Endpoint**: `POST /etl/scheduled/castle/sync`
- **Handler**: `FetchNewInvoicesHandler` (RegisterNewInvoicesCommand)
- **Flow**: Fetches invoices from Castle Click DB by GLDate (last month to today), paginated 100/page, stores in DynamoDB `castle-etl-inv` table
- **Key file**: `vsp-castle-api/src/castle-etl/use-cases/register-new-invoices/handler.ts`

### 2. Reprocess Invoices
- **Endpoint**: `POST /etl/scheduled/invoices/reprocessing`
- **Handler**: `ScheduledReprocessInvoicesHandler`
- **Flow**: Checks processing lock in DynamoDB (`castle-etl-invoice-reference`), batches 100 invoices at a time, calls `ReprocessInvoicesHandler` per batch
- **Key file**: `vsp-castle-api/src/castle-etl/use-cases/scheduled-reprocess-invoices/handler.ts`

### 3. Process Invoice (per batch)
- **Endpoint**: `POST /etl/process-invoice`
- **Handler**: `ReprocessInvoicesHandler` (ReprocessInvoicesCommand)
- **Flow**:
  1. Get Castle Click invoice details
  2. Fetch Jenark voucher details via vsp-jenark-api
  3. Determine `DeltaPayableState` via `InvoiceManager`
  4. Sync to payables-api (create/update)
  5. Publish `CastleEtlEvent` to `VSP_PAYMENTS_TOPIC` (SNS)
- **Key file**: `vsp-castle-api/src/castle-etl/use-cases/reprocess-invoices/handler.ts`

### 4. Review Batch Invoices
- **Endpoint**: `POST /etl/scheduled/invoices/review`
- **Handler**: `ReviewInvoicesHandler`
- **Flow**: Checks invoices in batch approval screen, verifies status on Jenark, handles allocation entities (partial payments across distributions)
- **Key file**: `vsp-castle-api/src/castle-etl/use-cases/review-invoices/handler.ts`

### 5. Conciliate (Batch Approved)
- **Endpoint**: `POST /etl/events/batch-approved`
- **Handler**: `ConciliateInvoicesHandler` (ConciliateCommand)
- **Flow**:
  1. Fetch batch details from payables-api
  2. Check each payable's status on Jenark
  3. Mark voided/paid payables accordingly
  4. Create checks on Jenark for unpaid payables
  5. Publish `PaymentImportedEvent` to `VSP_PAYMENTS_TOPIC`
- **Key file**: `vsp-castle-api/src/castle-etl/use-cases/conciliate-invoices/handler.ts`

### 6. Batch Created / Deleted
- **`POST /etl/events/batch-created`**: Puts voucher on hold in Jenark (`PutVoucherOnHoldHandler`)
- **`POST /etl/events/batch-deleted`**: Releases voucher hold in Jenark

---

## DeltaPayableState Machine

The `InvoiceManager` determines the state using priority-based logic (first match wins):

| State (enum) | Value | Condition | Action in payables-api |
|---------------|-------|-----------|------------------------|
| `notDigitalPaymentBatchCode` | 0 | Batch code not digital payment | DELETE payable |
| `notDigitalPaymentAssociation` | 1 | Association not digital payment | DELETE payable |
| `notDigitalPaymentVendor` | 2 | Vendor in exception list | DELETE payable |
| `deleted` | 3 | Invoice deleted in Castle Click | DELETE payable |
| `created` | 4 | Exported to Castle Click, no voucher yet | DELETE payable |
| `voucherNotFound` | 5 | Voucher not found in Jenark | DELETE payable |
| `reimbursement` | 6 | Vendor type RB, or code starts with $&#, or name has "reimburs"/"petty cash" | DELETE payable |
| `voided` | 7 | Voucher voided in Jenark | VOID payable |
| `paid` | 8 | Paid in Jenark | CREATE/UPDATE payable |
| `posted` | 9 | Posted on Jenark, awaiting payment | CREATE/UPDATE payable |
| `updatePayment` | 10 | Payment started but not complete | CREATE/UPDATE payable |
| `exported` | 11 | Has voucher number | CREATE/UPDATE payable |

**Key file**: `vsp-castle-api/src/castle-etl/models/invoices.manager.ts`

---

## Digital Payment Eligibility

An invoice is eligible for digital payment only when ALL three conditions are true:
1. **Batch code** is marked as digital payment in Castle Click (`APBatchCode` table)
2. **Association** is marked as digital payment in Castle Click (`CG_Associations` table)
3. **Vendor** is NOT in the digital payment exception list (`VendorDigitalPaymentException` table)

---

## Payables-API Consumer (castle-etl-handler)

### SQS Configuration
- **Queue**: `VSP_CASTLE_ETL_QUEUE`
- **DLQ**: `VSP_CASTLE_ETL_DLQ_QUEUE`
- **Batch size**: 10 messages
- **Polling**: Continuous on module init

### Message Schema (CastleEtlMessage)
```typescript
{
  isBatchDigitalPayment: boolean
  isAssociationDigitalPayment: boolean
  isVendorDigitalPayment: boolean
  wasDeleted: boolean
  wasExported: boolean
  foundVoucher: boolean
  isReimbursement: boolean
  wasVoided: boolean
  wasPaid: boolean
  wasPosted: boolean
  paymentStarted: boolean
  state: DeltaPayableState
  managementCompanyId: string
  castleClickInvoice: { InvoiceId, GLDate, BatchCode }
  jenarkInvoice: { voucherNumber, description, invoiceCode, postDate, invoiceDate,
                   invoiceAmount, discountAmount, balance, paidDate, posted, void,
                   account, subAccount, vendorCode, vendorLegalName, vendorName,
                   distributions[], wasPaid, paymentStarted, batchNo, glDate }
}
```

### Processing Logic

**DELETE states** (notDigitalPayment*, deleted, voucherNotFound, reimbursement):
- Soft-deletes payable from DB (sets `deletedAt`)

**CREATE/UPDATE states** (paid, updatePayment, posted, exported):
- Each Jenark distribution becomes a separate payable
- `voucherNumber = {voucherNumber}-{expenseId}`
- `externalId = castleClickInvoice.InvoiceId`
- `tags = [castleClickInvoice.BatchCode]`
- If `posted`/`exported`: only updates if current status is `INVALID_INFORMATION`
- Publishes `PayableCreatedEvent` to SNS

**VOID state**:
- Sets `voidedAt` timestamp
- Deletes payment association if not already sent to repay
- Sends `ChangePayableBatch` SQS message

### Key files
- `payables-api/src/payables/use-cases/castle-etl-handler/handler.ts`
- `payables-api/src/payables/use-cases/castle-etl-handler/handler.consumer.ts`
- `payables-api/src/payables/use-cases/castle-etl-handler/command.ts`

---

## Payable Lifecycle Status Flow

```
OPEN
  -> PENDING_APPROVAL (initial after creation)
  -> PROCESSING (assigned to batch)
  -> AWAITING_FUNDS (batch approved)
  -> PENDING_PAYMENT (payment imported from Jenark)
  -> PAID (confirmed paid)

Alternative paths:
  PENDING_APPROVAL -> INVALID_INFORMATION (bad data)
  Any -> VOID (voucher voided in Jenark)
  Any -> REJECTED / ERRORED (validation failures)
  PAID -> REISSUED (recheck issued)
  PROCESSING -> CANCELLED (batch deleted)
```

---

## Check Creation on Jenark (Conciliation)

When a batch is approved in payables-api, `vsp-castle-api` creates checks on Jenark:

```typescript
Check {
  bank, entity, accountNumber, subAccountNumber,
  vendorCode, vendorName, associationEntity,
  postDate, apAccount, apSubAccount,
  checkDate?, checkId,
  checkVs: [{           // one per voucher
    voucherNumber, glDate, voucherBalance,
    checkVds: [{         // one per distribution
      entity, account, subAccount,
      amount, discAmount, expenseId
    }]
  }]
}
```

Vouchers are grouped by `voucherNumber-balance`. The Jenark API endpoint is `POST /check`.

---

## Castle Adapter Endpoints (vsp-castle-api)

| Endpoint | Purpose |
|----------|---------|
| `GET /castle-adapter/associations/:id/migrated` | Check if association is migrated |
| `GET /castle-adapter/vendors/association/:id` | List vendors (filterable) |
| `GET /castle-adapter/vendors/association/:id/details/:externalId` | Vendor details |
| `GET /castle-adapter/bank-accounts/association/:id/vendor/:vendorId` | Bank accounts |
| `GET /castle-adapter/gl-accounts/association/:id` | GL accounts |
| `GET /castle-adapter/departments/association/:id` | Departments |
| `POST /castle-adapter/invoices` | Create invoice (multipart file upload) |
| `GET /castle-adapter/invoices/duplication-check` | Check duplicate invoice |
| `POST /castle-adapter/invoice-events/external-invoice-created` | Update voucher number |

---

## DynamoDB Tables

| Table | Purpose | Key Fields |
|-------|---------|------------|
| `castle-etl-inv` | Monitors Castle Click invoices in ETL pipeline | castleClickId (GSI), status, isPaid, isVoid |
| `castle-etl-invoice-reference` | Processing lock and state | id="1", isRunning, latest_castle_click_invoice_id |

---

## External API Integrations

| Service | Base URL Env Var | Key Operations |
|---------|-----------------|----------------|
| vsp-jenark-api | `VSP_JENARK_API` | GET /balance/{bank}, GET /voucher/{number}, POST /check, PUT /on-hold/{number}/{bool} |
| payables-api | `VS_PAYABLES_API` | Full REST: associations, vendors, invoices, payables, batches (auth: `vs-api-key` header) |
| Castle API | `CASTLE_API_BASE_URL` | POST /voucher/{originId} (update voucher number) |
| Accounting Integration | `ACCOUNTING_INTEGRATION_API` | Vendor/account mapping across systems |

---

## SQS Consumer: External Invoice Created

- **Queue**: `VSP_PAYABLES_EXTERNAL_INVOICE_CREATED_QUEUE`
- **DLQ**: `VSP_PAYABLES_EXTERNAL_INVOICE_CREATED_DLQ_QUEUE`
- **Handler**: `ExternalInvoiceCreatedConsumerHandler`
- **Purpose**: When external system creates invoice in payables-api, updates voucher number on Castle Click via Castle API
- **Key file**: `vsp-castle-api/src/castle-adapter/use-cases/external-invoices-created-event/handler.consumer.ts`

---

## Deployment

- `vsp-castle-api` runs on **EC2** (not ECS Fargate), managed by **PM2**
- SSH: `ssh -i "~/Downloads/prod-vsp-castle-api-key.pem" ec2-user@ec2-18-206-127-185.compute-1.amazonaws.com`
  - Key file is `prod-vsp-castle-api-key.pem` (no `-prod` suffix at the end)
- PM2 logs: `/home/ec2-user/.pm2/logs/vsp-castle-api-out.log` and `vsp-castle-api-error.log`
- App lives at `/home/ec2-user/vsp-castle-api/`

---

## VPN Connectivity

- The EC2 uses a **VPN tunnel** to reach the Castle Click / Jenark on-premises systems
- **VPN is up** when `ping 170.55.119.9` succeeds from the EC2 (that's the jenark API IP from `VSP_JENARK_API` env var)
- **VPN egress IP**: Traffic exits through `170.55.119.2` — this is the on-prem customer gateway (AWS VPN connection: `vpn-05d5a0536c87488b0`, VGW: `vgw-055e738972f162bde`)
- **FortiClient is legacy** — `forticlient vpn status`, the crontab VPN observer (`/home/ec2-user/vpn/vpn-observer.sh`), and the commented-out crontab entry are all legacy artifacts from when FortiClient was used. Do NOT touch or uncomment the crontab — it is intentionally disabled and no longer relevant.

---

## ETL Stuck / isRunning Lock

The reprocess pipeline (`/etl/scheduled/invoices/reprocessing`) uses a **DynamoDB mutex** to prevent concurrent runs:
- **Table**: `castle-etl-invoice-reference`, **key**: `id = "1"`, **field**: `isRunning` (BOOL)
- If `isRunning = true`, the handler logs `"process is already running - skipping"` and returns immediately
- The flag is set `true` at start and reset `false` in a `finally` block — but if the process crashes or the EC2 restarts, **the flag gets stuck as `true`**

### Diagnosing a stuck lock
```bash
aws dynamodb get-item \
  --table-name castle-etl-invoice-reference \
  --key '{"id": {"S": "1"}}' \
  --region us-east-1 --profile prod
```

### Resetting the lock (when ETL is not actively running)
```bash
aws dynamodb put-item \
  --table-name castle-etl-invoice-reference \
  --item '{"id": {"S": "1"}, "isRunning": {"BOOL": false}}' \
  --region us-east-1 --profile prod
```

---

## Instructions for Claude

When working with Castle ETL issues:
1. **Trace the full flow**: Castle Click DB -> vsp-castle-api (DeltaPayableState) -> SNS -> payables-api consumer
2. **Check DeltaPayableState first** when debugging why a payable was created/deleted/voided
3. **Digital payment eligibility** requires all 3 conditions (batch, association, vendor) - check each
4. **Each Jenark distribution = 1 payable** in payables-api, identified by `{voucherNumber}-{expenseId}`
5. **Conciliation creates checks on Jenark** only after batch approval - issues here affect actual payments
6. **DynamoDB `isRunning` lock** — check first when ETL seems to be skipping everything. Reset with `put-item` above if stuck.
7. **VPN connectivity** — test with `ping 170.55.119.9` from EC2, NOT `forticlient vpn status` (misleading). No ping = jenark calls will fail.
8. **vsp-jenark-api** is the only gateway to Jenark data - never query Jenark DB directly from these services
9. The ETL endpoints are internal-only, triggered externally via EventBridge → `vsp-payments-handler` Lambda (schedules: sync=60min, reprocess=60min, review=10min)
