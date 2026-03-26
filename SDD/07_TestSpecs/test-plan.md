# Test plan – MinSide Boliglån Self-Service

**Project:** NFF Demo – MinSide Mortgage Self-Service
**Date:** 2026-03-25
**Status:** Draft v1.0

---

## 1. Test scope and strategy

### 1.1 Test levels

| Level             | Tool                          | What is tested                                          |
|-------------------|-------------------------------|---------------------------------------------------------|
| Unit tests        | Jest + React Testing Library  | Components, utility functions, calculators              |
| Integration tests | Jest + MSW (Mock Service Worker)| API routes (Next.js Route Handlers)                   |
| E2E tests         | Playwright                    | User flows end-to-end (with Mambu sandbox)             |
| Manual tests      | Browser                       | Visual check, responsive design                        |

### 1.2 Test environment

| Environment | URL                                    | Auth         | Mambu   |
|-------------|----------------------------------------|--------------|---------|
| Local       | http://localhost:3000                  | Vipps / Demo | Sandbox |
| Staging     | https://nffdemo-staging.vercel.app     | Demo         | Sandbox |
| Demo        | https://nffdemo.vercel.app             | Demo         | Sandbox |

### 1.3 Sandbox data

- **Loan object:** `8a19dc979d254c1d019d2630046f7e3d` (loan account 99004082)
- **Client:** `8a19b6a69d255395019d262ee2a572f4` (Regan Stracke, customer no. 885603059)
- **Interest rate:** 4.0%
- **Installment count:** 24
- **Due date:** day 15

---

## 2. F01 – Login (Vipps / Demo)

### 2.1 Happy Path

| Test ID  | Test case                                       | Steps                                                             | Expected result                                       |
|----------|-------------------------------------------------|-------------------------------------------------------------------|-------------------------------------------------------|
| F01-HP-1 | Demo login                                      | 1. Go to /login<br>2. Click "Demo-innlogging"                    | Redirect to /dashboard. Regan Stracke shown in navbar |
| F01-HP-2 | Redirect to login when unauthorised             | 1. Navigate directly to /dashboard without a session             | Redirect to /login                                    |
| F01-HP-3 | Log out                                         | 1. Log in<br>2. Click "Logg ut"                                  | Redirect to /login. Cookie deleted.                   |

### 2.2 Error scenarios

| Test ID   | Test case                                       | Steps                                                             | Expected result                                           |
|-----------|-------------------------------------------------|-------------------------------------------------------------------|-----------------------------------------------------------|
| F01-ERR-1 | Demo login with DEMO_MODE=false                 | 1. Set DEMO_MODE=false<br>2. Attempt demo login                  | Login fails. Error message shown.                         |
| F01-ERR-2 | Session expires                                 | 1. Log in<br>2. Set JWT to expired<br>3. Navigate to /dashboard  | Redirect to /login. Message: "Din økt er utløpt."        |
| F01-ERR-3 | Protect API routes                              | 1. Call GET /api/loan without a session                          | 401 Unauthorized. `{ "error": "Unauthorized" }`           |

### 2.3 Unit tests

```typescript
// __tests__/auth/sessionGuard.test.ts
describe('SessionGuard', () => {
  it('redirects to /login when no session', async () => {
    mockGetServerSession.mockResolvedValue(null);
    const { redirect } = await import('next/navigation');
    await renderSessionGuard();
    expect(redirect).toHaveBeenCalledWith('/login');
  });

  it('renders children when session exists', async () => {
    mockGetServerSession.mockResolvedValue({ user: { id: '123' } });
    const { getByText } = await renderSessionGuard(<div>Protected content</div>);
    expect(getByText('Protected content')).toBeInTheDocument();
  });
});
```

---

## 3. F02 – Loan overview

### 3.1 Happy Path

