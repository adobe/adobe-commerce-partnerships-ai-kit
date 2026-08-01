# UI Code Patterns — Bridge

## A. Directory & Naming Conventions

### A.1 File naming

| Layer | Convention | Example |
|-------|-----------|---------|
| Pages | PascalCase + `Page` suffix optional | `customers.tsx`, `customerdetails.tsx` |
| Components | PascalCase | `AccountDetailsPanel.tsx`, `EditRenewalDialog.tsx` |
| Hooks | camelCase, `use` prefix | `usePaginatedCustomersList.ts` |
| Contexts | PascalCase + `Context` | `CartContext.tsx`, `PartnerContext.tsx` |
| Utils | camelCase | `renewalUtils.ts`, `queryUtils.ts` |
| Types | camelCase | `discountLevel.ts` |
| Models | PascalCase | `Customer.ts`, `Subscription.ts` |
| Styles | `<ComponentName>.module.css` or `<area>/<ComponentName>.module.css` | `CustomerDetails.module.css` |

### A.2 Directory structure

```
bridge/
├── pages/           # Next.js pages router (both UI pages and /api routes)
│   └── api/         # BFF API route handlers
├── components/
│   ├── Layout/      # Header, Sidebar, Layout wrapper
│   ├── common/      # Reusable micro-components (PromoCodeButton, EstimatedTotal)
│   ├── customerdetails/  # Customer hub panels + dialogs
│   ├── renewal/     # EditRenewalDialog + sub-components
│   │   └── lateRenewal/ # Late renewal specific components
│   ├── ThreeYearCommit/ # 3YC banners + dialogs
│   ├── catalogCart/ # Cart sidebar + FindOrCreateCustomer
│   └── checkout/    # Checkout review + summary
├── hooks/           # TanStack Query wrappers + utility hooks
├── contexts/        # CartContext, PartnerContext
├── utils/           # Pure utilities, API call helpers, query key factories
├── models/          # Zod schemas for runtime validation
├── types/           # TypeScript interfaces / type aliases
├── styles/          # CSS modules (mirroring components/ structure)
└── tests/           # Integration and component tests
```

---

## B. Code Patterns

### B.1 Page Component Pattern

```tsx
export default function ExamplePage() {
  const router = useRouter();
  const { paramId } = router.query;
  const paramIdString = typeof paramId === 'string' ? paramId : paramId?.[0] || '';

  const { data, isLoading, error } = useQuery({
    queryKey: ['example', paramIdString],
    queryFn: async () => {
      const response = await fetch(`/api/example?id=${paramIdString}`);
      if (!response.ok) throw new Error('Failed to fetch');
      return response.json();
    },
    enabled: !!paramIdString && router.isReady,
    staleTime: 5 * 60 * 1000,
  });

  return (
    <Layout activePage="example">
      <NavigationPanel items={[{ label: 'Parent', href: '/parent' }]} />
      {isLoading ? (
        <LoadingSkeleton count={20} />
      ) : error ? (
        <div>Error: {error.message}</div>
      ) : (
        <div>{/* render data */}</div>
      )}
    </Layout>
  );
}
```

### B.2 Hook Pattern (TanStack Query — paginated list)

```ts
// hooks/useExampleList.ts
export function useExampleList({ page, limit, search }: Params) {
  const isSearchMode = search.length > 0;

  return useQuery({
    queryKey: isSearchMode
      ? getExampleSearchKey(search, page, limit)
      : getExampleListKey(page, limit),
    queryFn: () => fetchExampleList({ page, limit, search }),
    staleTime: 5 * 60 * 1000,
    gcTime: 10 * 60 * 1000,
    placeholderData: keepPreviousData,  // avoids flash on page change
  });
}
```

### B.3 Mutation Pattern

```tsx
// In a component:
const createOrder = useCreateOrder();

const handleSubmit = async () => {
  try {
    const result = await createOrder.mutateAsync(payload);
    router.push({ pathname: '/orderConfirmation', query: { orderId: result.orderId } });
  } catch (error) {
    setErrorMessage(error.message);
    setShowErrorToast(true);
  }
};
```

### B.4 Context Consumer Pattern

```tsx
// Read from CartContext
const { cartItemIdToQuantityMap, addToCart, getTotalItems } = useCart();

// Read from PartnerContext
const { availableCurrencies, region, regionCurrencies } = usePartnerDetails();
```

### B.5 Design System — @react-spectrum/s2

**Import pattern:**
```tsx
import { Button, SearchField, Heading, Card, Badge, TextField, NumberField, Switch } from '@react-spectrum/s2';
import ShoppingCart from '@react-spectrum/s2/icons/ShoppingCart';
import Close from '@react-spectrum/s2/icons/Close';
```

**Commonly used components:**

| Component | Use case |
|-----------|---------|
| `Button` | All interactive buttons (`variant: "primary" | "secondary"`, `fillStyle: "fill" | "outline"`) |
| `SearchField` | Search inputs (use `{...({ placeholder: '...' } as any)}` for TS workaround) |
| `Heading` | Page headings (`level={1..6}`) |
| `Card` | Content cards with hover state |
| `Badge` | Status / promo code labels (`fillStyle: "bold"`, `variant: "positive" | "neutral" | "negative"`) |
| `TextField` | Text inputs (Spectrum-styled) |
| `NumberField` | Quantity inputs |
| `Switch` | Toggle (auto-renewal, 3YC enrollment) |
| `ProgressCircle` | Loading spinners (inline) |
| `Tabs` / `TabList` / `Tab` / `TabPanel` | Tab navigation |
| `ActionButton` | Icon-only buttons |

