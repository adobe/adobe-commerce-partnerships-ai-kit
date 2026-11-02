# UI_CODE_PATTERNS.md — adobe-commerce-partnerships-ref-app (UI)

## Part A — General UI Standards

NOT CAPTURED — the `generate-ui-service-card` skill package (`.claude/skills/generate-ui-service-card/`) contains only `SKILL.md`; no separate "Part A" template file with versioned general UI standards exists to copy verbatim. Do not fabricate generic standards here — Part B below documents only what this codebase actually does, sourced from the scans.

## Part B — Project-Specific Patterns

### B.1 Naming Conventions

| Kind | Pattern | Example |
|---|---|---|
| Page | lowercase, no separators, matches URL path | `pages/customerdetails.tsx`, `pages/orderConfirmation.tsx` (mixed casing — inconsistent) |
| Component file | PascalCase, one default export per file | `components/CatalogProductCard.tsx`, `components/customerdetails/AccountDetailsPanel.tsx` |
| Component sub-directory | lowercase domain grouping | `components/checkout/`, `components/customerdetails/`, `components/renewal/lateRenewal/` |
| Hook | `use<Noun>` | `hooks/usePaginatedCustomersList.ts`, `hooks/useCartRouteGuard.ts` |
| Fetch function | `fetch<Noun>` (read) / `search<Noun>` (search) / bare verb for one-offs | `fetchCustomerDetails`, `searchResellers`, `createCustomer` |
| Mutation hook | `use<Verb><Noun>` | `useCreateOrder`, `useReturnOrder` |
| Context + hook pair | `<Noun>Context` + `use<Noun>` (not `use<Noun>Context`) | `CartContext` / `useCart`, `PartnerContext` / `usePartnerDetails` |
| CSS Modules | `<ComponentName>.module.css`, one per component/page, co-located under `styles/` mirroring the page/component's domain folder | `styles/customerdetails/AccountDetailsPanel.module.css` |
| Utils file | `<domain>Utils.ts` or `<domain>.ts` | `utils/customerDetailsUtils.ts`, `utils/renewalUtils.ts`, `utils/cartUtils.ts` |

### B.2 File / Directory Structure

```
pages/                    — Next.js Pages Router (9 routes + _app/_document)
  api/                    — backend API routes (see backend service cards)
components/
  Layout/                 — Header, Sidebar, Layout shell (index.ts barrel export)
  common/                 — cross-domain reusable pieces (ResellerSelector, PromoCodeDialog, EstimatedTotal, ...)
  catalogCart/            — cart + find-or-create-customer flow (+ nested customerdetails/ for New/ExistingCustomer forms)
  customerdetails/        — customer detail tabs (AccountDetailsPanel, PurchaseHistoryPanel, ReturnOrderDialog, ...)
  checkout/               — checkout review + summary
  renewal/                — renewal editing (+ nested lateRenewal/ for the late-renewal flow)
  ThreeYearCommit/        — 3YC enrollment banners + dialogs
  upgrade/                — empty directory (no files) — reserved/leftover, not yet in use
contexts/                 — CartContext, PartnerContext (2 files)
hooks/                    — 7 hooks, all consumed across multiple pages
utils/                    — 19 files: client fetch wrappers, formatters, validators, logger, constants
models/                   — Zod schemas shared with the backend (same files imported by both layers — see backend CODE_PATTERNS.md)
types/                    — plain TS interfaces for request/response shapes not modeled as Zod schemas
styles/                   — CSS Modules, directory structure mirrors components/pages domain grouping
```

### B.3 Data Fetch Library Patterns

Verbatim query (from `hooks/usePaginatedCustomersList.ts`):
```ts
const { data, isLoading, error } = useQuery<PaginatedCustomersResponse>({
  queryKey: [...getCustomerListKey(resellerId, currentPage, itemsPerPage)],
  queryFn: () => fetchCustomersPage(resellerId, currentPage, itemsPerPage),
  enabled: enabled && !isSearchMode && !!resellerId,
  placeholderData: previousData => previousData,
  staleTime: STALE_TIME,
  gcTime: GC_TIME,
});
```
Verbatim mutation (from `hooks/useOrderOperations.ts`):
```ts
export function useCreateOrder() {
  return useMutation({
    mutationFn: async (orderData: CreateOrderRequest) => {
      const response = await fetch('/api/orders?type=NEW', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(orderData),
      });
      if (!response.ok) { /* throw Error with .status attached */ }
      return response.json();
    },
    retry: (failureCount, error: any) => failureCount < 3 && error.status >= 500,
    retryDelay: attemptIndex => Math.min(1000 * 2 ** attemptIndex, 10000),
  });
}
```
See `DATA_LAYER.md` for the full fetch-unit inventory and the annotated standard patterns.

