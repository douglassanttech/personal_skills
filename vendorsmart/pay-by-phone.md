# Plan: Pay by Phone Payment Type

## Context

A new payment type `'Pay by phone'` needs to be supported. When a payable is created with `paymentType === 'Pay by phone'`, it should:
- Follow the **normal invoice approval workflow** (no manual paymentNumber required up front)
- Have its batch **automatically created and approved** immediately after payable creation
- Sync to C3 as both an **invoice import** (normal flow) and **payment import** (auto-approved)
- Use a **check number from `bank_accounts.currentCheckNumber`** (incremented, like localcheck) — but **no physical check is generated**

**Key design principle**: Maximum reuse of the existing prepaid code path. The only structural difference vs. prepaid is *where the payment number comes from*: bank_accounts instead of user input.

---

## How Prepaid Works (Reference)

1. `payables.service.ts` detects `isPrepaid = status==APPROVED && paymentNumber != null`
2. Calls `createPaymentAndPayable(..., isPrepaid=true)` → `createPaidPaymentAndBatch`:
   - Batch → `BatchStatus.PAID`
   - Payment → `status=PAID`, `paymentProvider=EXTERNAL`, `checkNumber=payableEntity.paymentNumber`, `paidAt=glDate`
3. Calls `ApprovePrepaidInvoiceHandler.execute(...)`:
   - Sets `payment.paymentProvider = EXTERNAL`
   - Calls `batch.approvePayment(..., isPrepaid=true)` → emits `PaymentPaidEvent`
   - Persists batch → publishes events to `VSP_PAYMENTS_TOPIC`
4. C3 connector `shouldPaymentBeSync`: `PAID + EXTERNAL` → syncs. Uses `payment.checkNumber` as payment number.

---

## Implementation Plan

### Step 1 — `payables.service.ts` — Detect, prepare, and route Pay by Phone

**File:** `payables-api/src/payables/payables.service.ts`

**All changes are additive / minimal:**

**a) Detect** (~line 230), after `isPrepaid`:
```typescript
const isPayByPhone = dto.paymentType === 'Pay by phone';
```

**b) Include in `status` + `createPayment` calculations** (treat identically to `isPrepaid`):
```typescript
const status = PaymentStatus.PAID == dto.status || isPrepaid || isPayByPhone
  ? PaymentStatus.PAID : PaymentStatus.PENDING_PAYMENT;

const createPayment = status == PaymentStatus.PAID || isPrepaid || isPayByPhone;
```

**c) Before calling `createPaymentAndPayable`** for the `bankAccount` path — if `isPayByPhone`, increment the check number and inject it as the payment number so `createPaidPaymentAndBatch` sets it correctly on `payment.checkNumber`:
```typescript
if (isPayByPhone) {
  bankAccount.currentCheckNumber = (parseInt(bankAccount.currentCheckNumber) + 1).toString();
  payableEntity.paymentNumber = bankAccount.currentCheckNumber;
}
```

**d) Pass `isPrepaid=true` when `isPayByPhone`** in calls to `createPaymentAndPayable` and `createBankAccountAndPayable`:
```typescript
payable = await this.createPaymentAndPayable(
  payableEntity,
  bankAccount,
  dto.currentUser.id.toString(),
  isPrepaid || isPayByPhone,    // <-- only change
  dto.paymentProviderId,
  isPayByPhone,                  // <-- new flag: forceSaveBankAccount
);
```

**e) Call the SAME `approvePrepaidPaymentHandler`** for both paths — no new handler:
```typescript
if (isPrepaid || isPayByPhone) {
  await this.approvePrepaidPaymentHandler.execute(
    new ApprovePrepaidInvoiceCommand(dto.managementCompany, payable.payment.batch.id, dto.currentUser),
  );
}
```

Apply the same at the `createBankAccountAndPayable` path (~line 286). For that path, the check number increment needs to happen inside `createBankAccountAndPayable` (passing `isPayByPhone` through).