| Test ID  | Test case                                       | Steps                                                  | Expected result                                                 |
|----------|-------------------------------------------------|--------------------------------------------------------|-----------------------------------------------------------------|
| F02-HP-1 | Loan card shows correct data                    | 1. Log in<br>2. View /dashboard                        | Loan name "SPK Boliglån Annuitet", 253 000 NOK, 4.00%, 24 installments |
| F02-HP-2 | Customer card shows name and customer number    | 1. Log in<br>2. View customer card                     | "Regan Stracke", Customer no. 885603059                        |
| F02-HP-3 | Next installment payment is displayed           | 1. Log in<br>2. View "Next installment payment" card   | Due date, installment amount, principal and interest displayed  |
| F02-HP-4 | Self-service section is displayed               | 1. View dashboard                                      | Three self-service cards shown with correct information         |
| F02-HP-5 | Link to payment schedule works                  | 1. Click "Se betalingsplan →"                         | Navigates to /dashboard/betalingsplan                           |

### 3.2 Error scenarios

| Test ID   | Test case                                       | Steps                                         | Expected result                                                 |
|-----------|-------------------------------------------------|-----------------------------------------------|-----------------------------------------------------------------|
| F02-ERR-1 | Mambu API unavailable                           | 1. Mock Mambu GET /loans to return 503        | Error message "Kunne ikke hente lånedata. Prøv igjen."         |
| F02-ERR-2 | Invalid loan object (404)                       | 1. Mock Mambu GET /loans to return 404        | Error message shown. No crash.                                 |
| F02-ERR-3 | Network error (timeout)                         | 1. Mock fetch to throw NetworkError           | Error message shown. "Try again" button is visible.            |

### 3.3 Unit tests

```typescript
// __tests__/components/LoanSummaryCard.test.tsx
describe('LoanSummaryCard', () => {
  const mockLoan = {
    loanName: 'SPK Boliglån Annuitet',
    id: '99004082',
    loanAmount: 253000,
    accountState: 'APPROVED',
    interestSettings: { interestRate: 4.0 },
    scheduleSettings: { repaymentInstallments: 24, fixedDaysOfMonth: [15] },
  };

  it('renders loan name', () => {
    render(<LoanSummaryCard loan={mockLoan} />);
    expect(screen.getByText('SPK Boliglån Annuitet')).toBeInTheDocument();
  });

  it('formats loan amount in NOK', () => {
    render(<LoanSummaryCard loan={mockLoan} />);
    expect(screen.getByText('253 000 kr')).toBeInTheDocument();
  });

  it('shows APPROVED status badge', () => {
    render(<LoanSummaryCard loan={mockLoan} />);
    expect(screen.getByText('GODKJENT')).toBeInTheDocument();
  });
});
```

---

## 4. F03 – Payment schedule

### 4.1 Happy Path

| Test ID  | Test case                                           | Steps                                              | Expected result                                              |
|----------|-----------------------------------------------------|----------------------------------------------------|--------------------------------------------------------------|
| F03-HP-1 | All 24 installments are shown                       | 1. Navigate to /dashboard/betalingsplan            | Table with 24 rows + 1 summary row                           |
| F03-HP-2 | Paid installments have green badge                  | 1. View table                                      | Installment 1–2 (state: PAID) have green "Betalt" badge     |
| F03-HP-3 | Future installment has neutral badge                | 1. View table                                      | Future installments have "Fremtidig" badge                   |
| F03-HP-4 | Summary row totals correctly                        | 1. View bottom of table                            | Total principal = 253 000 NOK (equals loan amount)           |
| F03-HP-5 | Back link works                                     | 1. Click "← Tilbake til oversikt"                  | Navigates back to /dashboard                                 |

### 4.2 Error scenarios

| Test ID   | Test case                                           | Steps                                             | Expected result                                              |
|-----------|-----------------------------------------------------|---------------------------------------------------|--------------------------------------------------------------|
| F03-ERR-1 | Schedule endpoint fails                             | 1. Mock GET /schedule to 500                      | Error message "Kunne ikke hente betalingsplan."              |
| F03-ERR-2 | Empty schedule (no installments)                    | 1. Mock schedule to empty list                    | Message "Ingen terminer funnet for dette lånet."             |

