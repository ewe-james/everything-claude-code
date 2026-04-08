---
name: vitest-patterns
description: Testing patterns for React apps — Vitest unit tests, React Testing Library component tests, mock strategies, coverage configuration, and Playwright E2E tests.
origin: EWE
---

# Vitest & Testing Patterns

Unit, component, and E2E testing patterns for React + TypeScript projects.

## When to Activate

- Writing unit tests for utility functions or hooks
- Writing component tests with React Testing Library
- Setting up Vitest configuration or coverage
- Mocking modules, API calls, or timers
- Writing Playwright E2E tests for critical user flows

## Related Skills

- **frontend-patterns** — Component patterns being tested
- **shadcn** — Testing form integrations

---

## Vitest Configuration

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      thresholds: { lines: 80, branches: 80, functions: 80 },
      exclude: ['**/*.d.ts', '**/*.config.*', '**/test/**'],
    },
  },
})
```

```typescript
// src/test/setup.ts
import '@testing-library/jest-dom/vitest'
```

---

## Unit Tests (Functions & Hooks)

### Pure Functions

```typescript
import { describe, it, expect } from 'vitest'
import { formatCurrency } from '@/utils/formatCurrency'

describe('formatCurrency', () => {
  it('formats positive values with 2 decimals', () => {
    expect(formatCurrency(1234.5)).toBe('$1,234.50')
  })

  it('returns $0.00 for zero', () => {
    expect(formatCurrency(0)).toBe('$0.00')
  })

  it('handles negative values', () => {
    expect(formatCurrency(-99.9)).toBe('-$99.90')
  })
})
```

### Custom Hooks

```typescript
import { renderHook, act } from '@testing-library/react'
import { useDebounce } from '@/hooks/useDebounce'

describe('useDebounce', () => {
  beforeEach(() => { vi.useFakeTimers() })
  afterEach(() => { vi.useRealTimers() })

  it('returns initial value immediately', () => {
    const { result } = renderHook(() => useDebounce('hello', 300))
    expect(result.current).toBe('hello')
  })

  it('updates value after delay', () => {
    const { result, rerender } = renderHook(
      ({ value }) => useDebounce(value, 300),
      { initialProps: { value: 'hello' } },
    )

    rerender({ value: 'world' })
    expect(result.current).toBe('hello')

    act(() => { vi.advanceTimersByTime(300) })
    expect(result.current).toBe('world')
  })
})
```

---

## Component Tests (React Testing Library)

### Basic Component

```tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { Button } from '@/components/Button'

describe('Button', () => {
  it('renders label and calls onClick', async () => {
    const onClick = vi.fn()
    render(<Button onClick={onClick}>Submit</Button>)

    await userEvent.click(screen.getByRole('button', { name: 'Submit' }))
    expect(onClick).toHaveBeenCalledOnce()
  })

  it('is disabled when loading', () => {
    render(<Button loading>Submit</Button>)
    expect(screen.getByRole('button')).toBeDisabled()
  })
})
```

### Components with TanStack Query

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'

function createWrapper() {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  })
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  )
}

describe('MarketList', () => {
  it('shows markets after loading', async () => {
    vi.spyOn(marketApi, 'getMarkets').mockResolvedValue([
      { id: '1', name: 'Test Market', volume: 100 },
    ])

    render(<MarketList />, { wrapper: createWrapper() })

    expect(screen.getByText('Loading...')).toBeInTheDocument()
    expect(await screen.findByText('Test Market')).toBeInTheDocument()
  })
})
```

---

## Mock Strategies

### Module Mock

```typescript
vi.mock('@/api/marketApi', () => ({
  getMarkets: vi.fn(),
  postMarket: vi.fn(),
}))
```

### Partial Mock (keep other exports)

```typescript
vi.mock('@/utils/analytics', async (importOriginal) => {
  const actual = await importOriginal<typeof import('@/utils/analytics')>()
  return { ...actual, trackEvent: vi.fn() }
})
```

### API / fetch Mock

```typescript
const mockFetch = vi.fn()
global.fetch = mockFetch

beforeEach(() => {
  mockFetch.mockResolvedValue({
    ok: true,
    json: () => Promise.resolve({ data: [] }),
  })
})
```

### Timer Mock

```typescript
beforeEach(() => { vi.useFakeTimers() })
afterEach(() => { vi.useRealTimers() })

it('debounces input', async () => {
  // ... trigger input
  vi.advanceTimersByTime(500)
  // ... assert debounced result
})
```

---

## Playwright E2E Tests

```typescript
import { test, expect } from '@playwright/test'

test('user can login and see dashboard', async ({ page }) => {
  await page.goto('/login')
  await page.fill('[name="email"]', 'user@example.com')
  await page.fill('[name="password"]', 'password123')
  await page.click('button[type="submit"]')

  await expect(page).toHaveURL('/dashboard')
  await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible()
})

test('search filters market list', async ({ page }) => {
  await page.goto('/markets')
  await page.fill('[placeholder="Search markets"]', 'Bitcoin')

  const rows = page.getByRole('row')
  await expect(rows).toHaveCount(2) // header + 1 result
  await expect(rows.nth(1)).toContainText('Bitcoin')
})
```

---

## Testing Checklist

- [ ] Pure functions: edge cases, boundary values, error paths
- [ ] Hooks: initial state, state transitions, cleanup
- [ ] Components: render, user interaction, loading/error states
- [ ] API integration: success, error, loading states (mock fetch or msw)
- [ ] Coverage ≥ 80% lines/branches/functions