---

### Step 2 — `payables.service.ts` — Save incremented `currentCheckNumber` atomically

**File:** `payables-api/src/payables/payables.service.ts`

In `createPaymentAndPayable` (line ~332), add a `forceSaveBankAccount` parameter:
```typescript
private async createPaymentAndPayable(
  payableEntity: Payable,
  bankAccount: BankAccount,
  createdBy: string,
  isPrepaid = false,
  paymentProviderId?: string,
  forceSaveBankAccount = false,     // NEW
) {
  await this.repo.manager.transaction(async (entityManager) => {
    if (forceSaveBankAccount || (paymentProviderId && bankAccount.paymentProviderId !== paymentProviderId)) {
      if (paymentProviderId) bankAccount.paymentProviderId = paymentProviderId;
      await entityManager.getRepository(BankAccount).save(bankAccount);  // persists currentCheckNumber
    }
    payableEntity.payment = await this.createPaidPaymentAndBatch(...);
    payable = await entityManager.getRepository(Payable).save(payableEntity);
  });
}
```

This ensures the `currentCheckNumber` increment and the new payment are saved atomically.

---

### Step 3 — C3 Connector: `shouldUseGlDate`

**File:** `vsp-c3-connector/src/use-case/payment/createPayment.handler.ts` (line ~283)

Currently only prepaid invoices use `glDate`. For Pay by phone, glDate should also be used. Check whether the `Invoice` model in the C3 connector includes `paymentType`. If yes:

```typescript
private shouldUseGlDate(sampleInvoice: Invoice) {
    const isPrepaid = sampleInvoice?.invoiceType === 'Prepaid';
    const isPayByPhone = sampleInvoice?.paymentType === 'Pay by phone';
    const hasGlDate = sampleInvoice?.glDate !== null;
    return (isPrepaid || isPayByPhone) && hasGlDate;
}
```

If `paymentType` is not on the `Invoice` model, add it to the model and ensure the payables-api endpoint used by C3 connector returns this field.

---

## Critical Files

| File | Change |
|------|--------|
| `payables-api/src/payables/payables.service.ts` | `isPayByPhone` detection + check number increment + reuse prepaid path |
| `vsp-c3-connector/src/use-case/payment/createPayment.handler.ts` | Update `shouldUseGlDate` for pay by phone |

**No new handler files.** `ApprovePrepaidInvoiceHandler` and `ApprovePrepaidInvoiceCommand` are reused as-is.

**Reused unchanged:**
- `ApprovePrepaidInvoiceHandler` — `src/payables/use-cases/handle-prepaid-handler/handler.ts`
- `ApprovePrepaidInvoiceCommand` — `src/payables/use-cases/handle-prepaid-handler/command.ts`
- `createPaidPaymentAndBatch` — sets `checkNumber = payableEntity.paymentNumber` (works for both prepaid and pay by phone)
- `batch.approvePayment(..., isPrepaid=true)` — emits `PaymentPaidEvent` with `EXTERNAL` provider
- `shouldPaymentBeSync` in C3 connector — `PAID + EXTERNAL` already covers this path

---

## Verification

1. **Unit test**: Payable with `paymentType='Pay by phone'` → batch `BatchStatus.PAID`, payment `status=PAID, paymentProvider=EXTERNAL`, `payment.checkNumber = originalCheckNumber + 1`, `bank_accounts.currentCheckNumber` incremented.
2. **E2E test**: Mirror `test/v3.0/use-cases/create-prepaid-payable.e2e-spec.ts`. Assert `bank_accounts.currentCheckNumber` incremented by 1 and equals `payment.checkNumber`.
3. **C3 connector**: `shouldPaymentBeSync` `PAID + EXTERNAL` path is unchanged — no new sync logic needed. Confirm `shouldUseGlDate` returns `true` for pay by phone.
4. **No check generated**: `payment.status` is never `READY_TO_BE_PRINTED` — check generation only triggers on that status.
