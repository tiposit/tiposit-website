# TiposIt MVP — Critical Decision Logbook
**Project:** TiposIt Earned Wage Access (EWA) SaaS Platform  
**Version:** 1.0 MVP  
**Date:** April 2026  
**Author:** Drew Newton / TiposIt  

---

## Overview

TiposIt is a B2B2C fintech platform delivering Earned Wage Access, digital wallets, ICE Credit Cards (White/Blue/Black tiers), BNPL, and the Tipflic analytics subsidiary. This logbook documents every major architectural decision, regulatory workaround, and future upgrade path made during v1 MVP development.

---

## File Architecture

| File | Purpose | URL Route |
|------|---------|-----------|
| `index.html` | Marketing site / landing page | `/` |
| `app.html` | Employee client portal (mobile-first) | `/app` |
| `employer.html` | Employer/partner portal (desktop-first) | `/employer` |
| `dashboard.html` | Internal admin CRM | `/dashboard` |
| `vercel.json` | Vercel routing config | — |

**Deployment:** Static HTML/CSS/JS via Vercel (GitHub Desktop → push to `main` → auto-deploy ~30s)

---

## Decision Log

---

### DECISION 001 — No Backend / localStorage-First Architecture
**Status:** MVP Workaround  
**Decision:** All user data (profiles, clock records, transactions, employer accounts) stored in browser `localStorage`. Passwords stored via a custom 32-bit `simpleHash()` function. Sessions managed via `sessionStorage`.  
**Why:** Deploying a real backend (Node.js, PostgreSQL, auth tokens) requires server infrastructure, secrets management, and FDIC-compliant data handling — none of which can be delivered in an HTML-only MVP without a hosting backend.  
**Tradeoff:** Data is browser-local only — no cross-device sync, no real server-side security.  
**Upgrade Path (Q3 2026):** Replace localStorage with a Supabase backend (PostgreSQL + Row Level Security + Auth). All `localStorage.setItem(...)` calls map 1:1 to Supabase table inserts. `simpleHash()` replaces with bcrypt/Argon2 on the server side.

---

### DECISION 002 — Password Storage via simpleHash()
**Status:** Demo Only — NOT Production Secure  
**Decision:** Implemented a custom 32-bit integer hash (djb2-style) that converts passwords to a base-36 string stored in localStorage.  
**Why:** Actual bcrypt/Argon2 hashing requires a server-side runtime. Browser-native `SubtleCrypto` would work but adds async complexity for a proof-of-concept.  
**Tradeoff:** The hash is NOT salted and NOT collision-resistant at production scale.  
**Warning:** Any real user data must never use this function. Clearly labeled as demo-only in code.  
**Upgrade Path:** Server-side bcrypt (Node.js `bcryptjs`) with per-user salts on account creation.

---

### DECISION 003 — ICE Credit Cards as Placeholder (Regulatory Hold)
**Status:** Banking Regulation Workaround  
**Decision:** ICE White/Blue/Black credit cards are displayed as fully designed UI components with pre-apply flow, but no real card issuance occurs. Clicking "Pre-Apply" records intent to localStorage and shows a toast.  
**Why:** Issuing credit cards requires a bank partnership, BIN sponsorship, card network membership (Visa/MC), and full KYC/AML compliance. None of these are available at MVP launch.  
**Timeline:** Targeting Q3 2026 for real card issuance via a BaaS provider (Marqeta, Lithic, or Unit).  
**Upgrade Path:** Replace `preApplyCard()` function with an API call to the BaaS card-issuance endpoint. Card numbers, CVVs, and expiry dates are returned from the BaaS provider and stored encrypted server-side. Never stored in localStorage.

---

### DECISION 004 — EWA Withdrawals Are Simulated
**Status:** Banking Regulation Workaround  
**Decision:** The "Request Pay" flow in the employee app calculates the correct net payout (accrued wages minus 1.5% fee), shows a success UI, and logs the transaction — but no real ACH transfer occurs.  
**Why:** Real EWA disbursements require: (a) verified payroll data connection, (b) ACH origination rights through a banking partner, (c) employer authorization, and (d) CFPB compliance under the Earned Wage Access rule.  
**Upgrade Path:** Integrate Plaid for payroll/bank account linking (Plaid Income + Plaid Auth). Use a BaaS provider (Unit, Treasury Prime, or Column Bank) for ACH push disbursements. All balance calculations remain the same — only the "deliver funds" step changes.

---

### DECISION 005 — BNPL as Coming-Soon Placeholder
**Status:** Regulatory Workaround  
**Decision:** Buy Now Pay Later is shown in both employee app and employer portal as a "Coming Q3 2026" placeholder. A `preRegisterBNPL()` function captures intent.  
**Why:** BNPL products require a lending license or bank partnership for the credit facility, TILA disclosure compliance, and underwriting logic.  
**Upgrade Path:** Partner with a BNPL-as-a-service provider (Wisetack, Splitit, or Affirm for Business) or obtain a lender license in Texas as a first state.

