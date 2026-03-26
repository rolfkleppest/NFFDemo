# User Journey and Screen Flow – MinSide Boliglån

**Project:** NFF Demo – MinSide Mortgage Self-Service
**Date:** 2026-03-25
**Status:** Draft v1.0

---

## 1. Overall screen flow

```
┌─────────────┐
│  /login     │  Login page
│  F01        │
└──────┬──────┘
       │ Successful login (Vipps / Demo)
       ▼
┌─────────────────────────────────────────────────────────────┐
│  /dashboard                                                  │
│  F02 – Loan overview                                        │
│                                                             │
│  • Loan card (name, amount, interest, status)              │
│  • Next installment payment                                 │
│  • Customer card                                            │
│  • Self-service section (three action cards)               │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Self-service                                         │  │
│  │  [Change due date] [Change installment count] [Payment holiday]│  │
│  └───────┬───────────────────────┬───────────────────┬───┘  │
│          │                       │                   │      │
│      [Modal]                 [Modal]             [Modal]    │
│      F04                     F05                 F06        │
│                                                             │
│  [→ View payment schedule]                                  │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────┐
│  /dashboard/betalingsplan│
│  F03 – Payment schedule  │
│                         │
│  • Table 24 installments│
│  • Status per installment│
│  • Summary row          │
│                         │
│  [← Back to overview]   │
└─────────────────────────┘
```

---

## 2. Detailed screen description

### Screen 1: Login (`/login`)

**Purpose:** User authentication

**Displays:**
- Bank logo and app name "MinSide Boliglån"
- "Log in to view your loan details" (ingress)
- Button: "Log in with Vipps" (primary, orange) – only shown when `VIPPS_CLIENT_ID` is set
- Button: "Demo login (sandbox)" – only visible when `DEMO_MODE=true`
- Footer: version, year

**States:**
- Loading: Spinner in button during OIDC redirect
- Error: Red alert box with error message below the buttons
- Session expired: Yellow information box "Your session has expired. Please log in again."

**Navigation options:**
- Successful login → `/dashboard`
- Vipps button → Vipps OIDC (apitest.vipps.no)

---

### Screen 2: Dashboard / Loan overview (`/dashboard`)

**Purpose:** Give the user a complete overview of the loan and access to self-service

**Top navbar (persistent):**
- Left: Bank logo + "MinSide Boliglån"
- Right: User's name + "Log out" button

**Content:**

**Main loan card (large, full width):**
```
┌────────────────────────────────────────────────────────┐
│  SPK Boliglån Annuitet                    ● GODKJENT   │
│  Lånekonto: 99004082                                   │
│                                                        │
│  253 000 NOK        4,00 %        24 terminer          │
│  Loan amount        Interest      Count                │
│                                                        │
│  Due date: 15th of each month                          │
└────────────────────────────────────────────────────────┘
```

**Two-column section (medium screens and above):**

| Left: Next installment payment    | Right: Your profile        |
|-----------------------------------|----------------------------|
| Due date: 15 April 2026           | Regan Stracke              |
| Total: 11 250 NOK                 | Customer no.: 885603059    |
| Principal: 9 890 NOK              |                            |
| Interest: 1 360 NOK               |                            |
| [View payment schedule →]         |                            |

**Self-service section:**

Three cards in a row (horizontal on desktop, stacked on mobile):

| Card 1                    | Card 2                      | Card 3                    |
|---------------------------|-----------------------------|---------------------------|
| Change due date           | Change installment count    | Payment holiday           |
| Current: day 15           | Current: 24 installments    | Pause principal payments  |
| [Change →]                | [Change →]                  | 1–12 months               |
|                           |                             | [Apply →]                 |

**States:**
- Loading: Skeleton in all cards
- Error: Alert box with "Try again" button
- After self-service: Green success message (toast/alert at the top)

---

### Screen 3: Payment schedule (`/dashboard/betalingsplan`)

**Purpose:** Complete overview of all 24 installments

**Displays:**
- Back link: "← Back to overview"
- Heading: "Payment schedule – SPK Boliglån Annuitet"
- Subtext: "24 installments | Interest: 4.00% | Due date: 15th of each month"
- Table with 24 rows + summary row
- Disclaimer about estimates

**Table columns:**
`Installment | Due date | Principal | Interest | Fee | Total | Status`

**Status badges:**
- Green ✓ "Paid"
- Orange ⚠ "Due soon" (within 7 days)
- Blue ○ "Future"
- Red ✗ "Overdue"

**States:**
- Loading: Skeleton table (5 rows)
- Error: Alert box

---

### Modal 1: Change due date (F04)

**Trigger:** Click "Change →" in self-service card 1

**Step 1 – Select day:**
- Heading: "Change due date"
- Show current: "Current due date: 15th of each month"
- Day picker: Grid with numbers 1–28 (clickable boxes)
- Current day (15) is highlighted
- Information text about days 29–31
- Buttons: [Cancel] [Next →]

**Step 2 – Confirmation:**
- Heading: "Confirm change of due date"
- Summary box: From / To / Effective date
- Warning about differing amount in the first installment
- Buttons: [← Back] [Confirm change ✓]

**States:**
- Loading: Spinner in "Confirm" button
- Success: Close modal + green toast
- Error: Red error message in modal

---

### Modal 2: Change installment count (F05)

**Trigger:** Click "Change →" in self-service card 2

**Step 1 – Select number of installments:**
- Heading: "Change installment count"
- Show current: "Current: 24 installments (approx. 11 250 NOK/month)"
- Input field + slider (6–360)
- Live-updated cost summary (estimates)
- Warning about increased interest cost when extending
- Buttons: [Cancel] [Next →]

