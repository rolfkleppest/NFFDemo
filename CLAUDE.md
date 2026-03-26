# CLAUDE.md – MinSide Boliglån Demo

## Project purpose

This is a **Spec Driven Design (SDD) demo** showing how fast a production-quality app can be built with Claude. The workflow is:

1. **Build SDD** (complete) → 2. **Build code** → 3. **Demo the app** → 4. **Delete code** (reset for next demo)

The app is a self-service mortgage portal ("MinSide Boliglån") for a Norwegian bank, built on top of the real Mambu core banking API (sandbox environment).

---

## Tech stack

| Layer | Choice |
|---|---|
| Framework | Next.js 14+ (App Router) |
| Auth | NextAuth.js – BankID OIDC (Signicat) primary, Vipps secondary |
| Backend | Mambu v2 REST API (proxied server-side) |
| Styling | Tailwind CSS |
| Language | TypeScript |

---

## Repository structure

```
NFFDemo/
├── CLAUDE.md              ← you are here
├── .env.local             ← credentials (not in git)
├── Src/                   ← Next.js app (to be built)
└── SDD/                   ← Spec Driven Design documents
    ├── 01_Requirements/requirements.md
    ├── 02_Architecture/architecture.md
    ├── 03_Features/
    │   ├── F01_Login.md
    │   ├── F02_LoanOverview.md
    │   ├── F03_PaymentSchedule.md
    │   ├── F04_ChangeDueDate.md
    │   ├── F05_ChangeInstallmentCount.md
    │   └── F06_PaymentHoliday.md
    ├── 04_DataModels/      ← Mambu API response examples (JSON) + Relations.md
    ├── 05_API/mambu-api-mapping.md
    ├── 06_UX/user-journey.md
    └── 07_TestSpecs/test-plan.md
```

---

## Key technical decisions

### Mambu API pattern
Mambu v2 uses a **colon-action POST pattern** for mutations – NOT REST PATCH:
```
POST /api/loans/{loanKey}:changeDueDatesSettings   ← F04
POST /api/loans/{loanKey}:changeLoanTerm           ← F05
POST /api/loans/{loanKey}:applyPaymentHoliday      ← F06 (unconfirmed endpoint)
```
All mutations return **204 No Content** on success.

### Mambu Accept header – varies per endpoint
**Always check `SDD/04_DataModels/` for the actual `Accept` header used per endpoint.**
The header is not always `application/vnd.mambu.v2+json` – Mambu can require different
media type variants for different endpoints. Use the captured Postman JSON files as the
authoritative source when implementing the `fetchMambu` client.

### Server-side proxy (no browser → Mambu calls)
All Mambu calls go through Next.js API routes to hide credentials and avoid CORS.
The internal helper is `mambuPost(path, body)` in `lib/mambu/client.ts`.

### Authentication
- `DEMO_MODE=true` → `CredentialsProvider` bypasses BankID/Vipps entirely
- BankID/Vipps buttons are **not rendered in the DOM** when `DEMO_MODE=true`
- JWT session: fixed 30-minute lifetime (`maxAge: 1800`), not sliding
- `clientKey` is NOT stored in session – fetched dynamically from `loan.accountHolderKey`

### Partial failure handling
- `getLoan()` and `getClient()` are **required** – crash and return 502 if they fail
- `getSchedule()` is **optional** – return `null` if it fails (non-blocking)

---

## Demo environment

```
MAMBU_BASE_URL=https://knowit.sandbox.mambu.com
MAMBU_API_KEY=I8zPeIO8RRQ7pSzGgiffXDOJs5PyvZOf
DEMO_MODE=true
DEMO_LOAN_KEY=8a19a4029d1e987b019d1f2d626d6dc9
```

Sandbox credentials: `apiuser` / `50XMAXwestern` (Basic Auth).
The demo loan belongs to customer **Regan Stracke** (customer no. 885603059).

> F06 (Payment Holiday) Mambu endpoint is **not confirmed** against the sandbox.
> F01–F05 are fully specced and ready to implement.

---

## Testing workflow

After generating code, run through the test specs in `SDD/07_TestSpecs/test-plan.md`:

1. Execute the relevant test cases for each feature implemented
2. If a test fails due to a **bug in the code** → fix the code
3. If a test fails due to an **ambiguity or error in the SDD** → update the relevant SDD file first, then fix the code
4. Never silently skip a failing test – either fix the root cause or document why the spec needs updating

---

## SDD status

All 6 features are specced and reviewed. The SDD passed two junior developer review rounds. Do not modify SDD files unless a test failure reveals an ambiguity, or unless explicitly asked.
