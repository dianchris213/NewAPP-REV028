# Catatan Keuangan — Mini App

A mobile-first personal finance mini app built with TanStack Start, React 19, TypeScript, and Tailwind CSS. All data stays on the device (`localStorage`), so the app runs with no backend.

## Features

### Transactions — strict input sequence
`src/components/AddTransactionSheet.tsx` enforces a progressive, bank-grade input order. Each step stays disabled until the previous one is valid:

1. **Amount** — numeric only, > 0 and ≤ 1,000,000,000,000, formatted as `id-ID`
2. **Wallet account** — pick the account family (Tunai / Bank Utama / E-Wallet), then the registered account; expenses are blocked when the balance is insufficient
3. **Category** — chosen from the user's own categories (account-specific categories are marked with `•`)
4. **Date** — validated calendar date, defaults to today
5. **Note** — optional, max 80 characters

Saving is optimistic (pending state), the sheet is a focus-trapped `role="dialog"`, and `Esc` closes it.

- **All Transactions sheet** (`src/components/AllTransactionsSheet.tsx`): filter by month, week, type, category, keyword; reset filters; "current month" shortcut from the bottom navigation.
- **Transaction list** (`src/components/TransactionList.tsx`): inline edit and delete.

### Categories start empty
The app ships with **zero** categories. Users create them in **Settings → Kategori Transaksi**, per type (income/expense) and optionally scoped to a single wallet account. Duplicate names within the same type + account scope are rejected; names must be 2–24 characters.

### Wallet (`/wallet`)
- Combined balance across all accounts plus per-family grouping.
- **Card-based Add Wallet flow — no native `<select>`**: first choose one of three type cards (Tunai, Bank Utama, E-Wallet); selecting a type instantly reveals a responsive grid of provider cards (BCA, Mandiri, BNI, BRI, CIMB Niaga, Permata for banks; GoPay, OVO, DANA, ShopeePay, LinkAja for e-wallets; physical cash sources for Tunai). The provider selection auto-suggests the account name, and the sheet validates name length (2–30) and starting balance.
- **Per-account history sheet**: tapping a wallet card opens a drawer showing only the transactions that moved that account's balance, sorted newest first, with signed amounts and the account's current balance in the header.
- **Top Up** and **Transfer** sheets with balance checks and toast confirmation.
- **Wallet activity feed** filterable by Top Up / Transfer / All.

### App Lock
- Toggled in **Settings → App Lock**. Enabling it arms the lock immediately and it is restored on every app start.
- The **Wallet route is actively protected**: while App Lock is on and the session is locked, `/wallet` renders an unlock challenge screen and no financial data (balances, accounts, activity) is rendered at all until the user unlocks.

### Settings (`/settings`)
- Language toggle (ID / EN) — the whole settings screen is translated via `src/lib/i18n.ts`
- App Lock, push notifications, dark/light theme, cloud sync
- Category management (create/delete, per type and account)
- Local avatar upload (read as a data URL, never uploaded)
- Sign out and destructive account actions

### Other
- Analytics overview (`/analytics`)
- Telegram / Google style mock login (`/login`, `/signup`)
- Accessibility: `role="dialog"` + `aria-modal`, focus traps, `Esc` to close, `role="radiogroup"`/`aria-checked` on all card selectors, `role="alert"` inline errors, and body-level portals so sheets sit above the bottom navigation

## State

`src/lib/app-store.tsx` is a single React context store: user, transactions, wallets, wallet activity, categories, settings, language, lock state, and transaction filters. Persisted to `localStorage` (`tmab-state-v1`) with debounced writes.

### Delete + Undo (a11y)
- Delete opens a focus-trapped `role="alertdialog"`; focus lands on the destructive action.
- `Enter` confirms, `Escape` cancels and returns focus to the trigger button.
- After deletion a toast exposes an **Urungkan** (undo) action that receives focus; `Enter` restores the record, `Escape` dismisses the toast and returns focus to the trigger.

## Development

```sh
bun install
bun run dev
```

## Testing (Vitest)

Vitest runs in `jsdom` with Testing Library (`vitest.config.ts`, setup in `src/tests/setup.ts`).

```sh
bun run test          # single run of all suites (vitest run)
bun run test:watch    # watch mode
bunx vitest run src/tests/delete-undo.test.tsx   # delete dialog + undo toast only
```

Current suites:
- `src/tests/delete-undo.test.tsx` — delete confirmation dialog and undo toast: initial focus, Enter to confirm/undo, Escape to cancel/dismiss with focus return, Tab focus trap.
- `src/tests/fund-source.test.tsx` — fund source CRUD, duplicate guard, in-use delete block, search/type filter, audit log.

Typecheck and production build:

```sh
bunx tsgo --noEmit
bun run build
```

