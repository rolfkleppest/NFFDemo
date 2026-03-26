# F05 – Change Installment Count

**Feature ID:** F05
**Priority:** Must
**Status:** Draft v1.0
**Date:** 2026-03-25

---

## 1. Description

The user can change the number of remaining installments on the mortgage. The current installment count is 24. The user can choose to increase (extend the loan, lower installment amount) or reduce (shorter duration, higher installment amount) the number of installments. The app shows a preview of the new estimated installment amount based on the current outstanding balance, interest rate, and the new installment count – so that the user can make an informed decision before confirming the change.

---

## 2. User Story

> **As** a logged-in bank customer
> **I want** to adjust the number of remaining installments
> **So that** I can either pay down the loan faster or reduce the monthly payment burden

---

## 3. Acceptance Criteria

### AC1 – Open dialog

**Given** that the user is on the dashboard
**When** they click "Change" in the self-service card "Change installment count"
**Then** a modal dialog opens
**And** the current number of installments (24) is displayed
**And** the current estimated installment amount is displayed

### AC2 – Enter new installment count

**Given** that the dialog is open
**When** the user types or adjusts the number of installments using the slider/input
**Then** they can enter a number between 6 and 360
**And** numbers outside the limits are blocked or shown as invalid
**And** the current value (24) is pre-filled

### AC3 – Live calculation of new installment amount

**Given** that the user has changed the number of installments
**When** the new count is valid (6–360)
**Then** an estimated new installment amount is calculated and displayed in real time:
- Formula: annuity calculation based on outstanding balance, monthly interest rate, and new installment count
- Displays: "Estimated new installment amount: X NOK"
- Displays: "Total interest cost with new installment count: Y NOK"
- Comparison with current: "Difference per installment: +/- Z NOK"

### AC4 – Confirmation dialog

**Given** that the user has entered a valid new installment count
**When** they click "Next"
**Then** a confirmation summary is displayed:

| Field                              | Current    | New value   |
|------------------------------------|------------|-------------|
| Number of installments             | 24         | 36          |
| Estimated installment amount       | 11 250 NOK | 7 700 NOK   |
| Total interest cost (estimated)    | 17 000 NOK | 24 200 NOK  |
| Last installment (date)            | Jan 2028   | Jan 2029    |

**And** the user sees a warning: "Total interest cost increases when extending the installment count."

### AC5 – Successful change

**Given** that the user confirms
**When** the POST call to Mambu returns 204 No Content
**Then** the dialog closes
**And** a success message is displayed: "The installment count has been changed to 36 installments."
**And** the dashboard is updated with the new installment count and new installment amount

### AC6 – Error from Mambu

**Given** that the POST call fails (4xx/5xx)
**Then** an error message is displayed: "The change could not be completed. Please try again."
**And** the current installment count is unchanged

### AC7 – Limits enforced

**Given** that the user enters 5 (below the minimum limit)
**Then** the following is displayed: "Minimum 6 installments are required"
**And** the "Next" button is disabled

**Given** that the user enters 361 (above the maximum limit)
**Then** the following is displayed: "Maximum 360 installments (30 years)"
**And** the "Next" button is disabled

### AC8 – Unchanged count

**Given** that the user enters the same number of installments (24)
**When** they click "Next"
**Then** the following is displayed: "No change – the installment count is already 24"
**And** the "Confirm" button is disabled

---

## 4. UI Design

### Step 1 – Select new installment count

```
┌───────────────────────────────────────────────────────────┐
│  Change installment count                          [X]    │
│  ─────────────────────────────────────────────────────── │
│                                                           │
│  Current: 24 installments (approx. 11 250 NOK/mo)        │
│                                                           │
│  Number of installments:                                  │
│  ┌──────────────────────────────────────────────────┐    │
│  │  [  36  ]  ←  Input field (6–360)                │    │
│  └──────────────────────────────────────────────────┘    │
│                                                           │
│  ──────●─────────────────────────────────────────────    │
│  6     24     60     120    180    240    300    360      │
│        (now)                                             │
│                                                           │
│  ┌──────────────────────────────────────────────────┐    │
│  │  Estimated new installment amount: 7 700 NOK/mo  │    │  ← Updates live
│  │  Current installment amount:      11 250 NOK/mo  │    │
│  │  Difference per installment:      -3 550 NOK     │    │
│  │  Total interest cost (new):       24 200 NOK     │    │
│  │  Last installment:                January 2029   │    │
│  └──────────────────────────────────────────────────┘    │
│                                                           │
│  ⚠ Extending the loan duration increases total interest.  │
│                                                           │
│  [Cancel]                              [Next →]          │
└───────────────────────────────────────────────────────────┘
```

### Step 2 – Confirmation