### 4.3 Unit tests

```typescript
// __tests__/components/PaymentScheduleTable.test.tsx
describe('PaymentScheduleTable', () => {
  it('shows correct status badge for PAID installment', () => {
    const installments = [
      { number: 1, state: 'PAID', dueDate: '2026-02-15', /* ... */ }
    ];
    render(<PaymentScheduleTable installments={installments} />);
    expect(screen.getByText('Betalt')).toBeInTheDocument();
  });

  it('highlights installment due within 7 days', () => {
    const soon = new Date();
    soon.setDate(soon.getDate() + 3);
    const installments = [
      { number: 3, state: 'PENDING', dueDate: soon.toISOString().split('T')[0] }
    ];
    render(<PaymentScheduleTable installments={installments} />);
    expect(screen.getByText('Forfaller snart')).toBeInTheDocument();
  });
});
```

---

## 5. F04 – Change Due Date

### 5.1 Happy Path

| Test ID  | Test case                                         | Steps                                                                   | Expected result                                                    |
|----------|---------------------------------------------------|-------------------------------------------------------------------------|--------------------------------------------------------------------|
| F04-HP-1 | Modal opens with current day                      | 1. Click "Endre" in the due date card                                   | Modal opens. "Nåværende forfallsdato: 15. hver måned" shown.       |
| F04-HP-2 | Select day 20, confirm                            | 1. Click day 20 in grid<br>2. Click "Neste"<br>3. Click "Bekreft endring"| Success toast: "Due date changed to the 20th of each month"       |
| F04-HP-3 | Dashboard updates after change                    | 1. After F04-HP-2                                                       | Dashboard shows new due date day 20                                |
| F04-HP-4 | Cancel closes without API call                    | 1. Open modal<br>2. Click "Avbryt"                                      | Modal closes. No PATCH call to Mambu.                             |

### 5.2 Error scenarios

| Test ID   | Test case                                         | Steps                                              | Expected result                                                  |
|-----------|---------------------------------------------------|----------------------------------------------------|------------------------------------------------------------------|
| F04-ERR-1 | Invalid day (0) sent to API                       | POST /api/loan/forfallsdato with `{ day: 0 }`      | 400 Bad Request. Error message "Velg en dag mellom 1 og 28."    |
| F04-ERR-2 | Invalid day (29) sent to API                      | POST /api/loan/forfallsdato with `{ day: 29 }`     | 400 Bad Request.                                                 |
| F04-ERR-3 | Mambu PATCH fails (502)                           | 1. Mock Mambu PATCH to 500<br>2. Select day and confirm | Error message in modal: "Endringen kunne ikke gjennomføres."  |
| F04-ERR-4 | Same day selected                                 | 1. Select day 15 (current day)<br>2. Click "Neste" | Message "Du har allerede valgt dag 15". Confirm disabled.        |
| F04-ERR-5 | Double-click on confirm                           | 1. Click "Bekreft" twice rapidly                   | Only one API call sent (button disabled during loading).         |

### 5.3 Unit tests – API Route

```typescript
// __tests__/api/forfallsdato.test.ts
describe('POST /api/loan/forfallsdato', () => {
  it('returns 400 for day 0', async () => {
    const req = new Request('http://localhost/api/loan/forfallsdato', {
      method: 'POST',
      body: JSON.stringify({ day: 0 }),
    });
    const res = await POST(req);
    expect(res.status).toBe(400);
  });

  it('returns 400 for day 29', async () => {
    const req = new Request('http://localhost/api/loan/forfallsdato', {
      method: 'POST',
      body: JSON.stringify({ day: 29 }),
    });
    const res = await POST(req);
    expect(res.status).toBe(400);
  });

  it('calls patchLoan with correct body for valid day', async () => {
    mockGetServerSession.mockResolvedValue({ user: { loanKey: 'test-key' } });
    mockPatchLoan.mockResolvedValue({ scheduleSettings: { fixedDaysOfMonth: [20] } });

    const req = new Request('http://localhost/api/loan/forfallsdato', {
      method: 'POST',
      body: JSON.stringify({ day: 20 }),
    });
    const res = await POST(req);

    expect(mockPatchLoan).toHaveBeenCalledWith('test-key', {
      scheduleSettings: {
        fixedDaysOfMonth: [20],
        scheduleDueDatesMethod: 'FIXED_DAYS_OF_MONTH',
      },
    });
    expect(res.status).toBe(200);
  });
});
```

