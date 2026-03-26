# F01 – Login (Vipps / Demo)

**Feature ID:** F01
**Priority:** Must
**Status:** Draft v1.1
**Date:** 2026-03-25

---

## 1. Description

The user logs in to MinSide via Vipps (primary). Login uses OpenID Connect (OIDC) and is handled by NextAuth.js. When `DEMO_MODE=true` is enabled, the user can log in without Vipps to demonstrate the app in a sandbox/demo context.

---

## 2. User story

> **As** a bank customer
> **I want** to securely log in to Min Side with Vipps
> **So that** I can view loan information and perform self-service actions

---

## 3. Acceptance criteria

### AC1 – Vipps login (primary)

**Given** that the user is on the `/login` page
**And** `DEMO_MODE=false`
**And** `VIPPS_CLIENT_ID` and `VIPPS_CLIENT_SECRET` are set in `.env.local`
**When** the user clicks "Log in with Vipps"
**Then** the user is redirected to the Vipps OIDC Authorization Endpoint (`apitest.vipps.no`)
**And** after successful Vipps authentication, the app looks up the user in Mambu by mobile number (see AC1a)
**And** the user is redirected to `/dashboard` with the correct loan data

> **Vipps sandbox-oppsett:** Credentials hentes fra [developer.vipps.no](https://developer.vipps.no/).
> Callback URL som må registreres i Vipps Developer Portal: `http://localhost:3000/api/auth/callback/vipps`

### AC1a – Vipps → Mambu mobilnummer-oppslag

**Given** that Vipps har autentisert brukeren
**And** Vipps-profilen inneholder `phone_number` (f.eks. `+4746435489`)
**When** NextAuth `profile()`-callbacken kjører
**Then** appen normaliserer nummeret (fjerner `+47` eller `47`-prefix) → `46435489`
**And** søker i Mambu: `GET /api/clients?mobilePhone=46435489`
**And** finner første ACTIVE klient
**And** henter nyeste aktive lån: `GET /api/loans?accountHolderKey={key}&accountState=ACTIVE&sortBy=lastModifiedDate:DESC&pageSize=1`
**And** setter `loanKey` = det funne lånets `id` i JWT
**And** setter kundens navn fra Mambu (ikke fra Vipps) for konsistens

**Fallback:** Hvis mobilnummeret ikke finnes i Mambu, brukes `DEMO_LOAN_KEY` fra `.env.local`

> **Merk:** I produksjon ville hver kunde ha ett aktivt boliglån. I sandbox-miljø kan en testklient ha mange lån — da returneres det sist-oppdaterte.

### AC3 – Demo login

**Given** that `DEMO_MODE=true`
**And** the user is on the `/login` page
**When** the user clicks "Demo login"
**Then** they are logged in with the loan and customer configured in `DEMO_LOAN_KEY` (`.env.local`)
**And** they are redirected to `/dashboard` without an OIDC flow
**And** the customer's real name (firstName + lastName) is fetched from Mambu during `authorize()` and stored in the JWT so the navbar can display it
**And** the name is NOT hardcoded in the source code – changing `DEMO_LOAN_KEY` automatically shows the new customer's name

### AC4 – Demo mode hides Vipps button

**Given** that `DEMO_MODE=true`
**When** the user opens `/login`
**Then** only the "Demo login" button is shown
**And** the Vipps button is not rendered in the DOM (hidden, not greyed out)

> **Also:** The Vipps button is NOT rendered if `VIPPS_CLIENT_ID` / `VIPPS_CLIENT_SECRET` are missing from `.env.local` (even when `DEMO_MODE=false`). The login page reads env vars server-side and passes `hasVipps` as a prop to `LoginForm`.

### AC5 – Session expires

**Given** that the user is logged in
**When** the JWT token expires (30-minute lifetime set in NextAuth config via `maxAge: 1800`)
**Then** the next API call will return 401 Unauthorized
**And** the user is redirected to `/login`
**And** a message is displayed: "Your session has expired. Please log in again."

> **Note:** NextAuth JWT tokens expire at a fixed time (`maxAge`), not on inactivity. "30 minutes" means maximum token lifetime, not a sliding session. The token is not automatically renewed on activity unless `updateAge` is set explicitly. For demo purposes a fixed 30 min is sufficient.

### AC6 – Logout

**Given** that the user is logged in
**When** the user clicks "Log out" in the navigation menu
**Then** the session cookie is deleted
**And** the user is redirected to `/login`
**And** a confirmation message is displayed: "You are now logged out."

### AC7 – Protected routes

**Given** that the user is NOT logged in
**When** the user attempts to navigate to `/dashboard` or another protected page
**Then** they are redirected to `/login`

---

## 4. UI design

### Login page – DEMO_MODE=false (production)

```
┌─────────────────────────────────────────┐
│  🏦  MinSide Boliglån                   │
│  ─────────────────────────────────────  │
│                                         │
│   Log in to view your loan details     │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │  [Vipps-logo]   Log in with     │   │
│  │                 Vipps           │   │
│  └─────────────────────────────────┘   │
│                                         │
│  Version 1.0 | NFF Demo 2026            │
└─────────────────────────────────────────┘
```

### Login page – DEMO_MODE=true (demo/sandbox)

BankID and Vipps buttons are **not rendered in the DOM** (AC4). No "or" separator.

```
┌─────────────────────────────────────────┐
│  🏦  MinSide Boliglån                   │
│  ─────────────────────────────────────  │
│                                         │
│   Log in to view your loan details     │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │  Log in (demo)                  │   │
│  └─────────────────────────────────┘   │
│                                         │
│  Version 1.0 | NFF Demo 2026            │
└─────────────────────────────────────────┘
```

**Colors and design:**
- Background: `#0f2044` (navy)
- Card background: `#1a3a6b` (dark blue)
- Accent color: `#4f8ef7` (light blue)
- Vipps button: Official Vipps orange (#ff5b24)
- Demo button: Neutral grey/teal

---

## 5. Technical implementation

### NextAuth configuration (`lib/auth/authOptions.ts`)

Providers are split by `DEMO_MODE`:
- `DEMO_MODE=true` → only `CredentialsProvider('demo')` is registered
- `DEMO_MODE=false` → Vipps OIDC provider is registered (if credentials are set)

```typescript
import type { NextAuthOptions } from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import { getLoan, getFirstActiveLoan } from '@/lib/mambu/loans';
import { getClient, getClientByPhone } from '@/lib/mambu/clients';

const isDemoMode = process.env.DEMO_MODE === 'true';
const demoLoanKey = process.env.DEMO_LOAN_KEY!;

async function resolveLoanKeyFromPhone(phone: string) {
  const client = await getClientByPhone(phone);
  if (!client) return { loanKey: demoLoanKey, mambuName: null };
  const loan = await getFirstActiveLoan(client.encodedKey);
  if (!loan) return { loanKey: demoLoanKey, mambuName: `${client.firstName} ${client.lastName}` };
  return { loanKey: loan.id, mambuName: `${client.firstName} ${client.lastName}` };
}

export const authOptions: NextAuthOptions = {
  session: { strategy: 'jwt', maxAge: 1800 },
  pages: { signIn: '/login', error: '/login' },
  providers: isDemoMode
    ? [
        CredentialsProvider({
          id: 'demo',
          name: 'Demo',
          credentials: {},
          async authorize() {
            // Fetch real customer name from Mambu – do NOT hardcode.
            const loan = await getLoan(demoLoanKey);
            const client = await getClient(loan.accountHolderKey);
            return {
              id: 'demo-user',
              name: `${client.firstName} ${client.lastName}`,
              email: client.emailAddress ?? 'demo@knowit.no',
              loanKey: demoLoanKey,
            };
          },
        }),
      ]
    : [
        // Vipps (OIDC – test environment)
        // Discovery: https://apitest.vipps.no/access-management-1.0/access/.well-known/openid-configuration
        // Callback URL to register at developer.vipps.no: {NEXTAUTH_URL}/api/auth/callback/vipps
        ...(process.env.VIPPS_CLIENT_ID && process.env.VIPPS_CLIENT_SECRET
          ? [{
              id: 'vipps',
              name: 'Vipps',
              type: 'oauth' as const,
              clientId: process.env.VIPPS_CLIENT_ID,
              clientSecret: process.env.VIPPS_CLIENT_SECRET,
              wellKnown: 'https://apitest.vipps.no/access-management-1.0/access/.well-known/openid-configuration',
              authorization: { params: { scope: 'openid name email phoneNumber' } },
              idToken: true,
              checks: ['pkce', 'state'] as ('pkce' | 'state')[],
              async profile(profile: Record<string, string>) {
                const phone = profile.phone_number ?? profile.phoneNumber ?? '';
                const vippsName = profile.name ?? `${profile.given_name ?? ''} ${profile.family_name ?? ''}`.trim();
                const { loanKey, mambuName } = phone
                  ? await resolveLoanKeyFromPhone(phone)
                  : { loanKey: demoLoanKey, mambuName: null };
                return {
                  id: profile.sub,
                  name: mambuName ?? vippsName,
                  email: profile.email ?? '',
                  loanKey,
                };
              },
            }]
          : []),
      ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.loanKey = ((user as unknown) as Record<string, unknown>).loanKey as string ?? demoLoanKey;
        token.name = user.name;
      }
      return token;
    },
    async session({ session, token }) {
      if (session.user) {
        session.user.loanKey = token.loanKey;
        session.user.name = token.name as string;
      }
      return session;
    },
  },
};
```

### Required env vars (`.env.local`)

```bash
# Demo mode (CredentialsProvider)
DEMO_MODE=true
DEMO_LOAN_KEY=<mambu loan id>

# Vipps (DEMO_MODE=false only)
# Callback URL: http://localhost:3000/api/auth/callback/vipps
VIPPS_CLIENT_ID=
VIPPS_CLIENT_SECRET=

NEXTAUTH_SECRET=<random secret>
NEXTAUTH_URL=http://localhost:3000
```

### SessionGuard component (`components/auth/SessionGuard.tsx`)

```typescript
// Used in dashboard/layout.tsx
import { redirect } from 'next/navigation';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth/authOptions';

export default async function SessionGuard({
  children,
}: {
  children: React.ReactNode;
}) {
  const session = await getServerSession(authOptions);
  if (!session) redirect('/login');
  return <>{children}</>;
}
```

---

## 6. Out of Scope

- BankID (not implemented)
- SMS OTP as an alternative to Vipps
- National identity number validation
- Multi-factor authentication beyond Vipps
- Remembered login / "Remember me" functionality
- Account creation / registration
