# Mambu API Mapping – MinSide Boliglån

**Project:** NFF Demo – MinSide Mortgage Self-Service
**Date:** 2026-03-25
**Status:** Draft v1.0

---

## 1. Overview

All communication with Mambu takes place server-side via Next.js API Routes. No Mambu calls are made directly from the browser. This document maps the application's features to concrete Mambu API endpoints and describes our internal Next.js API routes.

**Base URL:** `https://knowit.sandbox.mambu.com/api`

---

## 2. Authentication against Mambu

All Mambu API calls require two authentication mechanisms:

### 2.1 Basic Auth (username:password)

```
Authorization: Basic <base64(MAMBU_USERNAME:MAMBU_PASSWORD)>
```

### 2.2 API Key (header)

```
apikey: <MAMBU_API_KEY>
```

### 2.3 Standard headers for all Mambu calls

```http
Authorization: Basic <base64>
apikey: <MAMBU_API_KEY>
Accept: application/vnd.mambu.v2+json
Content-Type: application/json
```

---

## 3. Mambu API endpoints per feature

### 3.1 GET /api/clients/{clientKey} – Fetch client data (F02)

| Parameter     | Value                                    |
|---------------|------------------------------------------|
| Method        | GET                                      |
| Path          | `/api/clients/{clientKey}`               |
| Used in       | F02 – Loan overview (customer info)      |
| clientKey     | `8a19b6a69d255395019d262ee2a572f4`       |

**Request:**
```http
GET /api/clients/8a19b6a69d255395019d262ee2a572f4
Accept: application/vnd.mambu.v2+json
Authorization: Basic <base64>
apikey: <key>
```

**Response (200 OK):**
```json
{
  "encodedKey": "8a19b6a69d255395019d262ee2a572f4",
  "id": "885603059",
  "firstName": "Regan",
  "lastName": "Stracke",
  "state": "ACTIVE",
  "assignedBranchKey": "8a19c1ab966cca9901966cf9dceb29b8",
  "creationDate": "2024-01-15T10:00:00+01:00"
}
```

---

### 3.2 GET /api/loans/{loanKey}?detailsLevel=FULL – Fetch loan data (F02, F04, F05, F06)

| Parameter     | Value                                         |
|---------------|-----------------------------------------------|
| Method        | GET                                           |
| Path          | `/api/loans/{loanKey}?detailsLevel=FULL`      |
| Used in       | F02, F04, F05, F06                            |
| loanKey       | `8a19dc979d254c1d019d2630046f7e3d`            |

**Request:**
```http
GET /api/loans/8a19dc979d254c1d019d2630046f7e3d?detailsLevel=FULL
Accept: application/vnd.mambu.v2+json
Authorization: Basic <base64>
apikey: <key>
```

**Response (200 OK) – relevant fields:**
```json
{
  "encodedKey": "8a19dc979d254c1d019d2630046f7e3d",
  "id": "99004082",
  "loanName": "SPK Boliglån Annuitet",
  "loanAmount": 253000,
  "accountState": "APPROVED",
  "accountHolderKey": "8a19b6a69d255395019d262ee2a572f4",
  "productTypeKey": "8a19b2ee97346d40019734c0b48d2866",
  "interestSettings": {
    "interestRate": 4.0,
    "interestCalculationMethod": "DECLINING_BALANCE_DISCOUNTED",
    "interestApplicationMethod": "AFTER_DISBURSEMENT",
    "accrueInterestAfterMaturity": false
  },
  "scheduleSettings": {
    "repaymentInstallments": 24,
    "gracePeriod": 0,
    "gracePeriodType": "NONE",
    "fixedDaysOfMonth": [15],
    "scheduleDueDatesMethod": "FIXED_DAYS_OF_MONTH",
    "repaymentPeriodCount": 1,
    "repaymentPeriodUnit": "MONTHS"
  },
  "balances": {
    "principalBalance": 241500.00,
    "principalDue": 0,
    "principalPaid": 11500.00,
    "interestDue": 0,
    "interestPaid": 2719.88,
    "feesBalance": 0,
    "feesDue": 0,
    "feesPaid": 0
  },
  "currency": {
    "currencyCode": "NOK"
  },
  "customFieldValues": [
    { "fieldKey": "_loanAccountDisbursementDetails", "value": "..." },
    { "fieldKey": "_panteobjekter", "value": "..." },
    { "fieldKey": "_sikkerhetsdokumenter", "value": "..." }
  ]
}
```