---

## 6. F05 – Change Installment Count

### 6.1 Happy Path

| Test ID  | Test case                                         | Steps                                                                      | Expected result                                                 |
|----------|---------------------------------------------------|----------------------------------------------------------------------------|-----------------------------------------------------------------|
| F05-HP-1 | Modal opens with current installment count        | 1. Click "Endre" in the installment count card                             | Modal opens. "Nåværende: 24 terminer" shown.                    |
| F05-HP-2 | Change to 36 installments                         | 1. Type 36 in input<br>2. Click "Neste"<br>3. Click "Bekreft endring"      | Success toast: "Terminlengden er endret til 36 terminer."       |
| F05-HP-3 | Live calculation updates                          | 1. Open modal<br>2. Change input to 36                                     | Estimated installment amount and interest cost update instantly.|
| F05-HP-4 | Change to 12 installments (shorter term)          | 1. Type 12 in input<br>2. Confirm                                          | Success. Installment amount increases, total interest decreases.|

### 6.2 Error scenarios

| Test ID   | Test case                                         | Steps                                               | Expected result                                              |
|-----------|---------------------------------------------------|-----------------------------------------------------|--------------------------------------------------------------|
| F05-ERR-1 | Installment count below minimum (5)               | POST with `{ installments: 5 }`                     | 400 Bad Request. "Minimum 6 terminer er påkrevd."            |
| F05-ERR-2 | Installment count above maximum (361)             | POST with `{ installments: 361 }`                   | 400 Bad Request. "Maksimalt 360 terminer."                   |
| F05-ERR-3 | Non-integer (12.5)                                | POST with `{ installments: 12.5 }`                  | 400 Bad Request.                                             |
| F05-ERR-4 | Mambu PATCH fails                                 | Mock Mambu to 409 (conflict)                        | Error message in modal.                                      |

### 6.3 Unit tests – Calculator

```typescript
// __tests__/utils/calculations.test.ts
describe('calculateMonthlyPayment', () => {
  it('calculates correctly for 253000 NOK at 4% over 24 months', () => {
    const result = calculateMonthlyPayment(253000, 4.0, 24);
    expect(result).toBeCloseTo(11000, -2); // approx. 11 000 NOK
  });

  it('returns principal / installments for 0% interest rate', () => {
    const result = calculateMonthlyPayment(120000, 0, 12);
    expect(result).toBeCloseTo(10000, 0);
  });
});

describe('calculateTotalInterest', () => {
  it('returns 0 total interest for 0% rate', () => {
    const monthly = calculateMonthlyPayment(120000, 0, 12);
    expect(calculateTotalInterest(120000, monthly, 12)).toBeCloseTo(0, 0);
  });
});
```

---

## 7. F06 – Payment holiday

### 7.1 Happy Path