```
┌───────────────────────────────────────────────────────────┐
│  Confirm change of installment count               [X]   │
│  ─────────────────────────────────────────────────────── │
│                                                           │
│  ┌──────────────────────────────────────────────────┐    │
│  │               Current         New                │    │
│  │  Installments:    24            36               │    │
│  │  Installment:  11 250 NOK   7 700 NOK            │    │
│  │  Interest:     17 000 NOK  24 200 NOK            │    │
│  │  Last inst.:   Jan. 2028   Jan. 2029             │    │
│  └──────────────────────────────────────────────────┘    │
│                                                           │
│  ⚠ Warning: By extending the installment count you will   │
│    pay a total of 7 200 NOK more in interest over the    │
│    loan lifetime.                                        │
│                                                           │
│  [← Back]                   [Confirm change ✓]          │
└───────────────────────────────────────────────────────────┘
```

---

## 5. Calculation Logic

### Annuity formula (client-side estimate)

```typescript
// lib/utils/calculations.ts

/**
 * Calculates the monthly annuity payment amount.
 * @param principal   - Outstanding balance (loan amount)
 * @param annualRate  - Annual interest rate in percent (e.g. 4.0)
 * @param installments - Number of remaining installments
 */
export function calculateMonthlyPayment(
  principal: number,
  annualRate: number,
  installments: number
): number {
  const monthlyRate = annualRate / 100 / 12;
  if (monthlyRate === 0) return principal / installments;
  const numerator = principal * monthlyRate * Math.pow(1 + monthlyRate, installments);
  const denominator = Math.pow(1 + monthlyRate, installments) - 1;
  return numerator / denominator;
}

/**
 * Calculates total interest cost over the loan lifetime.
 */
export function calculateTotalInterest(
  principal: number,
  monthlyPayment: number,
  installments: number
): number {
  return monthlyPayment * installments - principal;
}
```

**Note:** The client-side calculation is an estimate only. Mambu calculates the actual schedule server-side using its own engine (DECLINING_BALANCE_DISCOUNTED).

---

## 6. Mambu API Call

### Endpoint
```
POST /api/loans/{loanKey}:changeLoanTerm
```

> Confirmed from `PostChangeLoanTerms.json`. Same colon-action pattern as the due date endpoint.

### Request body
```json
{
  "isPreview": false,
  "repaymentInstallments": 36
}
```

| Field                   | Type    | Required | Description                                                      |
|-------------------------|---------|----------|------------------------------------------------------------------|
| `isPreview`             | boolean | Yes      | `true` = calculate without saving (preview), `false` = save     |
| `repaymentInstallments` | number  | Yes      | Desired number of installments                                   |

> **isPreview pattern:** Call with `isPreview: true` to show the new payment schedule to the user before confirmation. Call with `isPreview: false` to save the change.

### Expected response
```
204 No Content
```
Empty response body. Success is confirmed via HTTP status code.

> **Note:** The sandbox returned 500 in the test example (`PostChangeLoanTerms.json`). This may be because the loan does not have the correct state (e.g. not disbursed). Use an actively disbursed loan in production.

---

## 7. Next.js API Route

### `app/api/loan/terminlengde/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth/authOptions';
import { patchLoan } from '@/lib/mambu/loans';

const MIN_INSTALLMENTS = 6;
const MAX_INSTALLMENTS = 360;

export async function POST(request: NextRequest) {
  const session = await getServerSession(authOptions);
  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { installments } = await request.json();

  if (
    !Number.isInteger(installments) ||
    installments < MIN_INSTALLMENTS ||
    installments > MAX_INSTALLMENTS
  ) {
    return NextResponse.json(
      { error: `Number of installments must be between ${MIN_INSTALLMENTS} and ${MAX_INSTALLMENTS}.` },
      { status: 400 }
    );
  }

  try {
    await mambuPost(`/loans/${session.user.loanKey}:changeLoanTerm`, {
      isPreview: false,
      repaymentInstallments: installments,
    });

    return NextResponse.json({ success: true });
  } catch (error) {
    return NextResponse.json(
      { error: 'Could not update installment count.' },
      { status: 502 }
    );
  }
}
```

---

## 8. Validation Rules

| Rule                   | Limit     | Error message                                          |
|------------------------|-----------|--------------------------------------------------------|
| Minimum installments   | 6         | "Minimum 6 installments are required"                  |
| Maximum installments   | 360       | "Maximum 360 installments (30 years) are allowed"      |
| Integer required       | N/A       | "Enter an integer for the number of installments"      |
| Unchanged value        | N/A       | "No change – the same installment count is selected"   |

---

## 9. Out of Scope

- Calculation of actual monthly amount including fees (uses estimate)
- Remaining installments (difference between total and paid) – uses total
- Partial repayment / extra repayment
- Refinancing to a new loan
