# F06 – Betalingsfritak (Payment Holiday)

**Feature ID:** F06
**Priority:** Must
**Status:** Draft v1.1
**Date:** 2026-03-25

---

## 1. Description

**Betalingsfritak** allows the user to temporarily pause all payments on the loan (both principal and interest). During the betalingsfritak period the customer pays **nothing** – Mambu sets both `principal.amount.expected` and `interest.amount.expected` to 0 on the selected installments (`isPaymentHoliday: true`). This corresponds to Norwegian banking term **betalingsfritak**, NOT "avdragsfrihet" (which would mean paying only interest). The deferred principal and interest accrue and are redistributed over the remaining term, increasing total loan cost.

The user can choose 1–12 months of betalingsfritak. The app calculates and displays the total additional cost (extra interest) so that the user can make an informed decision.

> **Note:** This is implemented via a two-step read-modify-write on the Mambu schedule – see section 6.

---

## 2. User Story

> **As** a logged-in bank customer
> **I want** to apply for betalingsfritak for 1–12 months
> **So that** I can pause all loan payments during a period of changed financial circumstances,
> and I understand what this costs me in extra interest over the loan lifetime

---

## 3. Acceptance Criteria

### AC1 – Open dialog

**Given** that the user is on the dashboard
**When** they click "Søk om" in the self-service card "Betalingsfritak"
**Then** a modal dialog opens with information about betalingsfritak
**And** current loan information (outstanding balance, interest rate) is displayed

### AC2 – Select number of months

**Given** that the dialog is open
**When** the user sees the month selector
**Then** they can select 1 to 12 months
**And** the default/pre-selected value is 3 months
**And** selections over 12 months are not possible

### AC3 – Cost calculator (live)

**Given** that the user has selected a number of months (e.g. 3)
**And** the current outstanding balance is approx. 253 000 NOK at 4.0% interest
**When** the count is changed
**Then** the following is immediately calculated and displayed:
- "Månedlig betaling i perioden: 0 NOK (betalingsfritak)"
- "Utsatt rente per måned: ~843 NOK"
- "Estimert ekstrakostnad over lånets løpetid: ~X NOK"
- "Ny estimert siste termin: [dato]"

### AC4 – Additional cost is explained

**Given** that the costs are shown
**Then** there is an explanation:
> "Under betalingsfritaksperioden betaler du ingenting. Renter og avdrag utsettes og legges til lånesaldoen, noe som øker den totale rentekostnaden over resterende løpetid."

### AC5 – Confirmation dialog (two-step confirmation)

**Given** that the user has selected 3 months of betalingsfritak
**When** they click "Neste"
**Then** a confirmation summary is displayed:

| Felt                                           | Verdi                  |
|------------------------------------------------|------------------------|
| Betalingsfritaksperiode                        | 3 måneder              |
| Månedlig betaling i perioden                   | 0 NOK (betalingsfritak)|
| Utsatt rente per måned                         | ~843 NOK               |
| Estimert ekstrakostnad over lånets løpetid     | ~2 200 NOK             |
| Betalingsfritak starter                        | fra neste termin       |

> **Ekstrakostnaden forklart:**
> Under betalingsfritak betaler du ingenting. Renter og avdrag utsettes og legges til lånesaldoen. Siden utestående saldo ikke reduseres, betaler du renter på det utsatte beløpet gjennom hele resterende løpetid. Ekstrakostnaden over lånets løpetid (≈ 2 200 NOK) er en reell merkostnad.
> Se beregningslogikk i avsnitt 5.

**And** the user sees a warning: "Betalingsfritak er ikke gratis – det koster deg ekstra renter totalt."
**And** the user must actively confirm by clicking "Bekreft betalingsfritak"

### AC6 – Successful change

**Given** that the user confirms
**When** the PUT call to Mambu returns 200 OK
**Then** the dialog closes
**And** a success message is displayed: "Betalingsfritak for 3 måneder er innvilget. Du betaler ingenting i denne perioden."
**And** the dashboard is updated

### AC7 – Error from Mambu