| Test ID  | Test case                                             | Steps                                                                          | Expected result                                                          |
|----------|-------------------------------------------------------|--------------------------------------------------------------------------------|--------------------------------------------------------------------------|
| F06-HP-1 | Modal opens with explanatory text                     | 1. Click "Søk" in the payment holiday card                                     | Modal opens. Explanation of payment holiday shown. Loan status shown.    |
| F06-HP-2 | 3 months payment holiday – confirm                    | 1. Select 3 months<br>2. Click "Neste"<br>3. Check checkbox<br>4. Click confirm | Success toast: "Avdragsfrihet i 3 måneder er innvilget."                |
| F06-HP-3 | Cost calculator updates live                          | 1. Click 1, then 6, then 12 months                                             | Cost figures (interest/month, total, additional cost) update on each click.|
| F06-HP-4 | Confirm button is disabled without checkbox           | 1. Click "Neste" without checking checkbox                                     | Confirm button is greyed out. Cannot be clicked.                         |
| F06-HP-5 | 1 and 12 months are valid                             | 1. Select 1 month, confirm<br>2. Select 12 months, confirm                     | Both work without validation errors.                                     |

### 7.2 Error scenarios

| Test ID   | Test case                                             | Steps                                               | Expected result                                                    |
|-----------|-------------------------------------------------------|-----------------------------------------------------|--------------------------------------------------------------------|
| F06-ERR-1 | 13 months sent to API                                 | POST with `{ months: 13 }`                          | 400 Bad Request. "Maksimalt 12 måneder."                           |
| F06-ERR-2 | 0 months sent                                         | POST with `{ months: 0 }`                           | 400 Bad Request. "Velg minst 1 måneds avdragsfrihet."              |
| F06-ERR-3 | Mambu PATCH fails                                     | Mock Mambu to 500                                   | Error message: "Avdragsfrihet kunne ikke innvilges. Kontakt banken."|

### 7.3 Unit tests – Calculator

```typescript
// __tests__/utils/calculations.test.ts
describe('calculateHolidayCost', () => {
  it('calculates monthly interest-only payment', () => {
    const result = calculateHolidayCost(253000, 4.0, 3, 24);
    // 253000 * (0.04/12) ≈ 843 NOK/month
    expect(result.monthlyInterestOnly).toBeCloseTo(843, 0);
  });

  it('calculates total holiday cost for 3 months', () => {
    const result = calculateHolidayCost(253000, 4.0, 3, 24);
    expect(result.totalHolidayCost).toBeCloseTo(2530, -1);
  });

  it('returns extended last payment date', () => {
    const result = calculateHolidayCost(253000, 4.0, 3, 24);
    expect(result.newLastPaymentDate).toBeInstanceOf(Date);
  });
});
```

---

## 8. API integration tests (MSW)

```typescript
// __tests__/integration/mambu-client.test.ts
import { setupServer } from 'msw/node';
import { rest } from 'msw';
import { getLoan } from '@/lib/mambu/loans';

const server = setupServer(
  rest.get(
    'https://knowit.sandbox.mambu.com/api/loans/:key',
    (req, res, ctx) => {
      return res(ctx.json({
        encodedKey: '8a19dc979d254c1d019d2630046f7e3d',
        id: '99004082',
        loanName: 'SPK Boliglån Annuitet',
        loanAmount: 253000,
        accountState: 'APPROVED',
        interestSettings: { interestRate: 4.0 },
        scheduleSettings: { repaymentInstallments: 24, fixedDaysOfMonth: [15] },
      }));
    }
  )
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('getLoan', () => {
  it('returns loan data from Mambu', async () => {
    const loan = await getLoan('8a19dc979d254c1d019d2630046f7e3d');
    expect(loan.loanName).toBe('SPK Boliglån Annuitet');
    expect(loan.loanAmount).toBe(253000);
  });

  it('throws MambuApiError on 404', async () => {
    server.use(
      rest.get('*', (req, res, ctx) => res(ctx.status(404)))
    );
    await expect(getLoan('nonexistent')).rejects.toThrow('Mambu API error: 404');
  });
});
```

---

## 9. E2E tests (Playwright)