---

### DECISION 006 — Camera / Selfie Capture via getUserMedia
**Status:** Functional (with graceful fallback)  
**Decision:** Both employee signup (Step 2) and clock-in verification use `navigator.mediaDevices.getUserMedia()` to capture a selfie via the browser camera. If denied/unavailable, a "Skip" option is shown and the workflow continues without a photo.  
**Why:** Native browser camera API works without any native app or mobile SDK. Photos are stored as base64 data URIs in localStorage (tied to the user profile or shift record).  
**Limitation:** Base64 photos in localStorage can balloon storage limits (5MB typical cap). Large shift histories with photos may hit the limit.  
**Upgrade Path:** Upload selfies to Cloudflare R2 or AWS S3 using a presigned URL flow. Store only the URL in the user record.

---

### DECISION 007 — Chart.js Must Load in `<head>` (Not End of Body)
**Status:** Critical Bug Fix  
**Decision:** Chart.js CDN script tag is placed in `<head>` with no `defer` attribute across all files that use charts (dashboard.html, employer.html, app.html).  
**Why:** When the Chart.js script was placed at the bottom of `<body>`, the inline `<script>` blocks that called `new Chart(...)` executed first — before Chart.js was defined — causing "Chart is not defined" uncaught errors that broke the entire dashboard.  
**Rule:** Any library that is referenced in inline `<script>` blocks must be loaded in `<head>` synchronously. External scripts called inside functions (not top-level) can use `defer`.

---

### DECISION 008 — No ES6 Spread Operators in Chart Config
**Status:** Compatibility Fix  
**Decision:** All Chart.js configuration objects are written inline with explicit properties. No `{...SHARED_OPTS, ...overrides}` object spread syntax.  
**Why:** Some environments and older V8 builds reject spread syntax inside certain script contexts. Switching to plain object literals with explicit properties resolved all chart rendering failures.  
**Rule:** Use `var` and `function` declarations throughout. Arrow functions and spread operators are avoided in all MVP JS.

---

### DECISION 009 — Employer Code Generation
**Status:** MVP Implementation  
**Decision:** Employer codes are generated as: first 6 alphanumeric characters of the company name (uppercased, non-alphanumeric stripped) + 3-digit random suffix (100–999). Example: "Atlas Corp" → `ATLASCORP` → `ATLASC` + `347` = `ATLASC347`.  
**Why:** Codes need to be short enough to type on mobile, unique enough to avoid collisions across a few hundred early partners, and human-readable enough to distribute via onboarding email.  
**Upgrade Path:** Server-side code generation with collision detection against a database index. Add optional vanity code feature for enterprise partners.

---

### DECISION 010 — Demo Employer Codes Hard-Coded in app.html
**Status:** MVP Demo Workaround  
**Decision:** The `FAKE_EMPLOYERS` object in app.html hard-codes 6 test employer codes: `ATLAS001`, `SUNRISE01`, `VERDE001`, `LONESTAR1`, `GULF0001`, `DEMO0001`.  
**Why:** Without a real backend, the employee app cannot look up employer codes dynamically. The hard-coded list allows functional demos and user testing of the entire employer-linking flow.  
**Upgrade Path:** Replace the lookup with a `fetch('/api/employer/lookup?code=...')` call to the Supabase backend. Returns employer name, logo URL, and allowed hourly rate range.

---

### DECISION 011 — 2FA is Simulated (No Real SMS/Email)
**Status:** Demo Workaround  
**Decision:** In all portals, 2FA generates a random 6-digit code, logs it to the browser console, and auto-fills it after 3 seconds via `setTimeout`. No real SMS or email is sent.  
**Why:** Twilio/SendGrid integration requires API keys, a server-side proxy (cannot expose keys in frontend JS), and phone number verification.  
**Upgrade Path:** Route 2FA through a Supabase Edge Function that calls Twilio Verify API. The browser makes a `POST /api/2fa/send` call; the server sends the SMS. The OTP is verified server-side — never touches the browser.

---

### DECISION 012 — Revenue Share Displayed as Static UI
**Status:** MVP Placeholder  
**Decision:** The Revenue Share dashboard in the employer portal shows illustrative numbers and a calculation model but does not pull real transaction data.  
**Why:** Real revenue share requires: (a) real transaction processing, (b) employer attribution of every employee transaction, and (c) an automated disbursement ledger.  
**Upgrade Path:** Each EWA transaction records `employer_id` in the database. A monthly cron job calculates `total_platform_fees_attributed × revenue_share_rate` and triggers an ACH credit to the employer's linked bank account.

---