---

### 3.3 GET /api/loanproducts/{productKey}?detailsLevel=FULL – Fetch product data

| Parameter     | Value                                               |
|---------------|-----------------------------------------------------|
| Method        | GET                                                 |
| Path          | `/api/loanproducts/{productKey}?detailsLevel=FULL`  |
| Used in       | Validation of allowed limits (min/max installments) |
| productKey    | `8a19b2ee97346d40019734c0b48d2866`                  |

**Request:**
```http
GET /api/loanproducts/8a19b2ee97346d40019734c0b48d2866?detailsLevel=FULL
Accept: application/vnd.mambu.v2+json
Authorization: Basic <base64>
apikey: <key>
```

**Response (200 OK) – relevant fields:**
```json
{
  "encodedKey": "8a19b2ee97346d40019734c0b48d2866",
  "id": "SPK-BOLIGLAN-ANNUITET",
  "loanProductType": "FIXED_TERM_LOAN",
  "repaymentScheduleMethod": "FIXED",
  "interestSettings": {
    "defaultInterestRate": 4.0,
    "minInterestRate": 0.0,
    "maxInterestRate": 15.0
  },
  "loanAmountSettings": {
    "defaultLoanAmount": 200000,
    "minLoanAmount": 50000,
    "maxLoanAmount": 5000000
  },
  "repaymentScheduleSettings": {
    "defaultNumInstallments": 24,
    "minNumInstallments": 6,
    "maxNumInstallments": 360,
    "defaultGracePeriod": 0,
    "maxGracePeriod": 12
  }
}
```

---

### 3.4 GET /api/loans/{loanKey}/schedule?detailsLevel=FULL – Fetch payment schedule (F03)

| Parameter     | Value                                                    |
|---------------|----------------------------------------------------------|
| Method        | GET                                                      |
| Path          | `/api/loans/{loanKey}/schedule?detailsLevel=FULL`        |
| Used in       | F03 – Payment schedule                                   |

**Request:**
```http
GET /api/loans/8a19dc979d254c1d019d2630046f7e3d/schedule?detailsLevel=FULL
Accept: application/vnd.mambu.v2+json
Authorization: Basic <base64>
apikey: <key>
```

**Response (200 OK):**
```json
{
  "installments": [
    {
      "number": 1,
      "dueDate": "2026-02-15",
      "state": "PAID",
      "isPaymentHoliday": false,
      "principal": {
        "amount": {
          "expected": 9890.12,
          "paid": 9890.12,
          "due": 0
        },
        "tax": { "expected": 0, "paid": 0 }
      },
      "interest": {
        "amount": {
          "expected": 1359.88,
          "paid": 1359.88,
          "due": 0
        },
        "tax": { "expected": 0, "paid": 0 }
      },
      "fee": {
        "amount": {
          "expected": 0,
          "paid": 0,
          "due": 0
        }
      }
    },
    {
      "number": 3,
      "dueDate": "2026-04-15",
      "state": "PENDING",
      "isPaymentHoliday": false,
      "principal": {
        "amount": {
          "expected": 9956.00,
          "paid": 0,
          "due": 9956.00
        }
      },
      "interest": {
        "amount": {
          "expected": 1294.00,
          "paid": 0,
          "due": 1294.00
        }
      },
      "fee": {
        "amount": { "expected": 0, "paid": 0, "due": 0 }
      }
    }
  ]
}
```

**Installment states:**

| State             | Description                          |
|-------------------|--------------------------------------|
| `PENDING`         | Not paid, not yet due                |
| `PAID`            | Paid in full                         |
| `LATE`            | Due, not paid                        |
| `PARTIALLY_PAID`  | Partially paid                       |
| `GRACE`           | In payment holiday period            |

---

### 3.5 Self-service endpoints (F04, F05, F06)

> Mambu uses the **colon-action pattern** for these operations – not REST PATCH at the root level.
> Confirmed from an actual API call in `PostDueDate.json`.

---

#### F04 – Change due date

| Parameter | Value                                                       |
|-----------|-------------------------------------------------------------|
| Method    | **POST**                                                    |
| Path      | `/api/loans/{loanKey}:changeDueDatesSettings`               |
| Response  | **204 No Content** (empty body)                             |
| Used in   | F04                                                         |

**Request body:**
```json
{
  "fixedDaysOfMonth": [20],
  "notes": "Due date changed by customer via MinSide",
  "valueDate": "2026-03-25T12:00:00+01:00"
}
```

