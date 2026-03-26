# F02 – Loan Overview

**Feature ID:** F02
**Priority:** Must
**Status:** Draft v1.0
**Date:** 2026-03-25

---

## 1. Description

After login the user lands on the dashboard, which shows a complete overview of the mortgage. The page fetches loan data, customer information, and next installment details from the Mambu API server-side. The user sees key information clearly and has access to self-service features from this page.

---

## 2. User story

> **As** a logged-in bank customer
> **I want** to see an overview of my mortgage in one place
> **So that** I can quickly understand my loan situation and navigate to self-service features

---

## 3. Acceptance criteria

### AC1 – Data loading on login

**Given** that the user is authenticated
**And** the user navigates to `/dashboard`
**When** the page loads
**Then** loan data is fetched from Mambu (loan + client + schedule) server-side
**And** the page is rendered with correct data without client-side JS calls to Mambu

### AC2 – Loan card shows key info

**Given** that loan data has been fetched
**When** the user views the dashboard
**Then** the following information is displayed clearly:

| Field                    | Value (example)             | Source                               |
|--------------------------|-----------------------------|---------------------------------------|
| Loan name                | SPK Boliglån Annuitet       | `loan.loanName`                      |
| Loan ID                  | 99004082                    | `loan.id`                            |
| Outstanding balance      | 241 500 NOK                 | `loan.balances.principalBalance`     |
| Original loan amount     | 253 000 NOK                 | `loan.loanAmount`                    |
| Current interest rate    | 4.00 %                      | `loan.interestSettings.interestRate` |
| Remaining installments   | 22 of 24                    | calculated from schedule             |
| Due date (day)           | 15th of each month          | `loan.scheduleSettings.fixedDaysOfMonth` |
| Loan status              | APPROVED (badge)            | `loan.accountState`                  |

> **Outstanding balance is the most important value** for a borrower and is displayed prominently at the top of the card.

### AC3 – Customer information card

**Given** that client data has been fetched
**When** the user views the dashboard
**Then** the following is displayed:
- Full name: Regan Stracke
- Customer number: 885603059

### AC4 – Next installment payment

**Given** that the payment schedule has been fetched
**When** the user views the dashboard
**Then** the next unpaid installment is displayed with:
- Due date (e.g. 15 April 2026)
- Total installment amount (principal + interest + any fees)
- Principal amount
- Interest amount

### AC5 – Loading state (skeleton)

**Given** that data is being fetched (e.g. slow connection)
**When** the page loads
**Then** skeleton loading elements are shown in the cards
**And** the app is not empty/white while data is being fetched

### AC6 – Error handling

**Given** that the Mambu API is unavailable
**When** the server-side fetch fails
**Then** an error message is displayed: "Could not fetch loan data. Please try again or contact the bank."
**And** the page does not crash
**And** a "Try again" button is displayed

### AC7 – Self-service section

**Given** that the user is on the dashboard
**When** they view the "Self-service" section
**Then** three action cards are displayed:
1. "Change due date" → opens F04 modal
2. "Change installment period" → opens F05 modal
3. "Payment holiday" → opens F06 modal

### AC8 – Navigation to payment schedule

**Given** that the user is on the dashboard
**When** they click "View payment schedule"
**Then** they are navigated to `/dashboard/betalingsplan`

---

## 4. UI design

### Dashboard layout

```
┌──────────────────────────────────────────────────────────────┐
│  🏦 MinSide Boliglån          Regan Stracke  [Log out]       │
│  ─────────────────────────────────────────────────────────── │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  SPK Boliglån Annuitet                    ● APPROVED  │   │
│  │  Loan account: 99004082                               │   │
│  │                                                       │   │
│  │  241 500 NOK  ← Outstanding balance (primary value)   │   │
│  │                                                       │   │
│  │  253 000 NOK          4.00 %          24 installments │   │
│  │  Orig. amount         Rate            Count           │   │
│  │                                                       │   │
│  │  Due date: 15th of each month                         │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────┐  ┌────────────────────────┐    │
│  │  Next installment       │  │  Your profile          │    │
│  │  15 April 2026          │  │  Regan Stracke         │    │
│  │                         │  │  Customer no: 885603059│    │
│  │  Total:  11 250 NOK     │  │                        │    │
│  │  Principal:  9 890 NOK  │  └────────────────────────┘    │
│  │  Interest:  1 360 NOK   │                                 │
│  │                         │                                 │
│  │  [View payment schedule →]│                               │
│  └─────────────────────────┘                                 │
│                                                              │
│  Self-service                                                │
│  ─────────────────────────────────────────────────────────  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Change      │  │  Change      │  │  Payment holiday │  │
│  │  due date    │  │  installment │  │                  │  │
│  │  [day 15]    │  │  period      │  │  Pause principal │  │
│  │              │  │  [24 inst.]  │  │  1-12 months     │  │
│  │  [Change →]  │  │  [Change →]  │  │  [Apply →]      │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

**Design notes:**
- Top navbar: `#0f2044` (navy), white text
- Main loan card: gradient from `#1a3a6b` to `#0f2044`, white text
- Status cards: white background with `#e8f0fe` border
- Self-service cards: white background, blue accent line at top
- Typography: Inter or system-ui, 16px base

