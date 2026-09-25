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

The app has two independent, parallel sign-in systems that share the same three workspaces:

1. **Local password accounts** — the original system. Register with an email, password, and a role
   fixed at registration. State (users, sessions) lives in browser memory only; see
   [Security Model](#security-model).
2. **Google sign-in** — backed by Firebase Authentication, added alongside the local system without
   changing it. This is what the rest of this section documents.

### How Google sign-in works in this app

- The sign-in screen's **Continue with Google** button calls `signInWithPopup` with a
  `GoogleAuthProvider`, wired up in a small `<script type="module">` block near the top of
  `index.html` that exposes `window.DealIDFirebase.signInWithGoogle()` /
  `.signOut()` to the rest of the (non-module) app script.
- Google only ever returns a name and email — never a role — so a **first-time** Google sign-in
  shows a one-time role picker (payer / payee / reviewer; reviewer still requires the
  `CYBER-2026` officer code) before the workspace opens.
- The chosen role is saved in the browser's `localStorage` under the key
  `dealid:googleRole:<email>`, so returning to the same browser on a later visit skips the picker.
  This does **not** sync across browsers or devices — it's a client-only mapping, matching this
  prototype's "no backend, no database" design (see [Technical Highlights](#technical-highlights)).
- Once resolved, a Google-authenticated user is treated exactly like a password account internally
  (`establishSession`, workspace guards, the security log) — the only difference recorded is
  `S.auth.how === 'google'`, which is used solely so **Sign out** also calls Firebase's `signOut()`.
- The scripted **"Run the 3-minute demo"** tour and its three fixed demo accounts are completely
  untouched by any of this.

### Setting up your own Firebase project

1. Go to the [Firebase console](https://console.firebase.google.com) → **Add project** (or reuse
   an existing one).
2. **Project settings → General → Add app → Web** (the `</>` icon). Register an app; you don't need
   Firebase Hosting for this step. Copy the resulting config object
   (`apiKey`, `authDomain`, `projectId`, `storageBucket`, `messagingSenderId`, `appId`).
3. Paste that object into the `firebaseConfig` constant inside the `<script type="module">` block
   near the top of `index.html`. This Firebase **web API key is not a secret** — Firebase's real
   access control is enforced by the settings in steps 4–5, not by hiding this key — so it's safe
   to commit and ship in a static file.
4. **Authentication → Sign-in method → Add new provider → Google → Enable.** Skipping this step is
   the most common cause of a sign-in failure (surfaces in this app as
   `Google sign-in failed: auth/operation-not-allowed`).
5. **Authentication → Settings → Authorized domains.** `localhost` is authorized by default, which
   covers local preview (see [Local Preview](#local-preview)). For any real deployment, add that
   exact domain here too (e.g. `your-app.vercel.app`) — otherwise `signInWithPopup` fails with
   `auth/unauthorized-domain`.

### Firebase CLI (optional, for local development)

The CLI isn't required to run or deploy this static site — it's useful if you want to test
Google/email sign-in against the Firebase Auth **emulator** instead of the real `dead-id` project
while developing:

```bash
npm install -g firebase-tools
firebase login              # opens a browser for Google OAuth
                             # if it can't open a browser (e.g. over SSH / WSL), use:
                             # firebase login --no-localhost
firebase init emulators      # select Authentication Emulator
firebase emulators:start
```

### Troubleshooting

The app now shows the real Firebase error code instead of a generic message (e.g.
`Google sign-in failed: auth/popup-blocked`). Common ones:

| Error code | Cause | Fix |
|---|---|---|
| `auth/operation-not-allowed` | Google isn't enabled as a sign-in provider | Step 4 above |
| `auth/unauthorized-domain` | The current origin isn't in the authorized domains list | Step 5 above |
| `auth/popup-blocked` | The browser blocked the sign-in popup | Allow popups for the site and retry |
| `auth/popup-closed-by-user` | The account chooser was closed before completing sign-in | Retry and finish the popup flow |
| `Google sign-in is still loading...` | The page's `<script type="module">` hasn't finished loading yet (rare; happens on a very slow connection right after page load) | Wait a second and retry |

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
