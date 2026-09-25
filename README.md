# 🔒 Deal-ID Layer — Freeze the Transaction, Not the Person

**Smarter fraud holds. Fairer investigations. Innocent accounts stay alive.**

Deal-ID Layer is a transaction-level freeze system for digital payments. Instead of locking an
entire bank account when one payment is disputed, every payment gets its own cryptographically
sealed identity: a **Deal ID**. A complaint freezes only the disputed amount, while salary, rent,
GST and supplier payments continue moving.

Built for **DCGC 2.0 Hack Sprint (GeeksforGeeks × Google Cloud) — Problem Statement 7:
Indiscriminate Collateral Account Freezes.**

## Overview

When stolen money passes through two or three accounts, every account it touched can be frozen in
full, including accounts belonging to people who did nothing but receive a legitimate payment.
Deal-ID Layer demonstrates a narrower alternative:

- **Payers** propose payments with a stated purpose and a pre-settlement risk score.
- **Payees** accept or bounce incoming money and explain unusual inflows before an accusation.
- **Reviewers** inspect a tamper-evident proof pack and release or return one held transaction.

The prototype implements the transaction-level hold, role isolation, password hashing, risk scoring,
versioned explanations, and SHA-256 proof verification in the browser.

## Key Features

### Landing Experience

- Unified sign-in and registration entry point
- Role fixed at registration and never selectable at sign-in
- Guided 12-step demo tour for the complete fraud scenario
- One-click **Reset Demo** to replay the scenario

### Payer Portal

- Propose payments with a stated purpose
- See the risk score and the signals that fired before settlement
- Track each Deal ID from proposed to accepted, held, or settled
- View only the payer's account, balance, and transaction history

### Payee Portal

- Accept or bounce incoming money
- See held and available balances side by side
- Detect inflow spikes from the account's own history
- Keep versioned explanations with timestamps and SHA-256 hashes
- Compare explanation versions and see whether they pre-date a complaint
- View the 72-hour dispute SLA countdown

### Reviewer Portal

- Open the full proof pack for a disputed Deal ID
- Inspect the sealed deal hash and sender risk signals
- Review the complete explanation history
- Release the hold or return funds to the victim as a human decision
- Require the officer code `CYBER-2026` for reviewer registration

## Authentication

Alongside the local email/password accounts described below, the sign-in screen offers
**Continue with Google**, backed by Firebase Authentication:

- Google only ever returns a name and email — never a role — so a first-time Google sign-in shows a
  one-time role picker (reviewer still requires the `CYBER-2026` officer code) before the workspace opens.
- The chosen role is saved in the browser's `localStorage`, keyed by the Google account's email, so
  returning to the same browser skips the picker. It does not sync across browsers or devices.
- Google sign-in and the local password accounts are independent systems that share the same
  workspaces; the scripted demo tour and its three fixed demo accounts are untouched.
- **Deployment note:** in the Firebase console, under Authentication → Settings → Authorized domains,
  add your Vercel domain (e.g. `your-app.vercel.app`) or `signInWithPopup` will be rejected on that origin.

## Demo Accounts

| Role | User ID | Password |
|---|---|---|
| Payer | `rohit@quickcash.in` | `Quick@4417` |
| Payee | `priya@nairaa.studio` | `Nairaa@9082` |
| Reviewer | `kverma@cybercell.gov.in` | `Cyber@1021` |

## Judging the Demo

Press **Run the 3-minute demo** on the sign-in screen. The guided tour signs in as each party and
advances through twelve steps. **Reset demo** in the top bar restores the opening state.

Look for:

- A ₹38,000 dispute holding ₹38,000 rather than the payee's ₹2,52,500 balance
- Explanation versions showing their real last-edited time and complaint status
- Cross-role workspace attempts being refused and written to the security log
- A simulated proof-pack change producing a SHA-256 mismatch during verification

## Technical Highlights

| Area | Implementation |
|---|---|
| Frontend | Vanilla JavaScript, HTML, and CSS in one file |
| Styling | Responsive design tokens with light and dark theme support |
| Cryptography | Browser Web Crypto API with SHA-256 |
| Statistics | Median plus `3 × 1.4826 × MAD` for inflow-spike detection |
| Hosting | Static deployment on Vercel |
| Authentication | Local salted SHA-256 password accounts, plus Google sign-in via Firebase Authentication |
| Dependencies | No build step, backend, or database. Firebase Auth is the only external service, loaded from the Firebase CDN |

The risk engine evaluates five weighted signals before settlement:

| Signal | Weight |
|---|---:|
| First-ever transfer from this sender | 35 |
| Amount above 3× the account's median deal | 25 |
| Three or more transfers from this sender in 24 hours | 20 |
| Sender account opened under 30 days ago | 20 |
| Round figure at or above ₹50,000 | 15 |

A score of 50 or more holds the payment before it reaches the receiver's usable balance.

## Security Model

- Passwords are never stored. Each account uses a random salt and stores the SHA-256 digest of
  `salt + password`.
- Five failed sign-in attempts lock an account for 30 seconds.
- A user's role is fixed at registration; it cannot be changed at sign-in.
- Cross-role access attempts are blocked and recorded in the visible security log.
- `vercel.json` configures `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, and
  `Permissions-Policy` headers.
- The single-file prototype uses inline styles and scripts, so it does not configure a strict CSP.

This is a prototype. State lives in browser memory, there is no real banking or NPCI integration,
and the four-hour settlement hold is compressed to 90 seconds for the demo. A production system
should use a slow password KDF such as bcrypt or Argon2 and an append-only server-side proof ledger.

## Deploy to Vercel

### Drag and drop

1. Go to [vercel.com/new](https://vercel.com/new).
2. Drag this folder onto the drop zone.
3. Choose framework preset **Other** and leave build and output directory fields empty.
4. Deploy.

### Vercel CLI

```bash
cd deal-id-layer
npx vercel          # preview deployment
npx vercel --prod   # production deployment
```

### GitHub

```bash
git init
git add .
git commit -m "Deal-ID Layer prototype"
git branch -M main
git remote add origin https://github.com/<you>/<repo>.git
git push -u origin main
```

Import the repository at [vercel.com/new](https://vercel.com/new). No environment variables are
needed.

## Local Preview

```bash
npx serve .
# or
python3 -m http.server 8000
```

Open the printed URL. Serving the folder over HTTP keeps local behavior close to production and
allows the browser's Web Crypto API to run consistently.

## Files

```text
deal-id-layer/
├── index.html     # Complete application: auth, portals, risk engine, and proof pack
├── vercel.json    # Static deployment configuration and security headers
└── README.md      # Project documentation
```

## License

This project is licensed under the **MIT License**.

**Deal-ID Layer — because one disputed payment should never cost someone their entire livelihood.**