### B.4 State Library Patterns

Verbatim store read (from `pages/resellers.tsx`):
```ts
const { partnerName } = usePartnerDetails();
```
Verbatim store write/action call (from `pages/catalog.tsx`):
```ts
addToCart(offerId, productName, price, filters.currency, marketSegmentCode, productName, 1);
setShowCartModal(true);
```
See `STATE_MANAGEMENT.md` for full state shapes and all actions.

### B.5 Design System

- **Library:** Adobe `@react-spectrum/s2` (`Provider colorScheme="light"` set once in `pages/_app.tsx`) + individual icon imports from `@react-spectrum/s2/icons/*`.
- **Import path pattern:** `import { Button, TextField, ... } from '@react-spectrum/s2';` for components; `import IconName from '@react-spectrum/s2/icons/IconName';` — one default-export import per icon, never a barrel import of icons.
- **Scenario → component map (from actual usage):**

| Scenario | Component(s) used |
|---|---|
| Text input | `TextField` (with `isInvalid`/`errorMessage`/`isRequired` props) |
| Search box | `SearchField` |
| Buttons | `Button` (`variant="primary"\|"secondary"\|"accent"`, `fillStyle="outline"`, `size="S"\|"L"`) |
| Checkbox / toggle | `Checkbox`, `Switch` |
| Numeric input | `NumberField` |
| Tabs | `Tabs`, `TabList`, `Tab`, `TabPanel` |
| Card container | `Card` (with `UNSAFE_className` escape hatch for custom layout — seen in `ReturnOrderDialog.tsx`) |
| Loading indicator | `ProgressCircle isIndeterminate` |
| Radio group (catalog filters) | `RadioGroup`, `Radio` |
| Form wrapper | `Form` (native `onSubmit` handler, no built-in validation wiring — see B.6) |

- **A11y provided automatically by the library:** focus rings, keyboard interaction for `Tabs`/`Switch`/`RadioGroup`, ARIA roles/labels baked into interactive Spectrum components.
- **What is added manually:** every custom (non-Spectrum) interactive element — e.g. the card grid items in `pages/resellers.tsx`/`pages/customers.tsx` are plain `<div role="button" tabIndex={0} onKeyDown={...}>` with hand-written Enter/Space key handling, because they're styled with CSS Modules rather than a Spectrum `Card`/`ListItem`. `NavigationPanel.tsx`'s breadcrumbs are also hand-rolled `<span>` elements with inline styles, not a Spectrum breadcrumb component. `ErrorToast`/`SuccessToast`/`WarningToast` (`utils/ToastMessageUtils.tsx`) are fully custom, with manually-set `role="alert"|"status"` and `aria-live`.
- **Layout/visual styling:** CSS Modules throughout (`*.module.css`), not a CSS-in-JS or utility-class (Tailwind) system — every component/page pairs with its own `.module.css` file in `styles/`.

### B.6 Form Patterns

- **Form library:** none — `react-hook-form` is a declared dependency (`package.json`) but is not imported or used anywhere in the codebase. All forms use plain `useState` per field plus hand-written validator functions.
- **Where validation logic lives:** dedicated `*Validation.ts`/`*Utils.ts` files exporting one `validate<Field>(value): string | null` function per field, plus one `validateRequiredFields(data): ValidationErrors` aggregate (e.g. `utils/customerFormValidation.ts`).
- **When validation runs:** on submit only (`validateRequiredFields` called inside the submit handler) — not on blur or on every keystroke. Per-field error state is cleared reactively as the user types that field (`if (validationErrors[field]) setValidationErrors(...)`), but not re-validated until the next submit attempt.
- **Error display:** Spectrum `TextField`'s own `isInvalid`/`errorMessage` props, driven by the local `validationErrors` boolean map:
  ```tsx
  <TextField
    label="Customer name"
    value={customerData.companyName}
    onChange={value => handleInputChange('companyName', value)}
    isInvalid={validationErrors.companyName}
    errorMessage={validationErrors.companyName ? validateCompanyName(customerData.companyName) : undefined}
    isRequired
  />
  ```
