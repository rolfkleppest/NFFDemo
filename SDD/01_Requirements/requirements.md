# Requirements – MinSide Home Loan Self-Service

**Project:** NFF Demo – MinSide Mortgage Self-Service
**Technology:** Next.js 14 (App Router)
**Backend:** Mambu API (knowit.sandbox.mambu.com)
**Date:** 2026-03-25
**Status:** Draft v1.0

---

## 1. Purpose

Demonstrate how Spec Driven Design (SDD) + Claude makes it possible to build a complete self-service banking app for home loans in record time. The app will show real Mambu API integration with authentication via Vipps and a professional user interface in Norwegian.

---

## 2. Functional requirements

### 2.1 Authentication (F01)

| ID    | Requirement                                                                      | Priority |
|-------|----------------------------------------------------------------------------------|----------|
| F01-1 | User can log in with Vipps (OpenID Connect via Vipps test environment)          | Must      |
| F01-2 | Demo mode: Skip Vipps when `DEMO_MODE=true` (direct login)                     | Must      |
| F01-3 | Session expires after 30 minutes                                                 | Must      |
| F01-4 | User is logged out and redirected to the login page at session end              | Must      |
| F01-5 | PKCE flow is used for Vipps OIDC for increased security                        | Must      |

### 2.2 Loan overview (F02)

| ID    | Requirement                                                                        | Priority |
|-------|------------------------------------------------------------------------------------|----------|
| F02-1 | The app fetches loan data from the Mambu API on login                             | Must      |
| F02-2 | Displays loan name, loan amount, outstanding balance, interest rate, and status   | Must      |
| F02-3 | Displays customer information (name, customer number)                             | Must      |
| F02-4 | Displays next due date and installment amount                                     | Must      |
| F02-5 | Data is fetched server-side to avoid CORS and exposure of API keys               | Must      |
| F02-6 | Loading state is shown while data is being fetched                               | Must      |
| F02-7 | Error message is shown if the Mambu API is unavailable                           | Must      |

### 2.3 Payment schedule (F03)

| ID    | Requirement                                                                              | Priority |
|-------|------------------------------------------------------------------------------------------|----------|
| F03-1 | Displays the complete payment schedule with all remaining installments (dynamic count)  | Must      |
| F03-2 | Table shows: installment no., due date, principal, interest, fee, total, status         | Must      |
| F03-3 | Paid installments are visually marked (green/icon)                                      | Must      |
| F03-4 | Future installments are shown with their due date                                       | Must      |
| F03-5 | Data is fetched from the Mambu `/schedule` endpoint                                     | Must      |
| F03-6 | The table is responsive and can be scrolled horizontally on mobile                     | Must      |

### 2.4 Change due date (F04)

| ID    | Requirement                                                                             | Priority |
|-------|-----------------------------------------------------------------------------------------|----------|
| F04-1 | User can change the due date to a day between 1 and 28                                | Must      |
| F04-2 | The change is sent to Mambu via `POST /api/loans/{key}:changeDueDatesSettings`        | Must      |
| F04-3 | A confirmation dialog is shown before the change is applied                           | Must      |
| F04-4 | User sees the new due date confirmed after a successful change                        | Must      |
| F04-5 | Error message is shown if Mambu rejects the change                                    | Must      |
| F04-6 | The current due date is clearly shown in the dialog                                   | Must      |

### 2.5 Change loan term (F05)

| ID    | Requirement                                                                             | Priority |
|-------|-----------------------------------------------------------------------------------------|----------|
| F05-1 | User can change the number of remaining installments (min 6, max 360)                | Must      |
| F05-2 | New estimated installment amount is calculated and shown before confirmation          | Must      |
| F05-3 | The change is sent to Mambu via `POST /api/loans/{key}:changeLoanTerm`               | Must      |
| F05-4 | A confirmation dialog with a summary is shown                                         | Must      |
| F05-5 | The existing loan term is clearly shown                                               | Must      |

### 2.6 Payment holiday (F06)