## Built with

- TanStack Start (TanStack Router)
- TypeScript
- React 19
- Tailwind CSS
- sonner (toasts)

## Sumber Dana & Kantong (REV017+)

- **Settings → Sumber Dana**: pencarian nama + filter Jenis, urutan field `Jenis` lalu `Nama Sumber Dana`.
- **Hapus**: dialog konfirmasi (`role="alertdialog"`) hanya untuk sumber dana yang tidak dipakai; yang masih dipakai diblokir dengan pesan inline. Setelah dihapus muncul toast dengan aksi **Urungkan** (`restoreWallet`).
- **Tambah Kantong**: urutan `Jenis` → `Nama Sumber Dana` → `Nama Kantong` → `Saldo Awal`, validasi inline per-field, dan penyimpanan diblokir bila belum ada / belum memilih Sumber Dana.
- **Empty state**: daftar bawaan dihapus total; Sumber Dana & pilihan provider kosong sampai user membuatnya sendiri.
- Test: `bun run test` (lihat bagian **Testing (Vitest)**).

## Validation, quality gates & monitoring (REV021)

### Fund source name rule — minimum 3 characters
Fund source names are validated in two layers and both enforce the same rule:

- UI (`src/routes/settings.tsx`): `minLength=3`, `maxLength=24`, inline `role="alert"` error, `aria-invalid` / `aria-errormessage` wiring.
- Store (`src/lib/app-store.tsx` → `addWallet` / `renameWallet`): the name is trimmed and whitespace-collapsed, then rejected when it is shorter than 3 or longer than 24 characters, or when it duplicates an existing name in the same type + provider.

So `"Ka"` and `"  Ab  "` are rejected, `"Kas"` is accepted.

### API error handling for saving a fund source
`src/lib/wallet-api.ts` is the persistence seam (`persistWallet`, `WalletApiError` with an HTTP `status`). When the commit fails:

1. the optimistic row is rolled back (no phantom fund source),
2. a clear toast is shown (`Gagal menyimpan sumber dana. Coba lagi.` + hint that the input was kept),
3. the typed name stays in the form and focus returns to the field so the user can retry with one click,
4. a severity-tagged Sentry issue is created.

### Scripts

```sh
bun run typecheck   # tsc --noEmit (zero errors required)
bun run test        # Vitest: unit, flow and visual-contract suites
bun run lint        # ESLint incl. jsx-a11y
bun run build       # production build
bun run smoke       # smoke test against the production build (scripts/smoke.mjs)
```

Suites added in REV021:
- `src/tests/fund-source-add-flow.test.tsx` — end-to-end add flow, 3-character rule, live search filtering.
- `src/tests/fund-source-api-error.test.tsx` — API failure: toast, rollback, preserved input, focus recovery, Sentry severity mapping.
- `src/tests/visual-regression.test.tsx` — layout contract snapshots for the delete dialog and the Undo toast (container classes, element sizes, focus position). Update intentional changes with `bunx vitest run -u`.

Suites added in REV026:
- `src/tests/fund-source-reload-recovery.test.tsx` — E2E: load failure → "Muat ulang daftar" → successful refetch renders the real list; the empty state is never shown, and focus stays inside the sheet (on the retry button when the reload fails again).
- `src/tests/toast-a11y.test.tsx` — error toasts never steal focus on appear, expose a keyboard-reachable close button, return focus to the trigger on dismiss/auto-close, and leave focus untouched when the user already moved on.
- `src/tests/toast-focus-sequence.test.tsx` — regression for several error/success toasts raised in sequence: focus stays on the triggering control, is never parked on a removed toast node while stacked toasts are closed one by one, is not stolen when the user moved elsewhere, and axe reports no violations with multiple toasts visible.
- `src/tests/toast-close-keyboard.test.tsx` — E2E keyboard: the toast close button is focusable (no positive `tabindex`) and closes with Enter or Space, focus returns to the trigger, Tab continues normally afterwards; `Alt+T` reveals the toast region on demand, focus returns correctly after closing, and pressing it with no toast visible changes nothing.

### Continuous integration
`.github/workflows/ci.yml` runs on every pull request (and pushes to `main`): install → lint → typecheck → unit tests → E2E flow suites → axe-core accessibility suites → production build. Each layer is a separate step, so a failing PR shows immediately whether the regression is functional or accessibility-related.

### Toast accessibility
`src/lib/toast-a11y.ts` wraps sonner (`toastError` / `toastSuccess` / `toastInfo`): it records the focused element when a toast is raised and restores focus **only** if focus was actually lost (`<body>` or a detached node) after dismiss/auto-close. The toaster (`src/components/ui/sonner.tsx`) ships a close button on every toast and an `Alt+T` hotkey to jump into the toast region on demand — focus is never moved automatically, so keyboard navigation is not disrupted.