> ⚠️ **Timezone requirement (confirmed by test 2026-03-25):** `valueDate` MUST use local timezone offset (e.g. `+01:00`), NOT UTC (`Z`). Mambu returns `400 INVALID_DATE` with message "Invalid date offset... org offset is +01:00" if UTC is supplied. Use `new Date().toLocaleString('sv-SE', { timeZoneName: 'longOffset' })` to generate the correct format server-side.

---

#### F05 – Change installment count

> Confirmed from `PostChangeLoanTerms.json`. Note: the sandbox call returned 500 – likely a loan state issue.

| Parameter | Value                                                       |
|-----------|-------------------------------------------------------------|
| Method    | **POST**                                                    |
| Path      | `/api/loans/{loanKey}:changeLoanTerm`                       |
| Response  | **204 No Content**                                          |
| Used in   | F05                                                         |

**Request body:**
```json
{
  "isPreview": false,
  "repaymentInstallments": 36
}
```

**isPreview pattern:**
```json
// Step 1 – preview (no saving)
{ "isPreview": true, "repaymentInstallments": 36 }

// Step 2 – confirm and save
{ "isPreview": false, "repaymentInstallments": 36 }
```

---

#### F06 – Payment holiday

> **Confirmed 2026-03-25 via sandbox testing.**
> The correct approach is a two-step read-modify-write on the schedule.

| Step | Method | Path                                          |
|------|--------|-----------------------------------------------|
| 1    | GET    | `/api/loans/{encodedKey}/schedule?detailsLevel=FULL` |
| 2    | PUT    | `/api/loans/{encodedKey}/schedule`            |

**Step 1 – Read schedule (get installment encodedKeys)**

**Step 2 – Mark installments as payment holiday:**
```json
[
  {
    "encodedKey": "<from GET schedule>",
    "number": "1",
    "dueDate": "2026-03-25T01:00:00+01:00",
    "isPaymentHoliday": true,
    "principal": { "amount": { "expected": 98774 } },
    "interest":  { "amount": { "expected": 3068 } },
    "fee":       { "amount": { "expected": 0 } }
  }
]
```

**Response: 200 OK** – returns updated installment array.
Mambu sets `state: "GRACE"`, `principal.amount.expected: 0`, `interest.amount.expected: 0`.

> ⚠️ **Note:** The loan key used in the PUT must be the `encodedKey` (not the numeric `id`). Get the `encodedKey` from `getLoan()` response first.

**Effect in schedule response:**
```json
{
  "number": 3,
  "isPaymentHoliday": true,
  "principal": { "amount": { "expected": 0, "paid": 0, "due": 0 } },
  "interest":  { "amount": { "expected": 843.00, "paid": 0, "due": 843.00 } }
}
```

---

**Common error responses (all self-service endpoints):**

| Status | Cause                                                          |
|--------|----------------------------------------------------------------|
| 400    | Invalid request (field does not exist, invalid value)          |
| 401    | Invalid credentials                                            |
| 404    | Loan object not found                                          |
| 409    | Conflict with the loan's current state                         |
| 500    | Internal Mambu error (check loan state / disbursement status)  |
| 500    | Internal Mambu error                                           |

---

## 4. Internal Next.js API Routes

All internal routes require an active NextAuth session (JWT cookie). Unauthorised calls return `401 Unauthorized`.

### 4.1 GET /api/loan – Fetch combined loan data

**Purpose:** Returns loan + client + schedule in one response to minimise the number of round-trips from the frontend.

**Request:**
```http
GET /api/loan
Cookie: next-auth.session-token=<jwt>
```

**Response (200 OK):**
```json
{
  "loan": { /* MambuLoan object */ },
  "client": { /* MambuClient object */ },
  "schedule": { /* MambuSchedule object */ }
}
```

**Errors:**
```json
// 401
{ "error": "Unauthorized" }

// 502
{ "error": "Error fetching loan data", "details": 503 }
```

**Implementation:**
```typescript
// app/api/loan/route.ts
export async function GET() {
  const session = await getServerSession(authOptions);
  if (!session) return Response.json({ error: 'Unauthorized' }, { status: 401 });

  // Step 1: getLoan is required and must come first – accountHolderKey is needed to fetch client
  // clientKey is NOT stored in session (see CLAUDE.md)
  const loan = await getLoan(session.user.loanKey);  // throws → returns 502

  // Step 2: getClient (required) and getSchedule (optional) in parallel
  const [clientResult, scheduleResult] = await Promise.allSettled([
    getClient(loan.accountHolderKey),   // required – 502 if fails
    getSchedule(session.user.loanKey),  // optional – null if fails
  ]);

  if (clientResult.status === 'rejected') {
    return Response.json({ error: 'Could not fetch client data' }, { status: 502 });
  }

  const schedule = scheduleResult.status === 'fulfilled' ? scheduleResult.value : null;
  return Response.json({ loan, client: clientResult.value, schedule });
}
```