**Given** that the PUT call fails (4xx/5xx)
**Then** the following is displayed: "Betalingsfritak kunne ikke innvilges. Prøv igjen eller kontakt banken."
**And** no change has been made

### AC8 – Maximum limit enforced

**Given** that the user tries to enter 13 months
**Then** 13 is not selectable (the UI prevents this)
**And** an information text explains: "Maksimalt 12 måneder betalingsfritak er tillatt."

### AC9 – Timing: betalingsfritak starts from the next installment

**Given** that the user applies for betalingsfritak
**And** the next due date is ≤ 3 days away
**Then** an information notice is displayed: "Neste termin forfaller om X dager og vil ikke omfattes av betalingsfritaket. Betalingsfritaket starter fra terminen etter."
**And** Mambu handles the actual cutoff – the app does not block beyond the notice

### AC10 – Loading state during processing

**Given** that the user has confirmed
**And** the API call is being processed
**Then** a spinner is shown in the "Bekreft" button
**And** the button is disabled

---

## 4. UI Design

### Step 1 – Select betalingsfritak period

```
┌────────────────────────────────────────────────────────────────┐
│  Betalingsfritak                                         [X]   │
│  ──────────────────────────────────────────────────────────── │
│                                                                │
│  Betalingsfritak lar deg midlertidig pause alle betalinger     │
│  på lånet. Du betaler ingenting i denne perioden.              │
│                                                                │
│  Ditt lån:                                                     │
│  • Utestående saldo: 253 000 NOK                               │
│  • Rente: 4,00 %                                               │
│                                                                │
│  Velg antall måneder betalingsfritak:                          │
│                                                                │
│  ┌──┐  ┌──┐  ┌──┐  ┌──┐  ┌──┐  ┌──┐                         │
│  │ 1│  │ 2│  │ 3│  │ 4│  │ 5│  │ 6│                         │
│  └──┘  └──┘  └──┘  └──┘  └──┘  └──┘                         │
│  ┌──┐  ┌──┐  ┌──┐  ┌──┐  ┌──┐  ┌──┐                         │
│  │ 7│  │ 8│  │ 9│  │10│  │11│  │12│                         │
│  └──┘  └──┘  └──┘  └──┘  └──┘  └──┘                         │
│                                        (Maks 12 måneder)       │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐     │
│  │  📊 Kostnadsoversikt for 3 mnd betalingsfritak:      │     │
│  │                                                      │     │
│  │  Månedlig betaling i perioden:         0 NOK         │     │  ← Oppdateres live
│  │  Utsatt rente per måned:             843 NOK         │     │
│  │  Estimert ekstrakostnad (totalt):   2 200 NOK        │     │
│  │  Ny siste termin:                   April 2028       │     │
│  └──────────────────────────────────────────────────────┘     │
│                                                                │
│  ℹ Under betalingsfritaksperioden betaler du ingenting.        │
│    Renter og avdrag utsettes og øker den totale                │
│    rentekostnaden over lånets løpetid.                         │
│                                                                │
│  [Avbryt]                              [Neste →]              │
└────────────────────────────────────────────────────────────────┘
```

### Step 2 – Confirmation

```
┌────────────────────────────────────────────────────────────────┐
│  Bekreft betalingsfritak                                 [X]  │
│  ──────────────────────────────────────────────────────────── │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐     │
│  │  Betalingsfritaksperiode:          3 måneder         │     │
│  │  Betaling i perioden:              0 NOK             │     │
│  │  Utsatt rente per måned:           843 NOK           │     │
│  │  Ekstrakostnad over lånets løpetid: 2 200 NOK        │     │
│  │  Perioden starter:                 fra neste termin  │     │
│  └──────────────────────────────────────────────────────┘     │
│                                                                │
│  ⚠ VIKTIG: Betalingsfritak er ikke gratis.                     │
│    Du vil betale totalt ca. 2 200 NOK mer i renter             │
│    over resterende låpetid.                                    │
│                                                                │
│  Jeg forstår at betalingsfritak øker mine totale               │
│  rentekostnader.                                               │
│  [☐] Jeg bekrefter at jeg har lest og forstått                 │
│      informasjonen ovenfor.                                    │
│                                                                │
│  [← Tilbake]              [Bekreft betalingsfritak ✓]         │
└────────────────────────────────────────────────────────────────┘
```

