# F04 – Change Due Date

**Feature ID:** F04
**Priority:** Must
**Status:** Draft v1.0
**Date:** 2026-03-25

---

## 1. Description

The user can change the due date for their monthly installment payments. The current due date is the 15th of the month (`fixedDaysOfMonth: [15]`). The user selects a new day (1–28) and confirms the change via a dialog. The change is sent to Mambu via a POST call using Mambu's colon-action pattern: `POST /api/loans/{loanKey}:changeDueDatesSettings`.

Valid days are limited to 1–28 to avoid issues with months that have fewer than 29/30/31 days (e.g. February).

---

## 2. User story

> **As** a logged-in bank customer
> **I want** to change which day of the month my installment payment is due
> **So that** the payment date better aligns with my salary date or financial situation

---

## 3. Acceptance criteria

### AC1 – Open dialog

**Given** that the user is on the dashboard
**When** they click "Change" in the self-service card "Change due date"
**Then** a modal dialog opens
**And** the current due date (day 15) is clearly shown in the dialog

### AC2 – Day selection options

**Given** that the dialog is open
**When** the user sees the day picker
**Then** they can select a day from 1 to 28 (inclusive)
**And** days 29, 30, 31 are not available (greyed out or not shown)
**And** the current day (15) is pre-selected

### AC3 – Confirmation dialog

**Given** that the user has selected a new day (e.g. day 20)
**When** they click "Next" or "Confirm"
**Then** a summary is shown:
- "Current due date: 15th of each month"
- "New due date: 20th of each month"
- "The change takes effect from the next installment"
**And** the user must actively confirm by clicking "Confirm change"

### AC4 – Successful change

**Given** that the user confirms the change
**When** the POST call to Mambu returns 204 No Content
**Then** the dialog closes
**And** a success message is shown: "Due date has been changed to the 20th of each month"
**And** the dashboard is updated with the new due date
**And** the SWR/cache is invalidated so that new data is fetched

### AC5 – Error from Mambu

**Given** that the user confirms the change
**When** the POST call to Mambu fails (4xx/5xx)
**Then** an error message is shown in the dialog: "The change could not be completed. Please try again."
**And** the dialog remains open
**And** the old due date is still active

### AC6 – Cancel

**Given** that the dialog is open
**When** the user clicks "Cancel" or closes the dialog (X or Escape key)
**Then** the dialog closes without any change being made
**And** the Mambu API call is NOT executed

### AC7 – Loading during processing

**Given** that the user has confirmed the change
**When** the API call is being processed
**Then** a spinner is shown in the "Confirm change" button
**And** the button is disabled to prevent double-clicking

### AC8 – Same day selected

**Given** that the user opens the dialog
**And** selects the same day that is already set (day 15)
**When** they click "Next"
**Then** a message is shown: "You have already selected day 15 as the due date"
**And** the "Confirm change" button is disabled

---

## 4. UI design

### Step 1 – Select new day

```
┌───────────────────────────────────────────────────────┐
│  Change due date                                 [X]   │
│  ─────────────────────────────────────────────────── │
│                                                       │
│  Current due date: 15th of each month                 │
│                                                       │
│  Select new due date:                                 │
│                                                       │
│  ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐                │
│  │ 1│ │ 2│ │ 3│ │ 4│ │ 5│ │ 6│ │ 7│                │
│  └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘                │
│  ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐                │
│  │ 8│ │ 9│ │10│ │11│ │12│ │13│ │14│                │
│  └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘                │
│  ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐                │
│  │15│ │16│ │17│ │18│ │19│ │20│ │21│                │
│  └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘                │  (15 = selected/highlighted)
│  ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐                │
│  │22│ │23│ │24│ │25│ │26│ │27│ │28│                │
│  └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘                │
│                                                       │
│  ℹ Days 29–31 are not available (calendar reasons)   │
│                                                       │
│  [Cancel]                         [Next →]           │
└───────────────────────────────────────────────────────┘
```

### Step 2 – Confirmation

