# F03 – Payment Schedule

**Feature ID:** F03
**Priority:** Must
**Status:** Draft v1.0
**Date:** 2026-03-25

---

## 1. Description

The Payment schedule page displays a complete overview of all 24 installments for the loan. The user can see the due date, principal, interest, fees, total installment amount, and status for each individual installment. Paid installments are visually marked. The page is responsive and suitable for use on mobile.

---

## 2. User story

> **As** a logged-in bank customer
> **I want** to see the full payment schedule for my loan
> **So that** I can plan future payments and see which installments have been paid

---

## 3. Acceptance criteria

### AC1 – Table shows all remaining installments

**Given** that the user navigates to `/dashboard/betalingsplan`
**And** the payment schedule has been retrieved from Mambu
**When** the page loads
**Then** a table is displayed with all installments returned by the Mambu `/schedule` endpoint
**And** the number of installments is shown dynamically (e.g. "24 installments" – updated automatically after an F05 change)

### AC2 – Table columns

**Given** that the table has loaded
**Then** the following columns are displayed:

| Column       | Description                              | Field name        |
|--------------|------------------------------------------|-------------------|
| #            | Installment number (dynamic, 1–N)        | Installment       |
| Due date     | Date of due (formatted nb-NO)            | Due date          |
| Principal    | Principal amount in NOK                  | Principal         |
| Interest     | Interest amount in NOK                   | Interest          |
| Fee(s)       | Any fees (0 if none)                     | Fee(s)            |
| Total        | Sum of principal + interest + fees       | Total             |
| Status       | Badge: Paid / Overdue / Future           | Status            |

### AC3 – Status indication

**Given** that an installment is paid (state: `PAID`)
**Then** a green badge is shown with the text "Paid" and a checkmark icon

**Given** that an installment has state `PENDING` from Mambu and the due date is within 7 days
**Then** a yellow/orange badge is shown with the text "Due soon"
> **App-side logic:** Mambu returns `PENDING` for all future installments regardless of proximity. The app itself calculates whether the due date is ≤ 7 days from `new Date()` to distinguish "Due soon" from "Future".

**Given** that an installment has state `PENDING` from Mambu and the due date is more than 7 days away
**Then** a neutral grey badge is shown with the text "Future"

**Given** that an installment is overdue/unpaid (state: `LATE`)
**Then** a red badge is shown with the text "Overdue"

### AC4 – Summary row

**Given** that the table is displayed
**Then** a summary row is shown at the bottom with:
- Total principal (sum of all installments)
- Total interest (sum of all installments)
- Total fees
- Total payment over the life of the loan

### AC5 – Current installment highlighted

**Given** that the payment schedule has loaded
**And** there is an installment with an upcoming due date (within 30 days)
**Then** that row is visually highlighted (light blue background, bold text)

### AC6 – Responsive table

**Given** that the user is on mobile (viewport < 768px)
**When** they view the payment schedule
**Then** the table is horizontally scrollable within a container element
**And** the installment number and due date are sticky (locked to the left)

### AC7 – Loading state

**Given** that data is being fetched
**Then** a skeleton table with 5 rows is shown as a placeholder

### AC8 – Error handling

**Given** that the schedule endpoint returns an error
**Then** the message is shown: "Could not retrieve payment schedule. Please try again."
**And** a "Try again" button is displayed

---

## 4. UI design