**Note:** The "Bekreft" button is disabled until the user checks the confirmation checkbox.

---

## 5. Cost Calculator

### Note on Mambu behavior

Mambu sets **both** `principal.amount.expected` and `interest.amount.expected` to 0 when `isPaymentHoliday: true`. The customer pays **nothing** during the period. However, interest continues to accrue on the outstanding balance and is redistributed over the remaining term, increasing total loan cost.

### Formula

```typescript
// lib/utils/calculations.ts

/**
 * Estimates the additional cost of betalingsfritak.
 * During the period, both principal and interest are 0 (Mambu behavior).
 * However, interest accrues on the outstanding balance and is deferred,
 * increasing total lifetime cost.
 */
export function calculateHolidayCost(
  principal: number,
  annualRate: number,
  holidayMonths: number,
  remainingInstallments: number
): {
  monthlyDeferredInterest: number;
  extraLifetimeInterest: number;
  newLastPaymentDate: Date;
} {
  const monthlyRate = annualRate / 100 / 12;
  // Interest that would accrue per month during the holiday (deferred, not paid)
  const monthlyDeferredInterest = principal * monthlyRate;

  // Estimated additional cost: interest on unrepaid principal over remaining duration
  const normalMonthlyPayment = calculateMonthlyPayment(
    principal, annualRate, remainingInstallments
  );
  const normalMonthlyPrincipal = normalMonthlyPayment - monthlyDeferredInterest;
  const extraInterestPerMonth = normalMonthlyPrincipal * monthlyRate;
  const extraLifetimeInterest = extraInterestPerMonth * (remainingInstallments - holidayMonths);

  const newLastPaymentDate = new Date();
  newLastPaymentDate.setMonth(
    newLastPaymentDate.getMonth() + remainingInstallments + holidayMonths
  );

  return {
    monthlyDeferredInterest,
    extraLifetimeInterest,
    newLastPaymentDate,
  };
}
```

---

## 6. Mambu API Call

> **Confirmed 2026-03-25 via sandbox testing.**
> `POST :applyPaymentHoliday` returns 405. `POST :changeLoanTerm` with gracePeriod returns 500.
> The correct approach is a two-step read-modify-write on the schedule via `PUT /api/loans/{loanKey}/schedule`.

### Confirmed endpoint
```
PUT /api/loans/{loanKey}/schedule
```

### Process (two steps)

**Step 1 – Read current schedule**
```
GET /api/loans/{loanKey}/schedule?detailsLevel=FULL
```
Retrieve the list of installments and their `encodedKey` values.

**Step 2 – Mark N installments as payment holiday**
```
PUT /api/loans/{loanKey}/schedule
```

Body: array of the N first `PENDING` installments with `isPaymentHoliday: true`.

```json
[
  {
    "encodedKey": "<installment encodedKey from GET>",
    "number": "1",
    "dueDate": "2026-03-25T01:00:00+01:00",
    "isPaymentHoliday": true,
    "principal": { "amount": { "expected": 98774 } },
    "interest":  { "amount": { "expected": 3068 } },
    "fee":       { "amount": { "expected": 0 } }
  }
]
```

> Note: `principal.amount.expected` and `interest.amount.expected` should be sent with the current values. Mambu ignores them and sets both to 0 automatically when `isPaymentHoliday: true`.

### Response: 200 OK (with updated installments)

```json
[
  {
    "encodedKey": "...",
    "number": "1",
    "state": "GRACE",
    "isPaymentHoliday": true,
    "principal": { "amount": { "expected": 0, "paid": 0, "due": 0 } },
    "interest":  { "amount": { "expected": 0, "paid": 0, "due": 0 } }
  }
]
```

