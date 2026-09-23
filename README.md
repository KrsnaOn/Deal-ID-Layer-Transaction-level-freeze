# Deal-ID Layer

A clickable prototype for **Problem 7: Indiscriminate Collateral Account Freezes**
(DCGC 2.0 Hack Sprint · GeeksforGeeks × Google Cloud).

When a fraudster's stolen money passes through an innocent person's bank account, banks freeze
the **entire** balance under Section 102 BNSS, 2023. This prototype freezes only the disputed
transaction, and asks merchants to explain unusual inflow on the day it happens — timestamped,
versioned, and dated before any complaint exists.

## Stack

One static `index.html`. Vanilla JS and CSS, **no build step, no dependencies, no backend**.
The only external request is Google Fonts. All state lives in memory, so every reload starts
from a clean demo.

## Deploy to Vercel

### Option 1 — drag and drop (fastest, no Git)

1. Go to [vercel.com/new](https://vercel.com/new)
2. Drag this **folder** onto the drop zone
3. Framework Preset: **Other** · Build Command: *(leave empty)* · Output Directory: *(leave empty)*
4. Deploy

### Option 2 — Vercel CLI

```bash
cd deal-id-layer
npx vercel          # preview deployment
npx vercel --prod   # production deployment
```

First run asks you to log in and link the project. Accept the defaults; when it asks to
override build settings, say no.

### Option 3 — GitHub

```bash
cd deal-id-layer
git init
git add .
git commit -m "Deal-ID Layer prototype"
git branch -M main
git remote add origin https://github.com/<you>/<repo>.git
git push -u origin main
```

Then import the repo at [vercel.com/new](https://vercel.com/new). Same settings as Option 1.
Every later push redeploys automatically.

**No environment variables are needed.** There is no backend, no database and no API key.

## Demo accounts

Tap a chip on the sign-in screen to fill these.

| Role | User ID | Password |
|---|---|---|
| Payer | `rohit@quickcash.in` | `Quick@4417` |
| Payee | `priya@nairaa.studio` | `Nairaa@9082` |
| Reviewer | `kverma@cybercell.gov.in` | `Cyber@1021` |

Registering a **Reviewer** additionally requires the officer authorisation code `CYBER-2026`,
because a self-registered reviewer could freeze anyone's funds.

## Judging the demo in 3 minutes

Press **Run the 3-minute demo** on the sign-in screen. Twelve steps, click Next to advance.
The demo signs each party in as it goes. **Reset demo** in the top bar restores the opening state.

## What to look at

- **Selective freeze** — a ₹38,000 dispute holds ₹38,000, not the ₹2,52,500 balance.
- **Spike explanations** — the record shows *when it was last edited*, keeps every version with
  its own hash, and badges whether a version pre-dates the complaint.
- **Role-based access** — each dashboard's *Workspace access* card tries to open the other two
  workspaces and is refused, with the attempt logged.
- **Tamper evidence** — in the reviewer's proof pack, *Simulate a tampered record* then
  *Verify hash* shows the SHA-256 mismatch.

## Notes on the security model

- Passwords are never stored. Each account keeps a random salt and the SHA-256 of
  `salt + password`, re-derived and compared at sign-in. A production system should use a slow
  KDF such as **bcrypt** or **Argon2** — SHA-256 is used here because it is what the browser
  exposes natively and keeps the prototype dependency-free.
- A user's role is fixed on their account at registration. It is never chosen at sign-in, because
  a role picked at sign-in is exactly the vulnerability that would let anyone become a reviewer.
- `vercel.json` sets `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy` and
  `Permissions-Policy`. There is deliberately **no Content-Security-Policy**: the page is a single
  file with inline `<style>` and `<script>`, so any CSP would need `'unsafe-inline'` for both and
  would provide little real protection. Splitting the assets out is the right fix if a CSP is
  wanted.
- `crypto.subtle` requires a secure context. Vercel serves HTTPS, so this works. Opening
  `index.html` straight from disk also works in Chrome; other browsers may fall back.

## Local preview

```bash
npx serve .
# or
python3 -m http.server 8000
```

Then open the printed URL. Use a server rather than double-clicking the file, so the page runs
in the same conditions as production.