```
┌───────────────────────────────────────────────────────┐
│  Confirm change of due date                   [X]     │
│  ─────────────────────────────────────────────────── │
│                                                       │
│  You are about to change:                             │
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │  From:         15th of each month               │  │
│  │  To:           20th of each month               │  │
│  │  Takes effect: next installment                 │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  ⚠ Note: The first installment after the change may   │
│    have a different amount if the change is made      │
│    mid-installment period.                            │
│                                                       │
│  [← Back]                     [Confirm change ✓]     │
└───────────────────────────────────────────────────────┘
```

---

## 5. Mambu API call

### Endpoint
```
POST /api/loans/{loanKey}:changeDueDatesSettings
```

> Note: Mambu uses the colon-action pattern (not REST PATCH) for this operation.

### Request headers
```
Authorization: Basic <base64(user:pass)>
apikey: <api-key>
Accept: application/vnd.mambu.v2+json
Content-Type: application/json
```

### Request body
```json
{
  "fixedDaysOfMonth": [20],
  "notes": "Due date changed by customer via MinSide",
  "valueDate": "2026-03-25T12:00:00+01:00"
}
```

| Field              | Type     | Required | Description                                                             |
|--------------------|----------|----------|-------------------------------------------------------------------------|
| `fixedDaysOfMonth` | number[] | Yes      | New day of the month, e.g. `[20]`                                       |
| `notes`            | string   | No       | Log note (visible in the Mambu audit trail)                             |
| `valueDate`        | ISO8601  | Yes      | Timestamp for when the change applies. **Must use local timezone offset (+01:00), NOT UTC (Z).** Mambu returns `INVALID_DATE` (400) if UTC is used. Use `toLocaleString('sv-SE', { timeZoneName: 'longOffset' })` pattern. |

### Expected response
```
204 No Content
```
Empty response body. Success is confirmed solely via the HTTP status code.

---

## 6. Next.js API Route

### `app/api/loan/forfallsdato/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth/authOptions';
import { patchLoan } from '@/lib/mambu/loans';

export async function POST(request: NextRequest) {
  const session = await getServerSession(authOptions);
  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const body = await request.json();
  const { day } = body;

  // Validate day (1-28)
  if (!Number.isInteger(day) || day < 1 || day > 28) {
    return NextResponse.json(
      { error: 'Invalid day. Select a day between 1 and 28.' },
      { status: 400 }
    );
  }

  try {
    // valueDate MUST use local timezone offset (+01:00), NOT UTC (Z).
    // Mambu sandbox returns INVALID_DATE if UTC format is used.
    const valueDate = new Date().toLocaleString('sv-SE', { timeZoneName: 'longOffset' })
      .replace(' GMT+', '+').replace(' ', 'T').replace(/(\+\d{2})(\d{2})$/, '$1:$2');

    await mambuPost(`loans/${session.user.loanKey}:changeDueDatesSettings`, {
      fixedDaysOfMonth: [day],
      notes: 'Due date changed by customer via MinSide',
      valueDate,
    });

    // 204 No Content – no response body from Mambu
    return NextResponse.json({ success: true, newDay: day });
  } catch (error) {
    return NextResponse.json(
      { error: 'Could not update due date.' },
      { status: 502 }
    );
  }
}
```

---

## 7. Validation rules

| Rule       | Description                                      | Error message                          |
|------------|--------------------------------------------------|----------------------------------------|
| Day 1–28   | Only days 1–28 are valid                         | "Select a day between 1 and 28"        |
| Day 29–31  | Not permitted                                    | "This day is not available"            |
| Same day   | No change if selected day = current day          | "Day already selected"                 |
| Logged in  | Session must be active                           | Redirect to login                      |

---

## 8. Blackout period for changes near the due date

The spec states that the change takes effect "from the next installment". If the user changes the due date on the same day as (or close to) the upcoming due date, the result may be ambiguous.

**Handling:**
- The app displays a warning if the next due date is ≤ 3 days away: *"The next installment is due in X days. The change will take effect from the installment after that."*
- The actual validation is left to Mambu – the app does not enforce a lock period beyond the informational warning
- Mambu returns 4xx if the change is rejected due to timing; the app displays a generic error message

---

## 9. Out of Scope

- Changing the due date to a specific date (not just day-of-month)
- Temporary deferral of a single installment
- Changing the due date retroactively
- Notifying the user via SMS/email about the change