**Key confirmed behaviours (sandbox):**
- `state` → `"GRACE"`
- `principal.amount.expected` → `0` (set by Mambu)
- `interest.amount.expected` → `0` (set by Mambu – **both** are zeroed: this is betalingsfritak, not avdragsfrihet)
- Response is **200 OK** with the updated installments (not 204 No Content)

---

## 7. Next.js API Route

### `app/api/loan/avdragsfrihet/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from '@/lib/auth/session';
import { getLoan } from '@/lib/mambu/loans';
import { getSchedule } from '@/lib/mambu/schedule';
import { fetchMambu } from '@/lib/mambu/client';

export async function POST(request: NextRequest) {
  const session = await getServerSession();
  if (!session?.user?.loanKey) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { months } = await request.json();

  if (!Number.isInteger(months) || months < 1 || months > 12) {
    return NextResponse.json(
      { error: 'Betalingsfritak må være mellom 1 og 12 måneder.' },
      { status: 400 }
    );
  }

  const { loanKey } = session.user;

  try {
    // Step 1: get loan encodedKey (loanKey may be numeric id)
    const loan = await getLoan(loanKey);
    const loanEncodedKey = loan.encodedKey;

    // Step 2: fetch current schedule to get installment encodedKeys
    const schedule = await getSchedule(loanEncodedKey);
    if (!schedule) throw new Error('Schedule not available');

    // Step 3: select the first N PENDING installments
    const pendingInstallments = schedule.installments
      .filter(i => i.state === 'PENDING')
      .slice(0, months);

    if (pendingInstallments.length === 0) {
      return NextResponse.json({ error: 'Ingen ventende terminer funnet.' }, { status: 400 });
    }

    // Step 4: PUT schedule with isPaymentHoliday: true for selected installments
    const payload = pendingInstallments.map(inst => ({
      encodedKey: inst.encodedKey,
      number: inst.number,
      dueDate: inst.dueDate,
      isPaymentHoliday: true,
      principal: { amount: { expected: inst.principal.amount.expected } },
      interest:  { amount: { expected: inst.interest.amount.expected } },
      fee:       { amount: { expected: inst.fee.amount.expected } },
    }));

    await fetchMambu(`/loans/${loanEncodedKey}/schedule`, {
      method: 'PUT',
      body: JSON.stringify(payload),
    });

    return NextResponse.json({ success: true, gracePeriodMonths: months });
  } catch (error) {
    return NextResponse.json(
      { error: 'Betalingsfritak kunne ikke innvilges. Kontakt banken.' },
      { status: 502 }
    );
  }
}
```

---

## 8. Validation Rules

| Regel                     | Grense   | Feilmelding                                               |
|---------------------------|----------|-----------------------------------------------------------|
| Minimum måneder           | 1        | "Velg minst 1 måned betalingsfritak"                      |
| Maksimum måneder          | 12       | "Maksimalt 12 måneder betalingsfritak er tillatt"         |
| Heltall påkrevd           | N/A      | "Angi et heltall for antall måneder"                      |
| Bekreftelse påkrevd       | Avkrysning | Knapp deaktivert til avkrysning er gjort               |

---

## 9. Mambu isPaymentHoliday – Confirmed behavior

| Felt i Mambu                    | Verdi etter PUT med isPaymentHoliday: true |
|---------------------------------|--------------------------------------------|
| `state`                         | `"GRACE"`                                  |
| `isPaymentHoliday`              | `true`                                     |
| `principal.amount.expected`     | `0` (satt av Mambu)                        |
| `interest.amount.expected`      | `0` (satt av Mambu – **betalingsfritak**)  |

> **NB:** Dette er **betalingsfritak** (kunden betaler ingenting), IKKE **avdragsfrihet** (kunden betaler kun renter). Mambu sin "Payment Holiday" nullstiller begge felt.

---

## 10. Out of Scope

- Avdragsfrihet med rentebetaling (PAY_INTEREST_ONLY via gracePeriodType)
- Godkjenningsprosess med saksbehandler
- Betalingsfritak på deler av lånet
- Historikk over tidligere betalingsfritak
- Varsling via SMS/e-post om innvilget betalingsfritak
