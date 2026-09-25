# 🔒 Deal-ID Layer — Freeze the Transaction, Not the Person

**Smarter fraud holds. Fairer investigations. Innocent accounts stay alive.**

Deal-ID Layer is a transaction-level freeze system for digital payments. Instead of locking an entire bank account when one payment is disputed, every payment gets its own cryptographically sealed identity: a **Deal ID**. A complaint freezes only the disputed amount, while salary, rent, GST and supplier payments continue moving.

Built for **DCGC 2.0 Hack Sprint (GeeksforGeeks × Google Cloud) — Problem Statement 7: Indiscriminate Collateral Account Freezes.**

**🌐 Live:** [dead-id.web.app](https://dead-id.web.app) · **Firebase project:** `dead-id`

## Overview

When stolen money passes through two or three accounts, every account it touched can be frozen in full, including accounts belonging to people who did nothing but receive a legitimate payment. Deal-ID Layer demonstrates a narrower alternative:

- **Payers** propose payments with a stated purpose and a pre-settlement risk score.
- **Payees** accept or bounce incoming money and explain unusual inflows before an accusation.
- **Reviewers** inspect a tamper-evident proof pack and release or return one held transaction.

The prototype implements the transaction-level hold, role isolation, password hashing, risk scoring, versioned explanations, and SHA-256 proof verification in the browser.

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

## Firebase Integration

Firebase is used for **two independent things** in this project: **Authentication** (Google sign-in) and **Hosting** (serving the static site). There is no Firestore, no Realtime Database, and no Cloud Functions — the prototype deliberately keeps all state in browser memory.

| Firebase service | Used? | Purpose |
|---|---|---|
| Firebase Authentication | ✅ Yes | Google sign-in via `signInWithPopup` |
| Firebase Hosting | ✅ Yes | Serves `index.html` as a static SPA |
| Cloud Firestore | ❌ No | Roadmap — cross-institution proof-pack ledger |
| Realtime Database | ❌ No | Not used |
| Cloud Functions | ❌ No | Roadmap — server-side hold orchestration |
| Cloud Storage | ❌ No | Not used |

### Firebase SDK setup

The Firebase SDK is loaded as **ES modules directly from Google's CDN** — there is no `npm install`, no bundler, and no `node_modules`. This keeps the "one HTML file, no build step" property of the project intact.

```html
<script type="module">
  import { initializeApp } from "https://www.gstatic.com/firebasejs/10.13.2/firebase-app.js";
  import {
    getAuth, GoogleAuthProvider, signInWithPopup, signOut as fbSignOut
  } from "https://www.gstatic.com/firebasejs/10.13.2/firebase-auth.js";

  const firebaseConfig = {
    apiKey: "AIza…",                                  // replace with your own
    authDomain: "dead-id.firebaseapp.com",
    projectId: "dead-id",
    storageBucket: "dead-id.firebasestorage.app",
    messagingSenderId: "842375119569",
    appId: "1:842375119569:web:861c11bc0e68674e04bbf0"
  };

  const app = initializeApp(firebaseConfig);
  const auth = getAuth(app);
  const provider = new GoogleAuthProvider();

  window.DealIDFirebase = {
    async signInWithGoogle(){
      const result = await signInWithPopup(auth, provider);
      const u = result.user;
      return { uid: u.uid, name: u.displayName || u.email, email: u.email, photo: u.photoURL };
    },
    async signOut(){ try{ await fbSignOut(auth); }catch(e){ /* already signed out */ } }
  };
</script>
```

**Why `window.DealIDFirebase`?** The Firebase SDK requires `<script type="module">`, but the main application is a classic (non-module) `<script>`. Module scope is not shared with classic scripts, so the module block publishes exactly two functions onto `window`, which the rest of the app calls. This is the only coupling point between Firebase and the application logic.

**Why the API key is safe to commit.** A Firebase **web** API key is a public project identifier, not a credential. It identifies which Firebase project a request belongs to; it grants nothing on its own. Real access control comes from the **Authorized domains** list and the enabled sign-in providers (steps 4–5 below). This is Firebase's documented design and the reason the key ships inside a static HTML file here.

## Authentication

The app has two independent, parallel sign-in systems that share the same three workspaces:

1. **Local password accounts** — the original system. Register with an email, password, and a role fixed at registration. State (users, sessions) lives in browser memory only; see [Security Model](#security-model).
2. **Google sign-in** — backed by Firebase Authentication, added alongside the local system without changing it.

### How Google sign-in works in this app

- The sign-in screen's **Continue with Google** button calls `signInWithPopup` with a `GoogleAuthProvider`, exposed to the app as `window.DealIDFirebase.signInWithGoogle()`.
- Google only ever returns a name and email — never a role — so a **first-time** Google sign-in shows a one-time role picker (payer / payee / reviewer; reviewer still requires the `CYBER-2026` officer code) before the workspace opens.
- The chosen role is saved in the browser's `localStorage` under the key `dealid:googleRole:<email>`, so returning to the same browser on a later visit skips the picker. This does **not** sync across browsers or devices — it's a client-only mapping, matching this prototype's "no backend, no database" design.
- Once resolved, a Google-authenticated user is treated exactly like a password account internally (`establishSession`, workspace guards, the security log). The only difference recorded is `S.auth.how === 'google'`, used solely so **Sign out** also calls Firebase's `signOut()`.
- The scripted **"Run the 3-minute demo"** tour and its three fixed demo accounts are completely untouched by any of this.

### Setting up your own Firebase project

1. Go to the [Firebase console](https://console.firebase.google.com) → **Add project** (or reuse an existing one).
2. **Project settings → General → Add app → Web** (the `</>` icon). Register an app and copy the resulting config object (`apiKey`, `authDomain`, `projectId`, `storageBucket`, `messagingSenderId`, `appId`).
3. Paste that object into the `firebaseConfig` constant inside the `<script type="module">` block near the top of `index.html`.
4. **Authentication → Sign-in method → Add new provider → Google → Enable.** Skipping this step is the most common cause of a sign-in failure (surfaces in this app as `auth/operation-not-allowed`).
5. **Authentication → Settings → Authorized domains.** Add **every** domain the app is served from:

   | Domain | When it's needed |
   |---|---|
   | `localhost` | Local preview — authorized by default |
   | `dead-id.web.app` | Firebase Hosting (primary URL) |
   | `dead-id.firebaseapp.com` | Firebase Hosting (legacy URL) — added by default |
   | `your-app.vercel.app` | Vercel deployment |
   | your custom domain | Any custom domain you attach |

   Firebase Hosting domains for your own project are added automatically, but **Vercel and custom domains are not** — miss one and `signInWithPopup` fails with `auth/unauthorized-domain`.

### Firebase CLI

```bash
npm install -g firebase-tools

firebase login              # opens a browser for Google OAuth
                            # over SSH / WSL where no browser opens:
                            # firebase login --no-localhost

firebase projects:list      # confirm you can see the project
```

Optional — run the **Auth emulator** instead of hitting the real project while developing:

```bash
firebase init emulators     # select Authentication Emulator
firebase emulators:start
```

### Troubleshooting

The app surfaces the real Firebase error code instead of a generic message (e.g. `Google sign-in failed: auth/popup-blocked`). Common ones:

| Error code | Cause | Fix |
|---|---|---|
| `auth/operation-not-allowed` | Google isn't enabled as a sign-in provider | Step 4 above |
| `auth/unauthorized-domain` | The current origin isn't in the authorized domains list | Step 5 above |
| `auth/popup-blocked` | The browser blocked the sign-in popup | Allow popups for the site and retry |
| `auth/popup-closed-by-user` | The account chooser was closed before completing sign-in | Retry and finish the popup flow |
| `auth/network-request-failed` | Offline, or the gstatic CDN is unreachable | Check the connection; local password accounts still work offline |
| `Google sign-in is still loading...` | The `<script type="module">` hasn't finished loading yet | Wait a second and retry |

## Demo Accounts

| Role | User ID | Password |
|---|---|---|
| Payer | `rohit@quickcash.in` | `Quick@4417` |
| Payee | `priya@nairaa.studio` | `Nairaa@9082` |
| Reviewer | `kverma@cybercell.gov.in` | `Cyber@1021` |

## Judging the Demo

Press **Run the 3-minute demo** on the sign-in screen. The guided tour signs in as each party and advances through twelve steps. **Reset demo** in the top bar restores the opening state.

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
| Hosting | Firebase Hosting (primary) with Vercel as a mirror |
| Authentication | Local salted SHA-256 password accounts, plus Google sign-in via Firebase Authentication |
| Firebase SDK | v10.13.2, ES modules from `gstatic.com` — no npm, no bundler |
| Dependencies | No build step, backend, or database. Firebase Auth is the only external service |

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

- Passwords are never stored. Each account uses a random salt and stores the SHA-256 digest of `salt + password`.
- Five failed sign-in attempts lock an account for 30 seconds.
- A user's role is fixed at registration; it cannot be changed at sign-in.
- Cross-role access attempts are blocked and recorded in the visible security log.
- The Firebase web API key is a public project identifier, not a secret. Access is controlled by the Authorized domains list and the enabled sign-in providers.
- The single-file prototype uses inline styles and scripts, so it does not configure a strict CSP.

> ⚠️ **Security headers are per-host.** `vercel.json` headers apply **only** on Vercel. Firebase Hosting ignores that file entirely, so the Firebase deployment serves without them unless you add a `headers` block to `firebase.json` — see [Firebase Hosting](#1-firebase-hosting-primary) below for a config that sets the same headers on both hosts.

This is a prototype. State lives in browser memory, there is no real banking or NPCI integration, and the four-hour settlement hold is compressed to 90 seconds for the demo. A production system should use a slow password KDF such as bcrypt or Argon2 and an append-only server-side proof ledger.

## Deployment

The project deploys to two static hosts from the same folder. No build step, no environment variables.

| Host | URL | Config file |
|---|---|---|
| Firebase Hosting | `https://dead-id.web.app` | `firebase.json` + `.firebaserc` |
| Vercel | `https://<project>.vercel.app` | `vercel.json` |

### 1. Firebase Hosting (primary)

#### Configuration files

`.firebaserc` — maps the local folder to a Firebase project, so `firebase deploy` knows where to publish without a `--project` flag every time:

```json
{
  "projects": {
    "default": "dead-id"
  }
}
```

`firebase.json` — the hosting config:

```json
{
  "hosting": {
    "public": ".",
    "ignore": [
      "firebase.json",
      "**/.*",
      "**/node_modules/**"
    ],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ]
  }
}
```

| Key | Value | What it does |
|---|---|---|
| `public` | `"."` | Deploys the project root itself. There is no `dist/` because there is no build step |
| `ignore` | `firebase.json`, `**/.*`, `**/node_modules/**` | Excludes the config itself, all dotfiles (so `.git`, `.firebase`, `.gitignore` never ship), and dependencies |
| `rewrites` | `**` → `/index.html` | SPA fallback — every path serves the app instead of a 404 |

> **Note on `"public": "."`** — because the root is the deploy directory, non-dotfiles like `README.md` and `vercel.json` are uploaded and publicly readable at `https://dead-id.web.app/README.md`. That's harmless here (both are already public on GitHub), but add them to `ignore` if you'd rather they weren't served.

#### Recommended: add security headers

Firebase Hosting does not read `vercel.json`. To apply the same headers on both hosts, extend `firebase.json`:

```json
{
  "hosting": {
    "public": ".",
    "ignore": [
      "firebase.json",
      "**/.*",
      "**/node_modules/**",
      "README.md",
      "vercel.json"
    ],
    "rewrites": [
      { "source": "**", "destination": "/index.html" }
    ],
    "headers": [
      {
        "source": "**",
        "headers": [
          { "key": "X-Content-Type-Options", "value": "nosniff" },
          { "key": "X-Frame-Options", "value": "SAMEORIGIN" },
          { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" },
          { "key": "Permissions-Policy", "value": "geolocation=(), microphone=(), camera=(), payment=()" }
        ]
      }
    ]
  }
}
```

#### First-time setup

Only needed if you're pointing this at your own Firebase project:

```bash
firebase login
firebase init hosting
```

At the prompts:

| Prompt | Answer |
|---|---|
| Use an existing project | Select your project |
| Public directory | `.` |
| Configure as a single-page app | **Yes** |
| Set up automatic builds with GitHub | No (optional) |
| Overwrite `index.html`? | **No** — this would destroy the application |

> The last prompt matters. Answering "Yes" replaces `index.html` with Firebase's placeholder page and wipes the entire app.

#### Deploy

```bash
cd deal-id-layer
firebase deploy --only hosting
```

The CLI prints the live URL on success. Deploys are near-instant because the payload is a single HTML file.

#### Preview channels

Share a temporary URL for review without touching production — useful for showing judges a work-in-progress build:

```bash
firebase hosting:channel:deploy demo --expires 7d
```

Preview channel URLs are `https://dead-id--demo-<hash>.web.app`. They are **auto-added** to the Authorized domains list, so Google sign-in works on them.

#### Rollback

```bash
firebase hosting:versions:list        # find the version to restore
```

Or use **Firebase console → Hosting → Release history → ⋮ → Rollback** for a one-click revert to any previous release.

#### The `.firebase` cache

Running `firebase deploy` creates a local `.firebase/` folder containing `hosting..cache` — a manifest of file paths, modification times, and hashes used to skip re-uploading unchanged files. It is a build artifact, already listed in `.gitignore`, and safe to delete at any time.

### 2. Vercel (mirror)

#### Drag and drop

1. Go to [vercel.com/new](https://vercel.com/new).
2. Drag this folder onto the drop zone.
3. Choose framework preset **Other** and leave build and output directory fields empty.
4. Deploy.

#### Vercel CLI

```bash
cd deal-id-layer
npx vercel          # preview deployment
npx vercel --prod   # production deployment
```

#### GitHub

```bash
git init
git add .
git commit -m "Deal-ID Layer prototype"
git branch -M main
git remote add origin https://github.com/<you>/<repo>.git
git push -u origin main
```

Import the repository at [vercel.com/new](https://vercel.com/new). No environment variables are needed.

> After deploying to Vercel, add the `*.vercel.app` domain to **Firebase → Authentication → Settings → Authorized domains**, or Google sign-in will fail there with `auth/unauthorized-domain`.

### 3. Google Cloud Console (optional)

A Firebase project is backed by a Google Cloud project of the same ID. Open [console.cloud.google.com](https://console.cloud.google.com), select `dead-id`, and use it to:

- Review the **Identity Toolkit API** (which backs Firebase Authentication) under APIs & Services
- Restrict the web API key by HTTP referrer under **APIs & Services → Credentials**
- Manage IAM roles for teammates who need console access
- Monitor quotas and set up billing if you move beyond the Spark (free) tier

## Local Preview

```bash
npx serve .
# or
python3 -m http.server 8000
# or, matching production exactly:
firebase emulators:start --only hosting
```

Open the printed URL. Serving the folder over HTTP keeps local behavior close to production and allows the browser's Web Crypto API to run consistently. `localhost` is authorized in Firebase by default, so Google sign-in works locally without extra configuration.

> Opening `index.html` directly as a `file://` URL will break both Google sign-in (Firebase rejects the `null` origin) and SHA-256 hashing (Web Crypto requires a secure context). Always serve over HTTP.

## Files

```text
deal-id-layer/
├── index.html      # Complete application: auth, portals, risk engine, proof pack, Firebase module
├── firebase.json   # Firebase Hosting config — public dir, ignore rules, SPA rewrites
├── .firebaserc     # Firebase project alias (default → dead-id)
├── vercel.json     # Vercel static config and security headers
├── .gitignore      # Excludes .vercel, .firebase, .DS_Store, node_modules
└── README.md       # Project documentation
```

Generated at deploy time and not committed:

```text
├── .firebase/      # Firebase CLI upload cache (hosting..cache)
└── .vercel/        # Vercel CLI project link
```

## Roadmap

| Service | Purpose |
|---|---|
| Cloud Firestore | Append-only proof-pack ledger readable by both banks and the investigating officer |
| Cloud Functions | Server-side settlement and hold orchestration |
| Firebase App Check | Block requests from anywhere other than the real app |
| Cloud KMS | Institutional key custody for deal sealing |
| Cloud Audit Logs | Tamper-evident record of every hold, release, and return |
| NPCI / UPI integration | Issue Deal IDs on real payment rails |

## License

This project is licensed under the **MIT License**.

**Deal-ID Layer — because one disputed payment should never cost someone their entire livelihood.**
