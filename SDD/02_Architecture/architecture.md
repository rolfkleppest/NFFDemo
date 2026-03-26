# Architecture – MinSide Mortgage Self-Service

**Project:** NFF Demo – MinSide Mortgage Self-Service
**Technology:** Next.js 14 (App Router)
**Date:** 2026-03-25
**Status:** Draft v1.0

---

## 1. System Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         BROWSER (Client)                            │
│                                                                     │
│  ┌──────────────┐   ┌──────────────┐   ┌───────────────────────┐   │
│  │  Login Page  │   │  Dashboard   │   │  Self-Service Modals  │   │
│  │  (Vipps /    │   │  (Overview + │   │  (Due Date,           │   │
│  │   Demo)      │   │  Payment     │   │   Installment Count,  │   │
│  │              │   │  Schedule)   │   │   Payment Holiday)    │   │
│  └──────┬───────┘   └──────┬───────┘   └───────────┬───────────┘   │
│         │                  │                        │               │
└─────────┼──────────────────┼────────────────────────┼───────────────┘
          │  HTTPS           │  fetch()               │  fetch()
          ▼                  ▼                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   NEXT.JS SERVER (App Router)                       │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    API Routes (/api/*)                       │   │
│  │                                                             │   │
│  │  POST /api/auth/[...nextauth]  ← NextAuth.js               │   │
│  │  GET  /api/loan                ← Loan + Client + Schedule   │   │
│  │  POST /api/loan/forfallsdato   ← POST :changeDueDatesSettings  │   │
│  │  POST /api/loan/terminlengde   ← POST :changeLoanTerm        │   │
│  │  POST /api/loan/avdragsfrihet  ← POST :action (not confirmed)│   │
│  └────────────────────────────┬────────────────────────────────┘   │
│                               │                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                     lib/mambu/                               │  │
│  │  client.ts   – fetchMambu() wrapper (auth headers)           │  │
│  │  loans.ts    – getLoan(), updateLoan()                       │  │
│  │  clients.ts  – getClient()                                   │  │
│  │  schedule.ts – getSchedule()                                 │  │
│  └────────────────────────────┬────────────────────────────────┘   │
│                               │  Basic Auth / API Key              │
└───────────────────────────────┼─────────────────────────────────────┘
                                │  HTTPS
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│              MAMBU API (knowit.sandbox.mambu.com)                   │
│                                                                     │
│  GET  /api/clients/{key}                                            │
│  GET  /api/loans/{key}?detailsLevel=FULL                            │
│  GET  /api/loans/{key}/schedule?detailsLevel=FULL                   │
│  GET  /api/loanproducts/{key}?detailsLevel=FULL                     │
│  POST /api/loans/{key}:changeDueDatesSettings                       │
│  POST /api/loans/{key}:changeLoanTerm                               │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│              OIDC PROVIDER (Vipps – apitest.vipps.no)               │
│                                                                     │
│  Authorization Endpoint  (Vipps OIDC test environment)              │
│  Token Endpoint                                                     │
│  UserInfo Endpoint                                                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Next.js App Router – Folder Structure

```
nffdemo/
├── app/
│   ├── layout.tsx                    # Root layout (fonts, providers)
│   ├── page.tsx                      # Redirect → /login or /dashboard
│   ├── login/
│   │   └── page.tsx                  # F01 – Login (Vipps / Demo)
│   ├── dashboard/
│   │   ├── layout.tsx                # Dashboard layout (nav, sidebar)
│   │   ├── page.tsx                  # F02 – Loan overview
│   │   └── betalingsplan/
│   │       └── page.tsx              # F03 – Payment schedule (table)
│   └── api/
│       ├── auth/
│       │   └── [...nextauth]/
│       │       └── route.ts          # NextAuth handler (Vipps + Demo)
│       ├── loan/
│       │   ├── route.ts              # GET – fetches loan + client + schedule
│       │   ├── forfallsdato/
│       │   │   └── route.ts          # POST – change due date
│       │   ├── terminlengde/
│       │   │   └── route.ts          # POST – change installment count
│       │   └── avdragsfrihet/
│       │       └── route.ts          # POST – apply for payment holiday
│       └── health/
│           └── route.ts              # GET – health check
│
├── components/
│   ├── ui/                           # Generic UI components
│   │   ├── Button.tsx
│   │   ├── Card.tsx
│   │   ├── Modal.tsx
│   │   ├── Skeleton.tsx
│   │   ├── Badge.tsx
│   │   └── Alert.tsx
│   ├── auth/
│   │   ├── LoginCard.tsx             # Vipps / Demo login buttons
│   │   └── SessionGuard.tsx          # Wrapper that requires login
│   ├── loan/
│   │   ├── LoanSummaryCard.tsx       # F02 – Overview card
│   │   ├── LoanDetailsTable.tsx      # F02 – Details table
│   │   ├── PaymentScheduleTable.tsx  # F03 – Payment schedule
│   │   ├── NextPaymentCard.tsx       # F02 – Next installment payment
│   │   └── LoanStatusBadge.tsx       # Status badge (APPROVED, ACTIVE, etc.)
│   └── selfservice/
│       ├── ChangeDueDateModal.tsx    # F04 – Change due date
│       ├── ChangeInstallmentsModal.tsx # F05 – Change installment count
│       └── PaymentHolidayModal.tsx   # F06 – Payment holiday
│
├── lib/
│   ├── mambu/
│   │   ├── client.ts                 # fetchMambu() – base HTTP client
│   │   ├── loans.ts                  # getLoan(), patchLoan()
│   │   ├── clients.ts                # getClient()
│   │   ├── schedule.ts               # getSchedule()
│   │   └── types.ts                  # TypeScript types for Mambu responses
│   ├── auth/
│   │   ├── authOptions.ts            # NextAuth config (Vipps, Demo)
│   │   └── session.ts                # getServerSession wrapper
│   └── utils/
│       ├── formatters.ts             # NOK formatting, date formatting
│       └── calculations.ts           # Installment amount calculator, interest cost
│
├── hooks/
│   ├── useLoan.ts                    # SWR/React Query hook for loan data
│   └── useSelfService.ts             # Mutation hooks for self-service
│
├── types/
│   └── index.ts                      # Application-level TypeScript types
│
├── .env.local                        # Local environment variables (not in git)
├── .env.example                      # Template for environment variables
├── next.config.mjs          # NOTE: must be .mjs (not .ts) – Next.js 14.2.5 does not support next.config.ts
├── tailwind.config.ts
└── tsconfig.json
```

---

## 3. Mambu API Integration Pattern

### Principle: Server-side proxy

All Mambu API calls are made from the Next.js server side in order to:
1. Keep `MAMBU_API_KEY` and `MAMBU_PASSWORD` out of client-side code
2. Avoid CORS issues (Mambu does not allow direct browser calls)
3. Enable server-side caching via Next.js `fetch()` with `revalidate`

```
Browser  →  POST /api/loan/forfallsdato  →  Next.js API Route  →  POST /api/loans/{key}:action (Mambu)
```

### `lib/mambu/client.ts` – Base client

```typescript
const MAMBU_BASE_URL = process.env.MAMBU_BASE_URL!;
const MAMBU_API_KEY  = process.env.MAMBU_API_KEY!;
const MAMBU_USERNAME = process.env.MAMBU_USERNAME!;
const MAMBU_PASSWORD = process.env.MAMBU_PASSWORD!;

export async function fetchMambu<T>(
  path: string,
  options: RequestInit = {}
): Promise<T> {
  const credentials = Buffer.from(`${MAMBU_USERNAME}:${MAMBU_PASSWORD}`)
    .toString('base64');

  const response = await fetch(`${MAMBU_BASE_URL}${path}`, {
    ...options,
    headers: {
      'Authorization': `Basic ${credentials}`,
      'apikey': MAMBU_API_KEY,
      'Accept': 'application/vnd.mambu.v2+json',
      'Content-Type': 'application/json',
      ...options.headers,
    },
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({}));
    throw new MambuApiError(response.status, error);
  }

  return response.json() as Promise<T>;
}
```

### Example – `lib/mambu/loans.ts`

```typescript
export async function getLoan(loanKey: string): Promise<MambuLoan> {
  return fetchMambu<MambuLoan>(
    `/loans/${loanKey}?detailsLevel=FULL`
  );
}

export async function mambuPost<T = void>(
  path: string,
  body: unknown
): Promise<T> {
  return fetchMambu<T>(`/${path}`, {
    method: 'POST',
    body: JSON.stringify(body),
  });
}

// Example – change due date:
// await mambuPost(`loans/${loanKey}:changeDueDatesSettings`, { fixedDaysOfMonth: [20], ... })
// Example – change installment count:
// await mambuPost(`loans/${loanKey}:changeLoanTerm`, { isPreview: false, repaymentInstallments: 36 })
```

---

## 4. Vipps Integration (OpenID Connect)

### OIDC flow (Authorization Code + PKCE)

```
1. User clicks "Log in with Vipps"
2. NextAuth redirects to Vipps Authorization Endpoint (apitest.vipps.no)
   → response_type=code, scope=openid name email phoneNumber, code_challenge (PKCE)
3. Vipps displays login flow
4. Vipps redirects back to {NEXTAUTH_URL}/api/auth/callback/vipps with ?code=...
5. NextAuth exchanges code for tokens (Token Endpoint)
6. NextAuth fetches user info – including phone_number (E.164: +4746435489)
7. App normalises phone number (strips +47 prefix) → searches Mambu by mobilePhone
8. Finds first ACTIVE client → gets their most recently modified ACTIVE loan
9. loanKey stored in JWT cookie (HttpOnly)
10. User is redirected to /dashboard
```

### Demo mode (DEMO_MODE=true)

```typescript
// authOptions.ts
CredentialsProvider({
  id: 'demo',
  name: 'Demo Login',
  credentials: {},
  async authorize() {
    // Fetch real customer name from Mambu – do NOT hardcode.
    const loan = await getLoan(process.env.DEMO_LOAN_KEY!);
    const client = await getClient(loan.accountHolderKey);
    return {
      id: 'demo-user',
      name: `${client.firstName} ${client.lastName}`,
      email: client.emailAddress ?? 'demo@knowit.no',
      loanKey: process.env.DEMO_LOAN_KEY,
    };
  },
})
```

---

## 5. Configuration and Environment Variables

### Strategy

- `.env.local` – local development environment (not in git, see `.gitignore`)
- `.env.example` – template committed to the repo with empty values
- Vercel / Railway / Docker: set as environment variables
- `DEMO_MODE=true` enables mock login and uses hardcoded keys

### Overview of all variables

```env
# === Mambu API ===
MAMBU_BASE_URL=https://knowit.sandbox.mambu.com/api
MAMBU_API_KEY=
MAMBU_USERNAME=
MAMBU_PASSWORD=

# === Vipps OIDC (kun i bruk når DEMO_MODE=false) ===
# Callback URL å registrere i Vipps Developer Portal:
# http://localhost:3000/api/auth/callback/vipps
VIPPS_CLIENT_ID=
VIPPS_CLIENT_SECRET=

# === NextAuth ===
NEXTAUTH_SECRET=
NEXTAUTH_URL=http://localhost:3000

# === Demo ===
DEMO_MODE=true
DEMO_LOAN_KEY=8a19a4029d1e987b019d1f2d626d6dc9
```

---

## 6. Key Technical Decisions

### 6.1 Next.js App Router (not Pages Router)

**Chosen:** App Router (Next.js 14+)

**Rationale:**
- Server Components provide better performance (data fetched server-side without client JS)
- Nested layouts enable a dashboard shell with navigation
- API Routes (Route Handlers) run as Edge/Node.js server
- Better TypeScript support and modern React patterns

### 6.2 NextAuth.js for session management

**Chosen:** NextAuth.js v4 / Auth.js v5

**Rationale:**
- Ready-made OIDC integration with Vipps
- Secure cookie-based session management (HttpOnly, Secure, SameSite)
- Easy switching between providers (Vipps, Demo)

### 6.3 Tailwind CSS for styling

**Chosen:** Tailwind CSS + shadcn/ui

**Rationale:**
- Rapid prototyping with utility classes
- shadcn/ui provides professional, accessible components
- Easy customisation of colour palette (navy/blue banking theme)

### 6.4 SWR for client-side data fetching

**Chosen:** SWR (stale-while-revalidate)

**Rationale:**
- Automatic revalidation after self-service actions
- Built-in loading/error states
- Optimistic updates for better UX

### 6.5 Server-side Mambu proxy

**Decision:** All Mambu calls go through Next.js API Routes

**Rationale:**
- Mambu sandbox does not include CORS headers for browser calls
- API keys are kept server-side (security)
- Enables centralised validation and error handling

---

## 7. Error Handling Strategy

```typescript
// lib/mambu/client.ts
export class MambuApiError extends Error {
  constructor(
    public statusCode: number,
    public mambuError: unknown
  ) {
    super(`Mambu API error: ${statusCode}`);
  }
}

// app/api/loan/route.ts
// Loan and client are required for the page to render – schedule is secondary.
// If the schedule call fails, loan + client are returned without schedule,
// and the dashboard shows a warning that the payment schedule is unavailable.
export async function GET(request: Request) {
  const session = await getServerSession(authOptions);
  if (!session) return Response.json({ error: 'Unauthorized' }, { status: 401 });

  // Required: crash and return 502 if these fail
  let loan: MambuLoan;
  let client: MambuClient;
  try {
    [loan, client] = await Promise.all([
      getLoan(session.user.loanKey),
      getClient(session.user.clientKey),
    ]);
  } catch (error) {
    return Response.json(
      { error: 'Could not fetch loan data' },
      { status: 502 }
    );
  }

  // Optional: return schedule: null if the call fails
  let schedule = null;
  try {
    schedule = await getSchedule(session.user.loanKey);
  } catch {
    // schedule failure is non-blocking – UI displays a warning
  }

  return Response.json({ loan, client, schedule });
}
```

---

## 8. Sequence Diagram – Self-Service (Change Due Date)

```
User         Browser           Next.js API        Mambu API
  │               │                  │                 │
  │  Click modal  │                  │                 │
  │──────────────►│                  │                 │
  │               │  Show modal with │                 │
  │               │  current day     │                 │
  │  Select new   │                  │                 │
  │  day          │                  │                 │
  │──────────────►│                  │                 │
  │  Confirm      │                  │                 │
  │──────────────►│                  │                 │
  │               │  POST /api/loan/ │                 │
  │               │  forfallsdato    │                 │
  │               │  { day: 20 }     │                 │
  │               │─────────────────►│                 │
  │               │                  │  PATCH /loans/  │
  │               │                  │  {key}          │
  │               │                  │  scheduleSet..  │
  │               │                  │─────────────────►
  │               │                  │                 │
  │               │                  │  200 OK + loan  │
  │               │                  │◄─────────────────
  │               │  { success, loan}│                 │
  │               │◄─────────────────│                 │
  │  Confirmation │                  │                 │
  │◄──────────────│                  │                 │
```