| ID    | Requirement                                                                             | Priority |
|-------|-----------------------------------------------------------------------------------------|----------|
| F06-1 | User can apply for a payment holiday of 1–12 months                                  | Must      |
| F06-2 | During the payment holiday only interest is paid (PAY_INTEREST_ONLY)                 | Must      |
| F06-3 | Estimated interest cost during the payment holiday period is calculated and shown     | Must      |
| F06-4 | Extra cost (total additional interest over the loan's lifetime) is shown              | Must      |
| F06-5 | A confirmation dialog with a cost summary is shown                                    | Must      |
| F06-6 | The change is sent to Mambu via POST colon-action (endpoint not confirmed)           | Must      |
| F06-7 | A maximum of 12 months of payment holiday is allowed                                 | Must      |

---

## 3. Non-functional requirements

### 3.1 Security

| ID     | Requirement                                                                                    |
|--------|------------------------------------------------------------------------------------------------|
| NFR-S1 | API keys and credentials are stored only in `.env` files and are never exposed to the client |
| NFR-S2 | All Mambu API calls are made server-side (Next.js API Routes)                                |
| NFR-S3 | Vipps integration via OIDC with PKCE flow                                                    |
| NFR-S4 | HTTPS required in production                                                                  |
| NFR-S5 | No sensitive data is stored in `localStorage` or `sessionStorage`                            |
| NFR-S6 | JWT tokens are validated server-side on every API call                                       |
| NFR-S7 | CORS headers are set correctly; no direct browser-to-Mambu calls                            |

### 3.2 Performance

| ID     | Requirement                                                                         |
|--------|-------------------------------------------------------------------------------------|
| NFR-P1 | Page load under 2 seconds on 4G (Lighthouse score > 80)                           |
| NFR-P2 | API responses are cached where appropriate (Next.js fetch-caching)                |
| NFR-P3 | Skeleton loading is shown immediately when fetching data                           |

### 3.3 Accessibility and design

| ID     | Requirement                                                                              |
|--------|------------------------------------------------------------------------------------------|
| NFR-A1 | WCAG 2.1 AA level where possible                                                        |
| NFR-A2 | Responsive design – works on mobile (320px+), tablet, and desktop                      |
| NFR-A3 | Professional bank design: dark navy palette, clean cards, clear typography             |
| NFR-A4 | Norwegian as the primary language in the user interface                                 |
| NFR-A5 | All interactive elements have focus styles for keyboard navigation                     |

### 3.4 Configuration

All environment-dependent values are configured via `.env` files:

```env
# Mambu API
MAMBU_BASE_URL=https://knowit.sandbox.mambu.com/api
MAMBU_API_KEY=<api-key>
MAMBU_USERNAME=<username>
MAMBU_PASSWORD=<password>

# Vipps OIDC (kun i bruk når DEMO_MODE=false)
# Callback URL å registrere: http://localhost:3000/api/auth/callback/vipps
VIPPS_CLIENT_ID=<client-id>
VIPPS_CLIENT_SECRET=<client-secret>

# NextAuth
NEXTAUTH_SECRET=<random-secret>
NEXTAUTH_URL=http://localhost:3000

# Demo
DEMO_MODE=true
DEMO_LOAN_KEY=8a19a4029d1e987b019d1f2d626d6dc9
```

### 3.5 Compatibility

| ID     | Requirement                                                                         |
|--------|-------------------------------------------------------------------------------------|
| NFR-C1 | Supports Chrome, Firefox, Safari, Edge (last 2 major versions)                    |
| NFR-C2 | Mobile: iOS Safari 16+, Android Chrome 120+                                        |
| NFR-C3 | Node.js 20+ LTS                                                                     |

---

## 4. Data model – Mambu references

### Client (example from sandbox)

```json
{
  "encodedKey": "8a19b6a69d255395019d262ee2a572f4",
  "id": "885603059",
  "firstName": "Regan",
  "lastName": "Stracke",
  "assignedBranchKey": "8a19c1ab966cca9901966cf9dceb29b8"
}
```

### Loan (example from sandbox)

```json
{
  "encodedKey": "8a19dc979d254c1d019d2630046f7e3d",
  "id": "99004082",
  "loanName": "SPK Boliglån Annuitet",
  "loanAmount": 253000,
  "accountState": "APPROVED",
  "interestSettings": {
    "interestRate": 4.0,
    "interestCalculationMethod": "DECLINING_BALANCE_DISCOUNTED"
  },
  "scheduleSettings": {
    "repaymentInstallments": 24,
    "fixedDaysOfMonth": [15],
    "scheduleDueDatesMethod": "FIXED_DAYS_OF_MONTH"
  },
  "currency": { "currencyCode": "NOK" }
}
```

---

## 5. Scope limitations (Out of Scope for Demo)

- New loan application
- Document upload
- Message centre / chat
- Multi-loan view
- Account history / transactions
- KYC / AML