---

## 5. Technical implementation

### Server Component (`app/dashboard/page.tsx`)

```typescript
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth/authOptions';
import { getLoan } from '@/lib/mambu/loans';
import { getClient } from '@/lib/mambu/clients';
import { getSchedule } from '@/lib/mambu/schedule';
import { redirect } from 'next/navigation';
import LoanSummaryCard from '@/components/loan/LoanSummaryCard';
import NextPaymentCard from '@/components/loan/NextPaymentCard';
import SelfServiceSection from '@/components/selfservice/SelfServiceSection';

export default async function DashboardPage() {
  const session = await getServerSession(authOptions);
  if (!session) redirect('/login');

  // Required: crash and return error page if loan or client fetch fails
  const [loan, client] = await Promise.all([
    getLoan(session.user.loanKey),
    getClient(session.user.loanKey), // clientKey is fetched from loan.accountHolderKey internally
  ]);

  // Optional: payment schedule – return null if the call fails (AC6)
  let schedule = null;
  try {
    schedule = await getSchedule(session.user.loanKey);
  } catch {
    // Non-blocking: dashboard is shown without payment schedule
  }

  const nextInstallment = schedule?.installments.find(
    (i: { state: string }) => i.state === 'PENDING'
  ) ?? null;

  return (
    <main className="p-6 max-w-5xl mx-auto">
      <LoanSummaryCard loan={loan} />
      <div className="grid grid-cols-1 md:grid-cols-2 gap-6 mt-6">
        <NextPaymentCard installment={nextInstallment} />
        <ClientCard client={client} />
      </div>
      <SelfServiceSection loan={loan} />
    </main>
  );
}
```

### Mambu API calls

| Data             | Mambu call                                            |
|------------------|-------------------------------------------------------|
| Loan             | `GET /api/loans/{loanKey}?detailsLevel=FULL`          |
| Client           | `GET /api/clients/{clientKey}`                        |
| Payment schedule | `GET /api/loans/{loanKey}/schedule?detailsLevel=FULL` |

### TypeScript types

```typescript
// types/index.ts
export interface LoanSummary {
  encodedKey: string;
  id: string;
  loanName: string;
  loanAmount: number;
  accountState: 'PENDING_APPROVAL' | 'APPROVED' | 'ACTIVE' | 'CLOSED';
  interestSettings: {
    interestRate: number;
  };
  scheduleSettings: {
    repaymentInstallments: number;
    fixedDaysOfMonth: number[];
    scheduleDueDatesMethod: string;
  };
  currency: { currencyCode: string };
}

export interface ClientSummary {
  encodedKey: string;
  id: string;
  firstName: string;
  lastName: string;
}
```

### Formatting of NOK amounts

```typescript
// lib/utils/formatters.ts
export function formatNOK(amount: number): string {
  return new Intl.NumberFormat('nb-NO', {
    style: 'currency',
    currency: 'NOK',
    minimumFractionDigits: 0,
    maximumFractionDigits: 0,
  }).format(amount);
}

export function formatPercent(rate: number): string {
  return new Intl.NumberFormat('nb-NO', {
    style: 'percent',
    minimumFractionDigits: 2,
  }).format(rate / 100);
}
```

---

## 6. Out of Scope

- Display of multiple loans
- Graph/history of payments
- Account statement / transaction log
- Document section (mortgage documents, security documents)
- Push notifications about upcoming due dates