**Step 2 – Confirmation:**
- Comparison table: Current vs. New
- Buttons: [← Back] [Confirm change ✓]

---

### Modal 3: Payment holiday (F06)

**Trigger:** Click "Apply →" in self-service card 3

**Step 1 – Select period:**
- Heading: "Payment holiday"
- Explanation of what a payment holiday is
- Grid with 1–12 months (clickable boxes)
- Live cost overview (monthly interest, total cost, additional cost, new last installment)
- Information text about the additional interest cost
- Buttons: [Cancel] [Next →]

**Step 2 – Confirmation:**
- Cost overview
- Warning in orange/red: "Payment holiday is not free"
- Required confirmation checkbox
- Buttons: [← Back] [Confirm payment holiday ✓] (disabled until checkbox is checked)

---

## 3. Design system and visual identity

### Color palette

| Color              | Hex code   | Usage                                        |
|--------------------|------------|----------------------------------------------|
| Navy (primary)     | `#0f2044`  | Top bar, main loan card background           |
| Dark blue          | `#1a3a6b`  | Secondary background, hover states           |
| Light blue (accent)| `#4f8ef7`  | Primary buttons, links, active elements      |
| White              | `#ffffff`  | Card background, text on dark background     |
| Grey (light)       | `#f8fafc`  | Page background                              |
| Success (green)    | `#16a34a`  | Paid installments, success messages          |
| Warning (yellow)   | `#d97706`  | Due soon, warnings                           |
| Error (red)        | `#dc2626`  | Overdue, error messages                      |
| Neutral (grey)     | `#6b7280`  | Secondary text, disabled elements            |

### Typography

| Element        | Font       | Size  | Weight |
|----------------|------------|-------|--------|
| Heading H1     | Inter      | 28px  | 700    |
| Heading H2     | Inter      | 22px  | 600    |
| Heading H3     | Inter      | 18px  | 600    |
| Body text      | Inter      | 16px  | 400    |
| Label          | Inter      | 14px  | 500    |
| Small text     | Inter      | 12px  | 400    |

### Card design

```
Background: white (#ffffff)
Border: 1px solid #e2e8f0
Border-radius: 12px
Box-shadow: 0 1px 3px rgba(0,0,0,0.1)
Padding: 24px
```

### Button styles

| Type           | Background  | Text     | Hover       |
|----------------|-------------|----------|-------------|
| Primary        | `#4f8ef7`   | White    | `#3b7de8`   |
| Secondary      | Transparent | `#4f8ef7`| `#f0f4ff`   |
| Danger/cancel  | Transparent | `#6b7280`| `#f8fafc`   |
| Disabled       | `#e2e8f0`   | `#9ca3af`| (none)      |

---

## 4. Responsive design

### Breakpoints (Tailwind)

| Name   | Width    | Layout                                     |
|--------|----------|--------------------------------------------|
| sm     | 640px+   | Mobile (1 column)                          |
| md     | 768px+   | Tablet (2 columns)                         |
| lg     | 1024px+  | Desktop (3 columns, full width)            |
| xl     | 1280px+  | Wide desktop (max 1280px container)        |

### Mobile adaptations

- Top navbar: Collapsed (hamburger) or simplified
- Self-service cards: Stacked vertically (1 column)
- Payment schedule table: Horizontal scrolling with sticky left column
- Modals: Full-screen on mobile (`max-h-screen overflow-y-auto`)

---

## 5. Micro-interactions and animations

| Action                        | Animation                                   |
|-------------------------------|---------------------------------------------|
| Modal opens                   | Fade-in + slide up (200ms ease-out)         |
| Modal closes                  | Fade-out (150ms ease-in)                    |
| Success message (toast)       | Slide in from upper right corner            |
| Skeleton-loading              | Pulsing shimmer effect                      |
| Day/month picker (click)      | Scale 0.95 → 1.0 (100ms)                   |
| Button hover                  | Background colour transition (150ms)        |
| Live calculation              | Numbers count up/down (300ms)               |

---

## 6. Accessibility (WCAG 2.1 AA)

| Requirement                    | Implementation                                          |
|--------------------------------|---------------------------------------------------------|
| Focus visibility               | `focus:ring-2 focus:ring-blue-500` on all interactive elements |
| Colour contrast text           | Minimum 4.5:1 for normal text, 3:1 for large text      |
| Keyboard navigation (modals)   | Focus trap in open modals, Escape closes               |
| ARIA labels                    | `aria-label` on icons without text                     |
| Screen reader announcements    | `aria-live="polite"` on dynamic statuses               |
| Semantic HTML                  | `<main>`, `<nav>`, `<section>`, `<table>`, `<caption>` |

---

## 7. Norwegian text guide (UI copy)

### Key terms

| Norwegian term          | Used in                           |
|-------------------------|-----------------------------------|
| Terminbeløp             | Månedlig betaling (avdrag+renter) |
| Avdrag                  | Principal payment                 |
| Renter                  | Interest                          |
| Forfallsdato            | Due date                          |
| Terminlengde            | Number of installments            |
| Avdragsfrihet           | Payment holiday / grace period    |
| Restgjeld               | Outstanding balance               |
| Effektiv rente          | Effective interest rate           |
| Nominell rente          | Nominal interest rate             |
| Lånekonto               | Loan account                      |
| Neste termin            | Next installment                  |
| Betalingsplan           | Payment schedule / amortization   |

### Error messages (tone of voice)

- Clear and direct
- No technical jargon
- Always state what the user can do next
- Example: "Endringen kunne ikke gjennomføres. Prøv igjen, eller kontakt banken på 815 00 xxx."