- **Submission UX:** a full-overlay `ProgressCircle` + text label blocks the form while `isLoading`/`isEnrolling` is true; the submit button itself is disabled via `isDisabled={!customerData || isEnrolling}` rather than being replaced by a spinner-in-button.
- **One exception:** `components/ThreeYearCommit/ThreeYearCommit.tsx` uses Spectrum `TextField`'s built-in `validate` prop (a validator function passed directly to the component) instead of the manual `isInvalid`/`errorMessage` pattern used everywhere else — a second, less common form-validation idiom in this codebase.

### B.7 Error Handling in Components

Query error display (repeated verbatim across `resellers.tsx`, `customers.tsx`, `catalog.tsx`):
```tsx
) : error ? (
  <div className={styles.errorMessage}>
    <h3>Unable to load {resource}</h3>
    <p>{error?.message || 'An unexpected error occurred'}</p>
    <Button variant="secondary" onPress={() => invalidate{Resource}(queryClient)}>Try Again</Button>
  </div>
```
Mutation/imperative-action error display — toast-based, via `utils/ToastMessageUtils.tsx`'s `ErrorToast`/`SuccessToast`/`WarningToast` or the `useToastState()` hook that manages all three at once:
```tsx
<ErrorToast show={showErrorToast} message={errorMessage} onClose={() => setShowErrorToast(false)} />
```
No `ErrorBoundary` component exists anywhere in the codebase, and no custom `pages/_error.tsx` — an unhandled render-time exception is not caught by any app-level boundary (see `ROUTES.md → Error Routes`).

### B.8 Test Patterns

| Aspect | Value |
|---|---|
| Framework | Jest 30 via `next/jest`, `jest-environment-jsdom` |
| Component rendering | `@testing-library/react` (`render`, `screen`, `fireEvent`, `act`) + `@testing-library/jest-dom` matchers |
| Test file location | Repo-root `tests/` (not co-located with source); one full-page render test found: `tests/pricelistCurrencyGuard.test.tsx` |
| Naming convention | `<what-it-covers>.test.tsx` — descriptive, not mirroring the source filename |

Component test skeleton — real excerpt from `tests/pricelistCurrencyGuard.test.tsx` (renders a full page, mocks every dependency at the module boundary):
```tsx
import { render, screen, fireEvent, act } from '@testing-library/react';
import CatalogPage from '../pages/catalog';

jest.mock('../contexts/PartnerContext', () => ({ usePartnerDetails: jest.fn() }));
jest.mock('../contexts/CartContext', () => ({
  useCart: jest.fn().mockReturnValue({
    cartItemIdToQuantityMap: {},
    getTotalItems: jest.fn().mockReturnValue(0),
    /* ...every field the component destructures must be present, or it throws */
  }),
}));
jest.mock('@tanstack/react-query', () => ({
  useInfiniteQuery: jest.fn().mockReturnValue({ data: null, isLoading: false, /* ... */ }),
  useQueryClient: jest.fn().mockReturnValue({ prefetchInfiniteQuery: jest.fn(), /* ... */ }),
}));
// Heavy child components replaced with minimal stubs so the test stays focused:
jest.mock('../components/CatalogFilters', () => ({
  default: ({ filters, onFiltersChange }: any) => (
    <select data-testid="currency-select" value={filters.currency}
      onChange={e => onFiltersChange({ ...filters, currency: e.target.value })}>
      <option value="EUR">EUR</option>
    </select>
  ),
}));
```
**Convention to follow:** mock every context hook, every TanStack Query hook, and every non-trivial
child component at the module level via `jest.mock`, so the test isolates the page's own logic
(here: a currency-selection race condition) rather than its children's rendering. There is no
shared test-utility wrapper (e.g. a custom `render` that pre-wraps providers) — each test file
mocks the providers directly instead of rendering through them.

No hook-only unit test (testing a hook via `renderHook`) was found in this codebase — the one
component test present renders the full page instead.