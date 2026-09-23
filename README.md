# 🔒 Deal-ID Layer — Freeze the Transaction, Not the Person

**Smarter Fraud Holds. Fairer Investigations. Innocent Accounts Stay Alive.**

Deal-ID Layer is a transaction-level freeze system for digital payments. Instead of locking an entire bank account when one payment is disputed, it gives every payment its own cryptographically sealed identity — a **Deal ID** — so a complaint freezes only the disputed amount. Salary, rent, GST and supplier payments keep moving while the investigation runs.

Built for **DCGC 2.0 Hack Sprint (GeeksforGeeks × Google Cloud) — Problem Statement 7: Indiscriminate Collateral Account Freezes.**

**🌐 Live demo:** https://deal-id-layer.vercel.app

---

## Overview

When a fraud victim files a complaint today, the stolen money has usually passed through two or three accounts. Every account it touched gets frozen in full — including accounts belonging to people who did nothing but receive a legitimate payment from the wrong person.

Deal-ID Layer fixes this by giving each stakeholder a dedicated, access-controlled portal:

- **Payers** propose payments that carry a stated purpose and a pre-settlement risk score.
- **Payees** accept or bounce incoming money, and explain inflow spikes *before* anyone accuses them.
- **Reviewers** open a tamper-evident proof pack and release or return a single held transaction.

The Rajasthan High Court said it plainly in *Shree Balaji Enterprises v. RBI* (Aug 2026): *"first course should be to place a lien or hold upon the amount rather than debit-freeze the entire account."* This project is the missing technical mechanism that makes that possible.

---

## Key Features

### Landing Experience
- Unified entry point with **Sign In** and **Sign Up**
- Role is bound to the account at registration — never selectable at sign-in
- Guided 12-step demo tour that walks a judge through the full fraud scenario
- One-click **Reset Demo** to replay the scenario from a clean state

### Payer Portal
- Propose payments with a stated purpose (money never moves without receiver acceptance)
- Live risk score computed before settlement, with the exact signals that fired
- Track every Deal ID from proposed → accepted → held → settled
- See own account status, balance and transaction history only

### Payee Portal
- **Accept or bounce** — refuse suspicious money before it ever lands
- **Held vs Available** balance shown side by side — the core proof that only one transaction is frozen
- **Inflow spike detection** — statistical ceiling computed from the account's own history
- **Versioned spike explanations** — every revision keeps its own last-edited timestamp and SHA-256 hash
- Word-level diff between any two explanation versions
- **Before / After complaint** badges on every version
- 72-hour dispute SLA with a visible countdown

### Reviewer Portal (Cyber Cell)
- Open the full **proof pack** for a disputed Deal ID
- Sealed SHA-256 deal hash, sender risk signals, and the complete explanation history in one view
- AI dispute analysis powered by Google Gemini (advisory only)
- Release the hold or return funds to the victim — always a human decision
- Officer code required to register as a Reviewer

### Trust & Security
- Passwords never stored — per-account random salt + `SHA-256(salt + password)`
- 5 failed attempts lock the account for 30 seconds
- Strict role isolation — cross-role access attempts are blocked and written to a visible security log
- Newly registered users get a genuinely empty account, never a view of demo data
- API keys live only in serverless environment variables, never in shipped code

---

## 🤖 AI Dispute Analyzer

Deal-ID Layer includes a **Gemini-powered dispute analysis engine** that helps a reviewer weigh the evidence in a frozen transaction — without ever making the decision itself.

**How it works:**

1. **Proof pack aggregation** — Collects the sealed deal record (sender KYC, receiver KYC, amount, purpose, timestamps), the receiver's held-vs-available fund position, and the full version history of every spike explanation with its timestamps.
2. **Temporal evidence weighting** — The prompt explicitly instructs the model that an explanation written *before* a complaint existed carries real evidential weight, because it cannot have been authored to answer an accusation that had not yet been made.
3. **Bias correction** — The prompt states that a high automated risk score describes the **sender**, not the receiver. Without this instruction, a general-purpose model reproduces exactly the bias this project exists to correct.
4. **Structured output** — The response is constrained with `responseSchema` and `responseMimeType: 'application/json'`, so the model must return a machine-readable verdict rather than prose:
   - `verdict` — `likely_genuine` · `needs_scrutiny` · `likely_complicit`
   - `confidence` — integer percentage
   - `headline`, `supporting[]`, `concerns[]`, `recommendation`
