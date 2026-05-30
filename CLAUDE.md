# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Mahau CRM — a CRM for **Mahau Bar** (a Brazilian nightclub). The entire application is a **single file, `index.html`** (~1600 lines). There is no build step, no package manager, no test suite. The UI is entirely in Brazilian Portuguese; currency is BRL.

`recover-bday.html` is a standalone one-off recovery script that re-merges and re-saves the chunked birthday base in Firestore. It is not part of the app.

## Running / developing

- Edit `index.html` directly. JSX is transpiled **in the browser** by Babel Standalone, so changes take effect on reload — no compile.
- It must be served over HTTP, not opened via `file://`, because Firebase loads as an ESM module which browsers block from `file://`. Use `npx http-server -p 8080` from the repo root.
- Deployment is **Vercel** serving `index.html` from the repo root. Pushing to `main` triggers Vercel to republish.

## External dependencies (all via CDN, no install)

- **React 18** + **ReactDOM** (UMD production builds)
- **Babel Standalone** — in-browser JSX
- **SheetJS / xlsx** — Excel import & export
- **Firebase 10.12 Firestore** + **Firebase Auth** — persistence and authentication

## Architecture

### Persistence layer (`window._db`)

Firebase config and the entire DB API are defined in the `<head>` module script and exposed globally as `window._db`. On load it fires a `dbReady` event; `App` waits for `window._dbReady` before loading.

- All app state lives in a single Firestore document `mahau/crm`. Every mutation calls `save(nd)`, which writes the whole `data` object back. A `onSnapshot` subscription keeps multiple clients in sync in real time.
- The `data` shape is the `EMPTY` constant: `{customers, visits, goal, waLogs}`. Legacy fields `adminPin`/`managerPin`/`promoterPins`/`configured` were removed in the Firebase Auth migration.
- The birthday base is stored separately in chunked docs `bday_0..N` plus `bday_meta` (CHUNK = 5000 contacts each). Use `saveBday`/`loadBday`; it is loaded once.

### Auth & roles

**Firebase Auth (email/password)** with custom claims for roles. Six real accounts live in the `mahaucrm` Firebase Auth project. The `role` custom claim drives `isAdmin`/`isMgr`/`isPromoter` booleans in the app.

Three role tiers:
- `admin` ("Sócio") — `isAdmin`, full access incl. delete actions.
- `manager` ("Gerente") — `isMgr` (admin OR manager), full read/write minus admin-only destructive actions.
- `promoter` (Ygor, Matheus, Mahau) — `isPromoter`; sees only the "Brief Semanal" view. Cannot write to Firestore from the app.

Custom claims are applied via the one-off `set-claims.mjs` script, which requires a Firebase Admin SDK service account key. The key is revoked and deleted after use. Generate a new one only when claims need to change.

Firestore Security Rules enforce RBAC server-side: default-deny + role-based access on `/mahau/crm` (staff reads, manager+ writes) and `/mahau/bday_*` (staff reads, manager+ writes). Default-deny on any other collection.

### Views

`App` is one large component switching on a single `view` string. Navigation is the `NAV_FULL` array. Views: `painel`, `clientes`, `perfil`, `rfm`, `aniversarios`, `novo`, `visita`, `excel`, `analise`, `exportar`, `baseaniv`, `brief`, `predicao`, `performance`.

### Core domain logic

- **RFM segmentation** — `rfmSegMahau()` defines bar segments: VIP (avg ticket ≥ R$1000), Fiel (3+ visits in 30d), Potencial (recent but <3 visits), Em Risco (3+ visits but gone 14+ days), Perdido. `computeRFM(customers, visits)` builds the enriched list; `rfmExplain()` produces the human "why this segment / what to do" text.
- **Promoter assignment** — `getPromoter(id)` deterministically hashes a customer id to one of the promoters (`PROMOTERS = [Ygor, Matheus, Mahau]`). Assignment is computed, not stored.
- **Brief Semanal** — per-promoter weekly call sheet. "Called" state is tracked in localStorage keyed per promoter and per ISO-week-Monday, so it resets weekly.
- **Excel import** — auto-detects header row, maps columns, dedups by phone first then by name.
- **Normalizers** — `normAmt` (BRL string → number), `normPhone`, `normBday`. Reuse rather than re-parsing.
- **WhatsApp** — `waLink()` builds a `wa.me` URL; `WaBtn` renders the button and logs sends to `data.waLogs`.

## Conventions

- Styling is inline `style={{}}` objects throughout — no CSS framework. Match the dark gold-on-black palette (`#0a0a0a` bg, `#c8a030` gold accent, `Georgia, serif`).
- Reusable UI atoms: `F` (labeled input), `Sel`, `Bdg`, `Lbl`, `Card`, `GB`/`GH`, `WaBtn`. Prefer them over re-rolling markup.
- Keep all user-facing strings in Portuguese.

## Security (XSS)

- **`exportPDF` / `document.write` is the ONLY raw-HTML sink in the app.** Everything else renders through React/JSX, which auto-escapes. There is no `dangerouslySetInnerHTML`.
- **RULE: any output that builds HTML by string (`document.write`, `innerHTML`) MUST pass through `esc()`.** If you add a new raw export/render in the future, run user data through `esc()` before interpolating it.
- `esc()` and `cleanName()` live in the **Utils** block. `esc()` = the XSS control (HTML-escapes `& < > " '` on output). `cleanName()` = data hygiene on input (trims, collapses whitespace, strips control chars, caps length) — it is **not** an XSS control and does not restrict charset (accents, apostrophes like `D'Avila`, hyphens pass through).
- `notes` and `vPro` (promoter) are stored raw and today only ever hit JSX (safe). If either is ever rendered into a raw-HTML sink, escape it with `esc()`.
- Why it matters: a manager can write a customer `name`; that name flows into `exportPDF` when an admin exports the Análise PDF — without `esc()` it's a manager→admin privilege escalation. Promoters cannot write to Firestore (blocked by the rules), so they are not a write-side vector.

## Important notes

- The Firebase config in `index.html` and `recover-bday.html` is a client-side web config; committing it is expected for Firebase web apps. Access is governed by Firestore rules, not secrecy of the config. Do not remove it.
- The "Limpar" action and deletes are intentionally guarded (typed confirmation / admin-only) — preserve those guards.
- Service account keys (`*-firebase-adminsdk-*.json`) are gitignored and must never be committed. Generate keys only when needed, revoke and delete after use.