```typescript
// e2e/full-flow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('MinSide Boliglån – Full user flow', () => {

  test('Demo login and dashboard display', async ({ page }) => {
    await page.goto('/login');
    await page.click('button:has-text("Demo-innlogging")'); // Click demo login button
    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('text=SPK Boliglån Annuitet')).toBeVisible();
    await expect(page.locator('text=253 000')).toBeVisible();
  });

  test('View payment schedule', async ({ page }) => {
    await loginAsDemo(page);
    await page.click('text=Se betalingsplan'); // Click view payment schedule
    await expect(page).toHaveURL('/dashboard/betalingsplan');
    await expect(page.locator('table')).toBeVisible();
    const rows = page.locator('tbody tr');
    await expect(rows).toHaveCount(24);
  });

  test('Change due date – happy path', async ({ page }) => {
    await loginAsDemo(page);
    await page.click('text=Endre forfallsdato'); // Click change due date
    await expect(page.locator('[role=dialog]')).toBeVisible();
    await page.click('button:has-text("20")'); // Select day 20
    await page.click('button:has-text("Neste")'); // Click next
    await page.click('button:has-text("Bekreft endring")'); // Click confirm change
    await expect(page.locator('text=Forfallsdato er endret til 20')).toBeVisible();
  });

  test('Change installment count – validation (below minimum)', async ({ page }) => {
    await loginAsDemo(page);
    await page.click('text=Endre terminlengde'); // Click change installment count
    await page.fill('input[type=number]', '3');
    await expect(page.locator('text=Minimum 6 terminer')).toBeVisible();
    await expect(page.locator('button:has-text("Neste")')).toBeDisabled(); // Next button disabled
  });

  test('Payment holiday – confirm button disabled without checkbox', async ({ page }) => {
    await loginAsDemo(page);
    await page.click('text=Avdragsfrihet'); // Click payment holiday
    await page.click('button:has-text("3")'); // 3 months
    await page.click('button:has-text("Neste")'); // Click next
    const confirmBtn = page.locator('button:has-text("Bekreft avdragsfrihet")'); // Confirm payment holiday button
    await expect(confirmBtn).toBeDisabled();
    await page.check('input[type=checkbox]');
    await expect(confirmBtn).toBeEnabled();
  });

  test('Log out works', async ({ page }) => {
    await loginAsDemo(page);
    await page.click('button:has-text("Logg ut")'); // Click log out button
    await expect(page).toHaveURL('/login');
  });
});

async function loginAsDemo(page: any) {
  await page.goto('/login');
  await page.click('button:has-text("Demo-innlogging")'); // Click demo login button
  await page.waitForURL('/dashboard');
}
```

---

## 10. Performance tests

| Test                                   | Tool       | Acceptance criterion                    |
|----------------------------------------|------------|-----------------------------------------|
| Lighthouse Score – Desktop             | Lighthouse | Performance > 80, Accessibility > 90   |
| Lighthouse Score – Mobile              | Lighthouse | Performance > 70                        |
| Time to First Byte (TTFB)              | WebPageTest| < 500ms                                 |
| Largest Contentful Paint (LCP)         | Core Web Vitals | < 2.5s                           |
| First Input Delay (FID)                | Core Web Vitals | < 100ms                          |

---

## 11. Manual checklist

### Visual check

- [ ] Dashboard looks correct on mobile (375px), tablet (768px) and desktop (1280px)
- [ ] Payment schedule table scrolls horizontally on mobile
- [ ] All modals are visible and usable on mobile
- [ ] Colour contrast is readable (white text on navy background)
- [ ] Loading skeleton displays correctly
- [ ] Success/error toast messages appear and disappear after 5 seconds

### Accessibility

- [ ] All buttons can be reached with the Tab key
- [ ] Modals trap focus (no Tab out of modal)
- [ ] Escape closes open modals
- [ ] Screen reader announces dynamic updates
- [ ] Vipps button has correct aria-label

### Security

- [ ] `.env` variables are not exposed in client code (check Network tab)
- [ ] Direct calls to `knowit.sandbox.mambu.com` from the browser give a CORS error
- [ ] API routes return 401 without a valid session
- [ ] Cookie is HttpOnly and Secure (production)