> ⚠️ **Important:** `clientKey` is NOT stored in the JWT session. It is fetched dynamically via `loan.accountHolderKey` on each request. See CLAUDE.md for rationale.

---

### 4.2 POST /api/loan/forfallsdato – Change due date (F04)

**Request body:**
```json
{ "day": 20 }
```

**Response (200 OK):**
```json
{
  "success": true,
  "newDay": 20,
  "loan": { /* updated MambuLoan */ }
}
```

**Validation:**
- `day` is an integer
- `day` is between 1 and 28 (inclusive)

---

### 4.3 POST /api/loan/terminlengde – Change installment count (F05)

**Request body:**
```json
{ "installments": 36 }
```

**Response (200 OK):**
```json
{
  "success": true,
  "newInstallments": 36,
  "loan": { /* updated MambuLoan */ }
}
```

**Validation:**
- `installments` is an integer
- `installments` is between 6 and 360 (inclusive)

---

### 4.4 POST /api/loan/avdragsfrihet – Apply for payment holiday (F06)

**Request body:**
```json
{ "months": 3 }
```

**Response (200 OK):**
```json
{
  "success": true,
  "gracePeriodMonths": 3,
  "loan": { /* updated MambuLoan */ }
}
```

**Validation:**
- `months` is an integer
- `months` is between 1 and 12 (inclusive)

---

## 5. Common Error Handling Format

All internal API routes return errors in this format:

```json
{
  "error": "Human-readable error message (English)",
  "details": "<technical info/status code from Mambu - optional>"
}
```

---

## 6. TypeScript Types for Mambu (lib/mambu/types.ts)

```typescript
export interface MambuLoan {
  encodedKey: string;
  id: string;
  loanName: string;
  loanAmount: number;
  accountState: 'PENDING_APPROVAL' | 'APPROVED' | 'ACTIVE' | 'CLOSED' | 'WRITTEN_OFF';
  accountHolderKey: string;
  productTypeKey: string;
  interestSettings: {
    interestRate: number;
    interestCalculationMethod: 'DECLINING_BALANCE' | 'DECLINING_BALANCE_DISCOUNTED' | 'FLAT';
  };
  scheduleSettings: {
    repaymentInstallments: number;
    gracePeriod: number;
    gracePeriodType: 'NONE' | 'PAY_INTEREST_ONLY' | 'INTEREST_FORGIVENESS';
    fixedDaysOfMonth: number[];
    scheduleDueDatesMethod: 'FIXED_DAYS_OF_MONTH' | 'INTERVAL';
  };
  currency: { currencyCode: string };
}

export interface MambuClient {
  encodedKey: string;
  id: string;
  firstName: string;
  lastName: string;
  state: 'PENDING_APPROVAL' | 'ACTIVE' | 'INACTIVE' | 'BLACKLISTED' | 'REJECTED';
  assignedBranchKey: string;
}

export interface MambuInstallment {
  number: number;
  dueDate: string; // ISO 8601: "2026-04-15"
  state: 'PENDING' | 'PAID' | 'LATE' | 'PARTIALLY_PAID' | 'GRACE';
  isPaymentHoliday: boolean;
  principal: {
    amount: { expected: number; paid: number; due: number };
  };
  interest: {
    amount: { expected: number; paid: number; due: number };
  };
  fee: {
    amount: { expected: number; paid: number; due: number };
  };
}

export interface MambuSchedule {
  installments: MambuInstallment[];
}

export type MambuLoanPatch = Partial<{
  scheduleSettings: Partial<MambuLoan['scheduleSettings']>;
  gracePeriod: number;
  gracePeriodType: MambuLoan['scheduleSettings']['gracePeriodType'];
}>;
```

---

## 7. Mambu API Versioning

The app uses Mambu API v2, specified via the `Accept` header:

```
Accept: application/vnd.mambu.v2+json
```

Mambu v2 is REST-based and supports JSON Patch for PATCH operations. We use standard JSON merge patch (not JSON Patch RFC 6902).
