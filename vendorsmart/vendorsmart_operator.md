# Skill: vendorsmart_qa_operator

## Purpose
This skill is used whenever the user asks to:
- Fetch data from VendorSmart QA environment
- Troubleshoot issues in VendorSmart
- Simulate API calls
- Investigate payments, invoices, vendors or bank accounts
- Perform operational or technical actions in VendorSmart

---

## QA Environment

### API
Base URL:
https://api-v2.vendorsmarttest.com

---

### Management Company (C3)
Management Company ID:
18991

---

### User (C3)
Email:
c3-user-a@gmail.com

Password:
Admin@123

---

### UI Access
https://vendorsmarttest.com/

---

### Environment Variables
Bank and sensitive credentials are stored in:

- PAYABLES_PROD.env
- PAYABLES_QA.env

Claude should assume these files are available locally and reference them when needed.

---

## PROD Environment (to be filled)

### API
Base URL:
[ADD HERE]

### Management Company
[ADD HERE]

### User
Email:
[ADD HERE]

Password:
[ADD HERE]

### UI
[ADD HERE]

---

## Instructions for Claude

- Default to QA environment unless explicitly told to use PROD
- When debugging issues, always:
  1. Identify entity (invoice, vendor, payable, bank account)
  2. Determine environment (QA or PROD)
  3. Suggest API endpoints to validate data
  4. Suggest logs or DB tables to inspect if needed
- When possible, provide curl examples
- Be concise and action-oriented
- Assume the user is technical

---

## Common Context

- System integrates with:
  - C3
  - Castle Click (Jenark)
  - Payabli
- Payments may be in states like:
  - Authorized
  - Captured
  - Failed
- Vendor enablement and bank accounts are critical for ACH

---

## Behavior

When user says things like:
- "check this invoice"
- "why this payment failed"
- "get vendor data"
- "test this endpoint"

👉 Automatically use this skill.

swagger prod vsp-castle-api

http://ec2-18-206-127-185.compute-1.amazonaws.com:3000/docs