5. **Natural-language summary** — Rendered on the reviewer dashboard, for example:

   > *"Explanation v1 was filed 11 days before the complaint and matches the declared purpose. Sender risk signals do not implicate the receiver. Recommend releasing the hold."*

6. **Advisory only — the reviewer decides.** Given that the underlying problem is automated systems freezing accounts without human review, an AI that *makes* the call would reproduce the harm. Every result is labelled advisory, and no verdict can release or return funds on its own.

The engine is **model-agnostic and fail-safe**: it tries a pinned model, then `gemini-2.5-flash`, `gemini-flash-latest` and `gemini-2.0-flash`, treating HTTP 404 as "try the next one". Requests carry a 20-second timeout and always return HTTP 200 with a reason on failure, so a model deprecation or a bad network never breaks a live demo.

---

## Tech Stack

| Layer | Technology |
| --- | --- |
| Frontend | Vanilla JavaScript + HTML + CSS (no framework, no build step) |
| Styling | Inline design-token CSS with light/dark theme support |
| Backend / Server | Node.js serverless functions (Vercel Functions) |
| AI / ML | Google Gemini via the Google AI API (`generativelanguage.googleapis.com`) |
| Cryptography | Web Crypto API — native `SubtleCrypto.digest('SHA-256')` |
| Statistics | Median + 3 × 1.4826 × MAD (median absolute deviation) |
| Secret Storage | Vercel Environment Variables |
| CI / CD | Vercel ↔ GitHub auto-deploy on push to `main` |
| Hosting / Deployment | Vercel (production) |
| Testing | Playwright (headless Chromium) — 21 auth + 7 integration assertions |

---

## Folder Structure

```
deal-id-layer/
├── api/
│   ├── analyze.js              # Gemini dispute-analysis endpoint
│   └── status.js               # Feature-availability probe
├── .env.example                # Sample environment variables
├── .gitignore                  # Excludes every .env* file
├── index.html                  # The entire application (112 KB, self-contained)
├── vercel.json                 # Security headers + clean URLs
└── README.md                   # This file
```

> **Why one HTML file?** The whole application — auth, risk engine, hashing, charts, all three portals — lives in `index.html` with zero external requests. It runs offline, from a USB stick, and on conference Wi-Fi that drops. For a live demo, that resilience is a feature, not a shortcut.

---

## Getting Started (Run Locally)

### Prerequisites
- Node.js (for tooling)
- A Google AI Studio API key — **optional**, the app runs fully without one

### Steps

```bash
# 1. Clone the repository
git clone https://github.com/KrsnaOn/Deal-ID-Layer-Transaction-level-freeze.git
cd Deal-ID-Layer-Transaction-level-freeze

# 2. Set up environment variables (optional — only for AI analysis)
cp .env.example .env.local
# Add your GEMINI_API_KEY inside .env.local

# 3. Run the development server
npx vercel dev
```

