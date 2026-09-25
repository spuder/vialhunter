# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

VialHunter — a vendor-agnostic peptide price comparison tool. Pure static JAMstack app: **one `index.html` file (~2,300 lines), no build step, no server, no auth, no package.json**. All application state lives in the browser's localStorage; an optional Supabase project can be paired for cross-device sync.

## Commands

There is no build/lint/test tooling in this repo. To work on the app:

- **Run it**: open `index.html` directly in a browser, or serve the directory with any static file server (e.g. `python3 -m http.server`) — needed if you want to test the Supabase pairing-code deep link or anything relying on a real origin.
- **Deploy**: pushing to `main` triggers `.github/workflows/static.yml`, which uploads the entire repo as-is to GitHub Pages (no build step in CI either). Live at https://spuder.github.io/vialhunter/.
- There are no automated tests. Verify changes by opening the app in a browser and exercising the relevant flow (see README for feature list).

## Architecture

Everything — HTML, CSS, and JS — lives in `index.html`, delimited by `//JS-START` for the script section. Two third-party libraries are loaded from CDN unconditionally (PapaParse for CSV, SheetJS/xlsx for Excel); `@supabase/supabase-js` is lazy-loaded from a CDN only when a user actually pairs a Supabase project, keeping the app static/no-build by default.

### Data model

- `db` (localStorage key `vialhunterDB_v1`): `{vendors, warehouses, cart, settings}`. Vendors have contacts (with WhatsApp numbers) and can be "banished" (soft-hidden, excluded from optimizer/best-price). Each vendor owns one or more warehouses; each warehouse holds `items: [{code, name, spec, price}]`, plus `shipping`, `freeOver`, `minOrder`, `country`.
- `ordersDb` (key `vialhunterOrdersDB_v1`): purchase-tracking history, **deliberately separate from `db`** so personal order history never rides along in a shared vendor/pricing export. Each order has a shipping stepper (Ordered → Paid → Shipped → Received) and independent `tests[]` entries with their own stepper (Sample Shipped → Testing → Results).
- `syncMeta` (key `vialhunterSyncMeta_v1`): last-synced timestamp/error, sync-only bookkeeping, never exported.
- One-time migration functions run at load (`migrateOrderPhotos`, `migrateOrderTests`, the `OLDKEY`/pre-rename migration) — follow this pattern (idempotent, guarded by a shape check, `changed` flag before re-saving) if you add another schema shift.
- `settings.apiKey` (Claude API key) and `settings.supabaseUrl/supabaseKey` are always stripped before Export and stripped from anything Imported — never remove those `delete` calls when touching `exportPayload()` / import handlers.

### Catalog / matching engine (`buildCatalog`)

Prices from every warehouse are merged into cross-vendor product entries using a union-find over two kinds of keys: `name+normalized-dose` and `normalized-code` (including a dose inferred from a code's trailing number, e.g. `BC5` → `5mg`). Items with no resolvable dose and no code fall back to name-only matching. This is the core "same peptide, different vendor" matching logic — if you change matching behavior, `buildCatalog`'s key-generation logic is the single place to look, and `renderResults`' `baseKey`/`gmap` grouping (same-name-different-dose subgrouping in the UI) sits downstream of it.

### Cart optimizer (`optimize`)

Given the cart and the catalog, brute-forces every subset of warehouses (bitmask, feasible up to 14 warehouses) to find the cheapest full/partial coverage plan, accounting for per-warehouse shipping cost, free-shipping thresholds, and minimum order amounts. Preference order when comparing candidate plans: item coverage > minimum-order satisfied > lower total. Falls back to evaluating the full warehouse set (no combinatorial search) above 14 warehouses. Also computes "buy everything from one vendor" single-warehouse comparisons (with pins ignored, so "missing items" reflects real stock).

### Price list ingestion

`handleFile` dispatches by extension:
- **CSV/XLSX**: parsed client-side (PapaParse / SheetJS) into a 2-D grid, then `extractFromGrid` detects meta rows (Shipping/Minimum/Upload Date) and a header row (Code/Name/Specification/Price, tolerant of column order and naming) before falling back to positional guessing for headerless files.
- **PDF/images**: `extractMedia` sends the file as a base64 document/image block directly to `api.anthropic.com/v1/messages` from the browser (`anthropic-dangerous-direct-browser-access` header), using the user's own API key from Settings. The prompt is specifically tuned for merged/spanning-cell vendor price tables (one product name spanning several dose/variant rows) — read it before changing extraction behavior, since it encodes hard-won constraints about not letting a merged cell's value bleed into adjacent rows.

All ingestion paths converge on `openReview`/`gridToReview`, which show an editable review table before anything is written to `db`.

### Supabase sync (optional, off by default)

`save()` and `saveOrders()` both call `scheduleSync()`, which debounces (2s) a full-snapshot upsert of vendors/cart/orders to a single row per project, keyed by pairing. `suppressSync` guards against echoing a just-applied remote snapshot back out. Conflict handling (`openSbConflict`) prompts the user to pick a side when pairing a project that already has data; pairing an empty project offers a one-way local→remote migration. `applyRemoteSnapshot` always preserves the local device's own `apiKey`/Supabase credentials rather than accepting them from the remote row — credentials are per-browser, never synced.

## Working in this codebase

- Since it's a single file with no build step, there's no module boundary to preserve — but existing sections are organized under `/* ============ section ============ */` comments (state, Supabase sync, orders, catalog, results, cart, sync UI, etc.). Keep new code near its section rather than appending at the end.
- CSS custom properties (`--bg`, `--panel`, `--accent`, etc.) define both the dark ("Graphite & Gold") and light ("Warm Paper") themes via `:root` and `:root[data-theme="light"]`. Add new colors as variables in both blocks, not hardcoded hex values.
- `esc()` is used pervasively for HTML-escaping user data before template-literal interpolation into `innerHTML` — never interpolate user-controlled strings (vendor names, item names, notes, etc.) without it.
- `isQuotaError()` + a toast is the established pattern for localStorage `setItem` failures (photos/COAs can fill the ~5-10MB quota) — follow it for any new code that writes to localStorage directly.