### CI workflow
`.github/workflows/ci.yml` runs on every pull request and on pushes to `main`: install → lint → typecheck → test → build → smoke. Any failing step blocks the merge.

### Enabling Sentry
Monitoring (`src/lib/monitoring.ts`) is inert until a DSN is configured. Set these environment variables (e.g. in `.env`):

```sh
VITE_SENTRY_DSN=https://<key>@o0.ingest.sentry.io/<project>
VITE_SENTRY_ENV=production      # optional, defaults to the Vite mode
VITE_APP_RELEASE=rev021         # optional, used for release health
```

Behaviour once enabled:
- `initMonitoring()` boots Sentry in the browser only, with PII scrubbing (`beforeSend`) so no financial data leaves the device.
- `captureApiError(error, { operation, status })` creates one stable issue per operation + status (fingerprint `["api", operation, status]`) and tags `api.operation`, `api.status`, `api.severity`.
- Severity routing (`severityForStatus`): `4xx → warning`, `5xx → error`, unknown/transport → `fatal`. Configure Sentry alert rules on `level` or the `api.severity` tag (e.g. page on `fatal`, alert on `error`, digest `warning`).

## A11y tests (REV022)

- `src/tests/a11y.test.tsx` — axe-core scan (`axe-core`, jsdom) on the Sumber Dana sheet and the delete/undo flow: no serious/critical violations, `role="dialog"`/`aria-modal`, focus-trap, `aria-live` status announcements.
- `src/tests/undo-after-api-error.test.tsx` — failure → retry → delete → undo cycle keeps the typed input and focus correct.
- Run only the accessibility layer:

```sh
bunx vitest run src/tests/a11y.test.tsx
bunx vitest run src/tests/undo-after-api-error.test.tsx
```

ESLint (`bun run lint`) also enforces `eslint-plugin-jsx-a11y`, so A11y regressions fail CI before the tests run.

## Sentry source map upload (CI env)

`@sentry/vite-plugin` uploads source maps during `bun run build` when all three variables are present; without them the build simply skips the upload.

```sh
SENTRY_AUTH_TOKEN=sntrys_...   # token with project:releases + org:read scope (secret)
SENTRY_ORG=your-org-slug
SENTRY_PROJECT=your-project-slug
```

Add them as GitHub repository secrets (Settings → Secrets and variables → Actions) and expose them to the build step in `.github/workflows/ci.yml`. Never commit the token; rotate it if it leaks.

## Reading the snapshot artifact in a PR

Every pull-request run uploads `visual-regression-<PR number>` with `if: always()`, so it exists even when the test step fails.

1. Open the PR → **Checks** → job **verify**.
2. Scroll to **Artifacts** and download `visual-regression-<PR number>`.
3. It contains `src/tests/__snapshots__/**` (committed layout contracts) and `test-report/vitest-report.json` (the Vitest diff report — failed assertions include the expected/received snapshot diff).
4. If the diff is an intentional design change, run `bunx vitest run -u` locally and commit the updated snapshot; otherwise fix the component.

## Fund source filters (Sumber Dana)

- `src/lib/fund-source-filter.ts` holds pure helpers: `filterWallets`, `matchesFilters`, and `sanitizeFilters`.
- Invariant: a persisted filter never hides fund sources. On hydration, a stored type filter that matches no wallet is reset to `all`, and a stored query with zero matches is cleared. The sanitizer is idempotent, so no one-shot ref is needed.
- Filters changed by the user in the current session are respected as-is (an intentional "no results" empty state still shows).
- The Reset filter button clears `tmab-fund-source-query` / `tmab-fund-source-type` from localStorage and announces the reset via an `aria-live="polite"` summary (`fund-source-filter-reset-notice`).
- Tests: `src/tests/fund-source-filter.test.ts` (unit) and `src/tests/fund-source-persisted-filter.test.tsx` (integration/E2E: stale `cash` filter with `bank` wallets, reset + focus, API error path).

## Fund source identity (Sumber Dana)

- Wallet ids come from `createWalletId()` (`crypto.randomUUID` with a counter
  fallback). `Date.now()` alone collided within one millisecond, which made a
  second row (BRI) overwrite the first (BCA) because they shared a React key.
- List rows are keyed by `w.id` only — never by name, type, or provider.
- Restored state passes through `dedupeWallets()`: malformed rows are dropped and
  ids are made unique, so distinct fund sources are never merged.
- Persisted filters are validated on load by `sanitizeFilters()`: a stored type
  filter that matches no wallet falls back to `all`, so the list can never render
  an empty state while real fund sources exist.
- Regression coverage: `src/tests/fund-source-identity.test.tsx`.
