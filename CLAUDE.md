# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Mahau CRM — a CRM for **Mahau Bar** (a Brazilian nightclub). The entire application is a **single file, `index.html`** (~1600 lines). There is no build step, no package manager, no test suite. The UI is entirely in Brazilian Portuguese; currency is BRL.

`recover-bday.html` is a standalone one-off recovery script that re-merges and re-saves the chunked birthday base in Firestore. It is not part of the app.

## Running / developing

- Edit `index.html` directly. JSX is transpiled **in the browser** by Babel Standalone (`<script type="text/babel">`), so changes take effect on reload — no compile.
- It must be served over HTTP, not opened via `file://`, because Firebase loads as an ESM module (`<script type="module">`) which browsers block from `file://`. Use any static server, e.g. `python -m http.server` from the repo root, then open the printed URL.
- Deployment is **GitHub Pages** serving `index.html` from the repo root (that's why the file is named `index.html` rather than `mahau-crm.html`). There is no deploy command — pushing to `main` publishes.

## External dependencies (all via CDN, no install)

- **React 18** + **ReactDOM** (UMD production builds)
- **Babel Standalone** — in-browser JSX
- **SheetJS / xlsx** — Excel import & export
- **Firebase 10.12 Firestore** — persistence

## Architecture

### Persistence layer (`window._db`)
Firebase config and the entire DB API are defined in the `<head>` module script and exposed globally as `window._db`. On load it fires a `dbReady` event; `App` waits for `window._dbReady`/the event before loading.

- **All app state lives in a single Firestore document `mahau/crm`.** Every mutation calls `save(nd)`, which writes the *whole* `data` object back. A `onSnapshot` subscription (`window._db.sub`) keeps multiple clients in sync in real time.
- The `data` shape is the `EMPTY` constant: `{customers, visits, goal, adminPin, managerPin, promoterPins, configured, waLogs}`.
- **The birthday base is stored separately** in chunked docs `bday_0..N` plus `bday_meta` (`CHUNK = 5000` contacts each), because it can grow large and a single Firestore doc is capped at ~1MB. Use `saveBday`/`loadBday`; it is loaded once (no real-time sub — `subBday` is a no-op).

When changing the data model, remember both that a save rewrites the entire document and that `EMPTY` is merged over loaded data (`Object.assign({}, EMPTY, d)`), so new fields must be added to `EMPTY`.

### Auth & roles
PIN-based, no real accounts. Three role tiers, gated by booleans derived from `role`:
- `admin` ("Sócio") — `isAdmin`, full access incl. delete and PIN setup.
- `manager` ("Gerente") — `isMgr` (admin OR manager), can add customers/visits and import.
- `promoter_<Name>` — `isPromoter`; sees **only** the "Brief Semanal" view (`NAV` is restricted for promoters).

PINs are checked in `tryPin()` against `adminPin`/`managerPin`/`promoterPins`. Session is persisted in `localStorage` under `mahau_auth` with an 8-hour TTL. First run (`!configured`) forces a setup screen to set PINs.

### Views
`App` is one large component switching on a single `view` string (`{view==="..." && ...}`). Navigation is the `NAV_FULL` array (rendered as buttons). Views: `painel` (dashboard), `clientes`, `perfil` (customer detail), `rfm`, `aniversarios`, `novo` (+customer), `visita` (+visit), `excel` (import), `analise` (text report), `exportar`, `baseaniv` (birthday import/base), `brief` (weekly per-promoter call list), `predicao`, `performance`.

### Core domain logic
- **RFM segmentation** — `rfmSegMahau()` defines the bar's segments: **VIP** (avg ticket ≥ R$1000), **Fiel** (3+ visits in 30d & recent), **Potencial** (recent but <3 visits total), **Em Risco** (3+ visits but gone 14+ days), **Perdido** (everything else). `computeRFM(customers, visits)` builds the enriched list; `rfmExplain()` produces the human "why this segment / what to do" text. `rfmSeg` is legacy/dead — ignore it.
- **Promoter assignment** — `getPromoter(id)` deterministically hashes a customer id to one of the 4 promoters (`PROMOTERS = [Ygor, Matheus, Mahau, Aretha]`). Customers are not stored with an explicit promoter; assignment is computed.
- **Brief Semanal** — per-promoter weekly call sheet. "Called" state is tracked in `localStorage` keyed per promoter and per ISO-week-Monday (`mahau_called_<promoter>_<weekKey>`), so it resets weekly.
- **Excel import** (`readFile`/`doImport`) — auto-detects the header row, maps columns, and **dedups by phone first, then by name**, merging into empty fields on a match.
- **Normalizers** — `normAmt` (BRL string → number), `normPhone`, `normBday` (handles DD/MM, DD/MM/YYYY, YYYY-MM-DD, and Excel serial dates → "DD/MM"). Reuse these rather than re-parsing.
- **WhatsApp** — `waLink()` builds a `wa.me` URL (assumes Brazil +55); `WaBtn` renders the button and logs sends to `data.waLogs` via `logWa()`.
- `analyze()` generates the plain-text period report; `exportPDF()` opens a print window.

## Conventions

- Styling is **inline `style={{}}` objects** throughout — there is no CSS framework and almost no stylesheet (only a few global rules in `<head>`). Match the existing dark gold-on-black palette (`#0a0a0a` bg, `#c8a030` gold accent, `Georgia, serif`). Segment colors live in `SEGS`; promoter colors in `PROMOTER_COLORS`.
- Reusable UI atoms are short single-letter/short-name components: `F` (labeled input), `Sel`, `Bdg` (badge), `Lbl`, `Card`, `GB`/`GH` (buttons), `WaBtn`. Prefer them over re-rolling markup.
- Keep all user-facing strings in Portuguese.

## Important notes

- The Firebase config in `index.html` and `recover-bday.html` is a client-side web config; committing it is expected for Firebase web apps (access is governed by Firestore rules, not secrecy of the config). Do not "fix" this by removing it.
- The "Limpar" (clear data) action and deletes are intentionally guarded (typed confirmation / admin-only) — preserve those guards when editing.