Open the URL shown in your terminal (typically http://localhost:3000).

> **Important:** serve over `http://localhost` or HTTPS, not `file://`. The Web Crypto API requires a secure context, and SHA-256 hashing will silently fall back without one.

### Demo Credentials

| Role | Email | Password |
| --- | --- | --- |
| Payer | `rohit@quickcash.in` | `Quick@4417` |
| Payee | `priya@nairaa.studio` | `Nairaa@9082` |
| Reviewer | `kverma@cybercell.gov.in` | `Cyber@1021` |

Registering a new Reviewer account requires the officer code `CYBER-2026`.

---

## Environment Variables

| Variable | Required | Effect |
| --- | --- | --- |
| `GEMINI_API_KEY` | No | Enables live Gemini analysis. Absent, the app runs in local mode |
| `GEMINI_MODEL` | No | Pins a specific model. Blank, the function auto-detects a working one |

**Graceful degradation:** `/api/status` reports whether a key is configured, and the UI shows either **Gemini · live** or **Gemini · local**. With no key set, every other feature behaves identically — the freeze mechanism, the hashing and the version history are all local computation. The demo cannot be broken by a missing key.

Verify a key works before deploying:

```bash
curl -s "https://generativelanguage.googleapis.com/v1beta/models?key=$GEMINI_API_KEY" | head -40
```

⚠️ **Never commit the key.** Public repositories are scraped continuously and Google automatically revokes exposed keys.

---

## Deployment (Vercel + Google Cloud)

### 1. Push to GitHub

```bash
git remote add origin https://github.com/<your-username>/<repo>.git
git branch -M main
git push -u origin main
```

### 2. Import into Vercel
- Go to [vercel.com/new](https://vercel.com/new)
- Import the repository — no build settings needed (framework: **Other**)
- Deploy

### 3. Add the Gemini key
- Get a key at [aistudio.google.com/apikey](https://aistudio.google.com/apikey)
- In Vercel: **Settings → Environment Variables → Add `GEMINI_API_KEY`** (Production)
- **Redeploy** — Vercel only binds environment variables on a new deployment

### 4. Turn off deployment protection
Vercel enables SSO protection by default, which blocks visitors without a Vercel account. For a public demo:
- **Settings → Deployment Protection → Vercel Authentication → Disabled**

### 5. Google Cloud Console (optional, for scaling)
- Open [console.cloud.google.com](https://console.cloud.google.com)
- Select the project backing your API key
- Enable the **Generative Language API**
- Manage quotas, billing and monitoring here if moving beyond the free tier

Every subsequent push to `main` redeploys automatically.

---

## Technical Highlights

**Risk engine** — five weighted signals evaluated *before* settlement. A score of 50+ holds the payment before it reaches the receiver's usable balance, which is the only window in which a freeze costs nobody anything.

| Signal | Weight |
| --- | --- |
| First-ever transfer from this sender | 35 |
| Amount above 3× the account's median deal | 25 |
| 3+ transfers from this sender in 24 hours | 20 |
| Sender account opened under 30 days ago | 20 |
| Round figure at or above ₹50,000 | 15 |

**Spike detection** — the ceiling is derived from each account's own history, so a busy merchant isn't flagged constantly and a quiet one isn't missed:

```
ceiling = median(x) + 3 × 1.4826 × MAD(x)
```

MAD is used instead of standard deviation because it is robust — a single huge day cannot inflate the ceiling enough to hide itself. On the seeded account: median ₹18,200, MAD ₹3,200, ceiling ₹32,433. The ₹47,500 day lands at 2.61× the median and fires the prompt.

**The core innovation** — unlike a messaging app, which shows that a message was edited but hides *when*, every explanation version here keeps its real last-edited timestamp. An explanation written eleven days before any complaint existed cannot have been written to answer it — and that is a fact a reviewer can verify rather than a claim they must weigh.

---

## Roadmap

| Service | Purpose |
| --- | --- |
| Cloud Firestore | Append-only proof-pack ledger readable by both banks and the investigating officer |
| Cloud Functions | Production settlement and hold orchestration |
| Cloud KMS | Institutional key custody for deal sealing |
| Cloud Audit Logs | Tamper-evident record of every hold, release and return |
| NPCI / UPI integration | Issue Deal IDs on real payment rails |

---

## Limitations

This is a prototype, and it should be judged as one. State lives in browser memory, so a page refresh resets it. There is no real banking integration, no NPCI connection and no persistence layer. The demo data is seeded, and the 4-hour settlement hold is compressed to 90 seconds so it can be shown live.

**What is real:** the SHA-256 hashing, the MAD statistics, the risk scoring, the password hashing and access control, and the Gemini integration.

---

## License

This project is licensed under the **MIT License**.

---

## Created By

**KrsnaOn** 💙  Contact: https://github.com/KrsnaOn/Deal-ID-Layer-Transaction-level-freeze

*Deal-ID Layer — because one disputed payment should never cost someone their entire livelihood.*