### DECISION 013 — Tipflic Analytics as Placeholder
**Status:** Future Product  
**Decision:** Tipflic (TiposIt's analytics subsidiary) is represented in both employee app and employer portal as "Coming Q3 2026" with a brief product description.  
**Why:** Tipflic requires a full analytics data pipeline (event tracking → warehouse → dashboards) and a separate UI. Building this in MVP scope is not feasible.  
**Upgrade Path:** Instrument all app events with a Segment analytics client. Route to a ClickHouse or BigQuery warehouse. Build Tipflic as a separate React dashboard app.

---

### DECISION 014 — Single-File SPA Pattern
**Status:** MVP Architecture  
**Decision:** Each portal is a single HTML file. All CSS is in `<style>` tags, all JS in `<script>` tags. Navigation between screens is pure CSS show/hide (`.active` class toggle).  
**Why:** Zero build toolchain required. No Webpack, no npm, no Node. Can be deployed by dragging files to GitHub Desktop and pushing to main.  
**Tradeoff:** Files are large (100KB–150KB). No code splitting, no lazy loading.  
**Upgrade Path:** Migrate to a Vite + React SPA with proper routing (React Router), component splitting, and lazy imports. The localStorage data layer and business logic functions map directly to React hooks and Zustand store slices.

---

### DECISION 015 — Vercel cleanUrls + Rewrites for Clean Routing
**Status:** Deployed  
**Decision:** `vercel.json` uses `cleanUrls: true` (strips `.html` from public URLs) plus explicit rewrite rules for each portal.  
**Config:**
```json
{
  "cleanUrls": true,
  "trailingSlash": false,
  "rewrites": [
    { "source": "/dashboard", "destination": "/dashboard.html" },
    { "source": "/app",       "destination": "/app.html" },
    { "source": "/employer",  "destination": "/employer.html" }
  ]
}
```
**Why:** Users and marketing links should use clean URLs (`tiposit.com/app`, not `tiposit.com/app.html`). Vercel's CDN rewrites the path server-side so the static file is served correctly.

---

## Regulatory & Compliance Checklist

| Item | Status | Notes |
|------|--------|-------|
| FDIC-compliant banking partner | ⏳ Pending | Required before real deposits |
| KYC / AML identity verification | ⏳ Pending | Required before wallet activation |
| ACH origination rights | ⏳ Pending | Required for EWA disbursements |
| BIN sponsorship (ICE Cards) | ⏳ Pending | Required for credit card issuance |
| Texas Lender License | ⏳ Pending | Required for BNPL |
| CFPB EWA Compliance | ⏳ Pending | Monitor 2024 CFPB rulemaking |
| Plaid integration | ⏳ Pending | Q3 2026 target |
| Privacy Policy / Terms of Service | ⏳ Pending | Legal review required |
| PCI DSS Compliance | ⏳ Pending | Required before any card data handled |

---

## MVP Feature Status

| Feature | App | Employer | Status |
|---------|-----|----------|--------|
| Account signup + login | ✅ | ✅ | Functional (localStorage) |
| 2FA verification | ✅ | ✅ | Simulated (no real SMS) |
| Camera selfie capture | ✅ | — | Functional (getUserMedia) |
| Clock in / Clock out | ✅ | — | Functional with live wage accrual |
| Digital wallet | ✅ | — | UI functional, no real funds |
| Same-day pay / EWA | ✅ | — | Simulated disbursement |
| ICE Credit Cards | ✅ | — | Pre-apply UI, no real issuance |
| BNPL | ✅ | ✅ | Placeholder — Q3 2026 |
| Company profile + logo | — | ✅ | Functional (FileReader → base64) |
| Employee management | — | ✅ | Functional (localStorage) |
| Revenue share dashboard | — | ✅ | Illustrative / static |
| Analytics charts | ✅ | ✅ | Functional (Chart.js) |
| Tipflic analytics | ✅ | ✅ | Placeholder — Q3 2026 |
| Admin CRM dashboard | — | — | ✅ `/dashboard` route |
| Tutorial walkthrough | ✅ | — | 6-step interactive tutorial |

---

## Known Limitations (v1)

1. **No cross-device sync** — localStorage is browser-local. Logging in on a different device or browser shows no data.
2. **No real money movement** — All financial operations are simulated.
3. **No email verification** — Any email address can be used to sign up.
4. **Camera requires HTTPS** — getUserMedia won't work on non-HTTPS origins. Vercel provides HTTPS automatically.
5. **localStorage ~5MB limit** — Storing base64 photos in localStorage can hit browser storage limits with many shift records.
6. **No server-side validation** — All form validation is client-side JS only.

---

## Upgrade Roadmap

| Quarter | Milestone |
|---------|-----------|
| Q2 2026 | Supabase backend — real auth, real data persistence |
| Q2 2026 | Twilio Verify — real SMS/email 2FA |
| Q3 2026 | Plaid integration — payroll + bank account linking |
| Q3 2026 | BaaS partnership — ACH disbursements (Unit or Column) |
| Q3 2026 | ICE card issuance via Marqeta or Lithic |
| Q3 2026 | BNPL launch via partner or lender license |
| Q4 2026 | Tipflic analytics platform v1 |
| Q4 2026 | React/Vite migration — proper SPA with code splitting |
| Q1 2027 | Mobile app (React Native) |

---

*Log maintained by TiposIt engineering team. Last updated: April 2026.*