**a11y:** Spectrum handles ARIA roles, keyboard navigation, and focus management for all interactive components automatically. Manual `aria-label` additions are required for icon-only buttons and non-descriptive actions.

**Provider requirement:** All Spectrum components must be rendered inside `<Provider colorScheme="light">` (configured in `pages/_app.tsx`).

**TypeScript note:** Some Spectrum S2 components have incomplete typings. Common workaround:
```tsx
<SearchField {...({ placeholder: 'Search...' } as any)} />
```

### B.6 Form Patterns

**Bridge forms do not use `react-hook-form`** (package present in deps but not used in component code). Forms use plain React `useState` + manual validation.

**Standard input dialog pattern (PromoCodeDialog):**
```tsx
const [inputValue, setInputValue] = useState('');
const [error, setError] = useState<string | null>(null);

const handleApply = () => {
  const trimmed = inputValue.trim();
  if (!trimmed) {
    setError('Please enter a value');
    return;
  }
  setError(null);
  onApply(trimmed);
};

// Reset on open
useEffect(() => {
  if (isOpen) {
    setInputValue(currentValue || '');
    setError(null);
  }
}, [isOpen, currentValue]);
```

**Dialog overlay pattern:** Custom dialogs use `createPortal` to render at document body. Backdrop click → close. `role="dialog"` + `aria-modal="true"` + `aria-labelledby`.

### B.7 Error Handling in Components

**Toast pattern (per-component — not global):**
```tsx
const [showErrorToast, setShowErrorToast] = useState(false);
const [errorMessage, setErrorMessage] = useState('');

// Show:
setErrorMessage('Something went wrong');
setShowErrorToast(true);

// In JSX:
{showErrorToast && (
  <ErrorToast message={errorMessage} onClose={() => setShowErrorToast(false)} />
)}
```

**Multiple toast types (useToastState hook):**
```tsx
const { toastState, showError, showWarning, showSuccess, clearError } = useToastState();
```

**Inline error state:**
```tsx
{error ? (
  <div>
    <h3>Unable to load data</h3>
    <p>{error.message}</p>
    <Button onPress={() => queryClient.invalidateQueries({ queryKey: [...] })}>Try Again</Button>
  </div>
) : null}
```

No global `ErrorBoundary` — errors are handled per-component or per-hook.

### B.8 Test Patterns

**Framework:** Jest 30 + @testing-library/react 16

**Integration tests (backend controllers):**
```ts
// tests/createorder.integration.test.ts
import { createNewOrder } from '../controllers/orderController';

jest.mock('../utils/logger', () => {
  const noopLogger = { info: jest.fn(), error: jest.fn(), warn: jest.fn() };
  return {
    createControllerLogger: () => noopLogger,
    logRequest: jest.fn(), logResponse: jest.fn(), logErrorResponse: jest.fn(),
    logger: { child: jest.fn().mockReturnValue(noopLogger) },
    default: { child: jest.fn().mockReturnValue(noopLogger) },
  };
});

// Environment isolation:
let savedEnv: NodeJS.ProcessEnv;
beforeEach(() => {
  savedEnv = { ...process.env };
  process.env.PARTNER_API_BASE_URL = 'https://api.test';
  (global.fetch as jest.Mock).mockClear();
});
afterEach(() => { process.env = savedEnv; });

// Mock fetch:
(global.fetch as jest.Mock).mockResolvedValueOnce({
  ok: true, status: 200,
  text: async () => JSON.stringify(responseFixture),
  headers: { get: () => null },
});
```

**Component tests:**
```tsx
// tests/pricelistCurrencyGuard.test.tsx
jest.mock('../contexts/PartnerContext', () => ({ usePartnerDetails: jest.fn() }));
jest.mock('../contexts/CartContext', () => ({
  useCart: jest.fn().mockReturnValue({ /* cart mock */ }),
}));
jest.mock('@tanstack/react-query', () => ({
  useInfiniteQuery: jest.fn().mockReturnValue({ data: null, isLoading: false, ... }),
  useQueryClient: jest.fn().mockReturnValue({ prefetchInfiniteQuery: jest.fn(), ... }),
}));
// Replace heavy components with null stubs:
jest.mock('../components/NavigationPanel', () => ({ __esModule: true, default: () => null }));

// Render and interact:
render(<CatalogPage />);
fireEvent.change(screen.getByTestId('currency-select'), { target: { value: 'EUR' } });
expect(prefetchAllMarketSegments).toHaveBeenCalledWith(...);
```

**Key test conventions:**
- Mock all external hooks and contexts at module level with `jest.mock`
- Mock heavy child components with `() => null` stubs
- Use `global.fetch` mock for HTTP calls (not `msw` or `nock`)
- `savedEnv` pattern for environment isolation
- No React Query `QueryClientProvider` wrapper needed — TanStack Query is mocked at module level in component tests