```
┌──────────────────────────────────────────────────────────────────────────┐
│  ← Back to overview                                                      │
│                                                                          │
│  Payment Schedule – SPK Boliglån Annuitet                                │
│  24 installments | Rate: 4.00 % | Due date: 15th of each month           │
│                                                                          │
│  ┌────┬──────────────┬──────────────┬──────────┬────────┬──────────┬──────────┐
│  │ #  │ Due date     │ Principal    │ Interest │ Fee(s) │ Total    │ Status   │
│  ├────┼──────────────┼──────────────┼──────────┼────────┼──────────┼──────────┤
│  │  1 │ 15 Feb 2026  │  9 890 NOK   │ 1 360 NOK│  0 NOK │11 250 NOK│ ✓ Paid   │
│  │  2 │ 15 Mar 2026  │  9 923 NOK   │ 1 327 NOK│  0 NOK │11 250 NOK│ ✓ Paid   │
│  │  3 │ 15 Apr 2026  │  9 956 NOK   │ 1 294 NOK│  0 NOK │11 250 NOK│ ⚠ Soon   │  ← Highlighted
│  │  4 │ 15 May 2026  │  9 989 NOK   │ 1 261 NOK│  0 NOK │11 250 NOK│ ○ Future │
│  │ .. │     ...      │    ...       │    ...   │  ...   │    ...   │    ...   │
│  │ 24 │ 15 Jan 2028  │ 10 453 NOK   │   797 NOK│  0 NOK │11 250 NOK│ ○ Future │
│  ├────┼──────────────┼──────────────┼──────────┼────────┼──────────┼──────────┤
│  │ Σ  │              │ 253 000 NOK  │17 000 NOK│  0 NOK │270 000 NOK│         │
│  └────┴──────────────┴──────────────┴──────────┴────────┴──────────┴──────────┘
│                                                                          │
│  Table information:                                                      │
│  • Calculated using the DECLINING_BALANCE_DISCOUNTED method              │
│  • Amounts are estimates – may differ due to interest rate adjustments   │
└──────────────────────────────────────────────────────────────────────────┘
```

**Colors:**
- Paid row: Green badge (`#16a34a`)
- Due-soon row: Yellow background (`#fef9c3`), orange badge (`#d97706`)
- Future row: Normal white/grey
- Summary row: `#f0f4ff` background, bold text

---

## 5. Mambu API response – Schedule

### Endpoint
```
GET /api/loans/{loanKey}/schedule?detailsLevel=FULL
Headers:
  Authorization: Basic <base64(user:pass)>
  apikey: <api-key>
  Accept: application/vnd.mambu.v2+json
```

### Example response

```json
{
  "installments": [
    {
      "number": 1,
      "dueDate": "2026-02-15",
      "state": "PAID",
      "principal": {
        "amount": { "expected": 9890.12, "paid": 9890.12 }
      },
      "interest": {
        "amount": { "expected": 1359.88, "paid": 1359.88 }
      },
      "fee": {
        "amount": { "expected": 0, "paid": 0 }
      }
    },
    {
      "number": 3,
      "dueDate": "2026-04-15",
      "state": "PENDING",
      "principal": {
        "amount": { "expected": 9956.00, "paid": 0 }
      },
      "interest": {
        "amount": { "expected": 1294.00, "paid": 0 }
      },
      "fee": {
        "amount": { "expected": 0, "paid": 0 }
      }
    }
  ]
}
```

---

## 6. Technical implementation

### Server Component (`app/dashboard/betalingsplan/page.tsx`)

```typescript
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth/authOptions';
import { getLoan } from '@/lib/mambu/loans';
import { getSchedule } from '@/lib/mambu/schedule';
import PaymentScheduleTable from '@/components/loan/PaymentScheduleTable';
import { redirect } from 'next/navigation';

export default async function BetalingsplanPage() {
  const session = await getServerSession(authOptions);
  if (!session) redirect('/login');

  const [loan, schedule] = await Promise.all([
    getLoan(session.user.loanKey),
    getSchedule(session.user.loanKey),
  ]);

  return (
    <main className="p-6 max-w-6xl mx-auto">
      <PaymentScheduleTable
        installments={schedule.installments}
        loanName={loan.loanName}
        interestRate={loan.interestSettings.interestRate}
        fixedDayOfMonth={loan.scheduleSettings.fixedDaysOfMonth[0]}
      />
    </main>
  );
}
```

### Status logic (`components/loan/PaymentScheduleTable.tsx`)

```typescript
function getInstallmentStatus(installment: Installment) {
  if (installment.state === 'PAID') return 'paid';
  const dueDate = new Date(installment.dueDate);
  const today = new Date();
  const daysUntilDue = Math.ceil(
    (dueDate.getTime() - today.getTime()) / (1000 * 60 * 60 * 24)
  );
  if (daysUntilDue < 0) return 'late';
  if (daysUntilDue <= 7) return 'due-soon';
  return 'future';
}
```

---

## 7. Out of Scope

- Downloading the payment schedule as a PDF
- Email notifications about upcoming due dates
- Filtering/sorting the table
- Display of amounts already paid vs. overdue
