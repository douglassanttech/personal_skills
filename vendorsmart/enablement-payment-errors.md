# Skill: Enablement Payment Errors

## Purpose

Use this skill when the user receives a `PaymentProviderPaymentProcessingFailureEvent` (from Payabli via SNS/email) and needs to:
- Identify the vendor and payment details for the failed payment
- Compose a support-ticket email to investigate the root cause

---

## Trigger

User pastes a JSON event like:
```json
{
  "eventName": "PaymentProviderPaymentProcessingFailureEvent",
  "data": {
    "paymentId": "...",
    "externalPaymentId": "...",
    "managementCompanyId": "...",
    "currentStatus": "Authorized",
    "source": "payabli",
    "response": { "responseData": { "resultText": "..." } }
  }
}
```

---

## Step 1 — Identify error type

Read `data.response.responseData.resultText` from the event.

| Pattern in resultText | Meaning | Action |
|---|---|---|
| `422: The 'to' address does not meet your minimum deliverability strictness` | Lob rejected the check — vendor address is invalid or undeliverable | Find correct vendor address |
| Other | Unknown error | Escalate with full event payload |

---

## Step 2 — Fetch vendor data from CloudWatch logs

The authorize request payload (sent to Payabli just before the failure) contains all vendor fields. Fetch it from the prod Lambda logs using `externalPaymentId` from the event.

```bash
# Get timestamps for 48h window
START=$(($(date -v-48H +%s) * 1000))
END=$(($(date +%s) * 1000))

aws logs filter-log-events \
  --log-group-name /aws/lambda/vsp-payabli-connector \
  --filter-pattern '"<externalPaymentId>"' \
  --start-time $START \
  --end-time $END \
  --profile prod --region us-east-1 \
  --output json
```

This returns all log entries for that payment. Find the entry with `"msg": "[Payabli API] Authorize transaction request"` whose response referenceId matches the `externalPaymentId`. That entry contains `vendorData` with:
- `name1` — vendor name
- `vendorNumber` — vendor code
- `address1`, `city`, `state`, `zip` — vendor address
- And `paymentDetails.totalAmount`

The authorize REQUEST entry is the one logged just before the authorize RESPONSE with the matching `referenceId`. Since many payments may be in the same log stream, identify the right one by:
1. Finding the authorize RESPONSE entry with `responseData.referenceId == externalPaymentId`
2. The authorize REQUEST immediately before it (by timestamp) is the matching vendor payload

---

## Step 3 — Compose support-ticket email

**Subject:** `Payment Failed — Invalid Vendor Address | <name1> (<vendorNumber>)`

**Body:**
```
Vendor name: <name1>
Vendor code: <vendorNumber>
Vendor address: <address1>
Vendor city: <city>
Vendor state: <state>
Vendor zip: <zip>

Payment total amount: <paymentDetails.totalAmount>
Payment ID in VS: <data.paymentId>
Payment ID in Payabli: <data.externalPaymentId>
Payabli transaction: https://app.payabli.com/gridsystems/pay-out/transactions?payout_paymentId=<data.externalPaymentId>

Error was:
<resultText>
This error means that we need to find out the correct vendor address.

Thanks!
```

---

## Notes

- Prod log group: `/aws/lambda/vsp-payabli-connector` in account `496974357483`, region `us-east-1`, AWS profile `prod`
- The authorize request log entry contains `"msg": "[Payabli API] Authorize transaction request"` and a `data.vendorData` object with all vendor fields
- The capture request (`"msg": "[Payabli API] Capture transaction request"`) does NOT include the payload (`"payload": null`), so always use the authorize request
- The `externalPaymentId` in the failure event matches `responseData.referenceId` in the authorize response
