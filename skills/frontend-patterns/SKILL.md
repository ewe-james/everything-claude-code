---
name: frontend-patterns
description: Core frontend patterns for daily React development — TanStack Start/Router, Next.js App Router, TanStack Query, component composition, custom hooks, performance optimization, and Error Boundaries.
origin: EWE
---

# Frontend Development Patterns

Core patterns for daily React development with TanStack Start / Next.js.

## When to Activate

- Building React components with TypeScript
- Setting up TanStack Start routes or Next.js App Router pages
- Fetching server data with TanStack Query or creating TanStack Query custom hooks
- Optimizing render performance (memo, lazy, virtualization)
- Structuring component composition or custom hooks

## Related Skills

- **zustand-patterns** — Global client state, slices, middleware, persist
- **shadcn** — shadcn/ui components, cn(), Form integration, CLI workflow
- **vitest-patterns** — Vitest unit tests, React Testing Library, Playwright E2E
- **framer-motion-patterns** — Animations, AnimatePresence, layout transitions

## TypeScript Naming Conventions

| Type | Prefix | Example |
|------|--------|---------|
| `type` | `T` | `TUser`, `TApiResponse` |
| `interface` | `I` | `IButtonProps`, `IAuthStore` |
| `enum` | `E` | `EStatus`, `EUserRole` |

```tsx
// ✅ Correct naming
type TUser = { id: string; name: string }
interface IUserProps { user: TUser }
enum EUserRole { Admin = 'admin', User = 'user' }

// ❌ Avoid
type User = { id: string; name: string }
interface UserProps { user: User }
enum UserRole { Admin = 'admin', User = 'user' }
```

## Component Patterns

### Composition Over Inheritance

```typescript
// ✅ GOOD: Component composition
interface ICardProps {
  children: React.ReactNode
  variant?: 'default' | 'outlined'
}

export function Card({ children, variant = 'default' }: ICardProps) {
  return <div className={`card card-${variant}`}>{children}</div>
}

export function CardHeader({ children }: { children: React.ReactNode }) {
  return <div className="card-header">{children}</div>
}

export function CardBody({ children }: { children: React.ReactNode }) {
  return <div className="card-body">{children}</div>
}

// Usage
<Card>
  <CardHeader>Title</CardHeader>
  <CardBody>Content</CardBody>
</Card>
```

### Compound Components

```typescript
interface ITabsContextValue {
  activeTab: string
  setActiveTab: (tab: string) => void
}

const TabsContext = createContext<ITabsContextValue | undefined>(undefined)

export function Tabs({ children, defaultTab }: {
  children: React.ReactNode
  defaultTab: string
}) {
  const [activeTab, setActiveTab] = useState(defaultTab)

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      {children}
    </TabsContext.Provider>
  )
}

export function TabList({ children }: { children: React.ReactNode }) {
  return <div className="tab-list">{children}</div>
}

export function Tab({ id, children }: { id: string, children: React.ReactNode }) {
  const context = useContext(TabsContext)
  if (!context) throw new Error('Tab must be used within Tabs')

  return (
    <button
      className={context.activeTab === id ? 'active' : ''}
      onClick={() => context.setActiveTab(id)}
    >
      {children}
    </button>
  )
}

// Usage
<Tabs defaultTab="overview">
  <TabList>
    <Tab id="overview">Overview</Tab>
    <Tab id="details">Details</Tab>
  </TabList>
</Tabs>
```

## Custom Hooks

### Debounce Hook

```typescript
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value)
    }, delay)

    return () => clearTimeout(handler)
  }, [value, delay])

  return debouncedValue
}

// Usage
const [searchQuery, setSearchQuery] = useState('')
const debouncedQuery = useDebounce(searchQuery, 500)

useEffect(() => {
  if (debouncedQuery) {
    performSearch(debouncedQuery)
  }
}, [debouncedQuery])
```

## TanStack Query (Server State)

### Query Key Factory

```typescript
export const marketKeys = {
  all:    ['markets'] as const,
  list:   (filters: TFilters) => [...marketKeys.all, 'list', filters] as const,
  detail: (id: string) => [...marketKeys.all, 'detail', id] as const,
}
```

### Fetch + Select + Enabled

```typescript
const { data: activeMarkets = [] } = useQuery({
  queryKey: marketKeys.list({ status: 'active' }),
  queryFn: () => getMarkets({ status: 'active' }),
  select: (markets) => markets.filter(m => m.volume > 0),
  enabled: !!userId,
  staleTime: 5 * 60_000,
})
```

### Mutation + Invalidation

```typescript
const queryClient = useQueryClient()

const { mutateAsync: createMarket, isPending } = useMutation({
  mutationFn: postMarket,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: marketKeys.all })
  },
})
```

### Separate queryFn from Hook

```typescript
// api/marketApi.ts
export const getMarkets = async (filters: TFilters): Promise<TMarket[]> => {
  const res = await fetch(`/api/markets?${new URLSearchParams(filters)}`)
  return res.json()
}

// hooks/useMarkets.ts
export const useMarkets = (filters: TFilters) =>
  useQuery({
    queryKey: marketKeys.list(filters),
    queryFn: () => getMarkets(filters),
  })
```

## Performance Optimization

### Memoization

```typescript
// ✅ useMemo for expensive computations
const sortedMarkets = useMemo(() => {
  return [...markets].sort((a, b) => b.volume - a.volume)
}, [markets])

// ✅ useCallback for functions passed to children
const handleSearch = useCallback((query: string) => {
  setSearchQuery(query)
}, [])

// ✅ React.memo for pure components
export const MarketCard = React.memo<IMarketCardProps>(({ market }) => {
  return (
    <div className="market-card">
      <h3>{market.name}</h3>
      <p>{market.description}</p>
    </div>
  )
})
```

### Code Splitting & Lazy Loading

```typescript
import { lazy, Suspense } from 'react'

const HeavyChart = lazy(() => import('./HeavyChart'))
const ThreeJsBackground = lazy(() => import('./ThreeJsBackground'))

export function Dashboard() {
  return (
    <div>
      <Suspense fallback={<ChartSkeleton />}>
        <HeavyChart data={data} />
      </Suspense>

      <Suspense fallback={null}>
        <ThreeJsBackground />
      </Suspense>
    </div>
  )
}
```

### Virtualization for Long Lists

```typescript
import { useVirtualizer } from '@tanstack/react-virtual'

export function VirtualMarketList({ markets }: { markets: Market[] }) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: markets.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 100,
    overscan: 5
  })

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`
            }}
          >
            <MarketCard market={markets[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  )
}
```

## TanStack Start Patterns (Preferred Framework)

### Route Definition

```tsx
// app/routes/users.$userId.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/users/$userId')({
  loader: async ({ params }) => {
    return fetchUser(params.userId)
  },
  component: UserProfile,
})

function UserProfile() {
  const user = Route.useLoaderData()
  return <div>{user.name}</div>
}
```

### Server Functions

```tsx
import { createServerFn } from '@tanstack/start'

const getUsers = createServerFn('GET', async () => {
  return db.users.findMany()
})

const createUser = createServerFn('POST', async (data: TCreateUserInput) => {
  return db.users.create({ data })
})
```

## Next.js Patterns

### Server vs Client Components

```tsx
// Server Component (default in App Router) — direct data fetching, no hooks
async function UserProfile({ userId }: { userId: string }) {
  const user = await fetchUser(userId)
  return <div>{user.name}</div>
}

// Client Component — for interactivity or browser APIs
'use client'
export function InteractiveCounter() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

### Route Handlers

```tsx
// app/api/users/route.ts
import { NextResponse } from 'next/server'

export async function GET() {
  const users = await db.users.findMany()
  return NextResponse.json(users)
}

export async function POST(request: Request) {
  const body = await request.json()
  const user = await db.users.create({ data: body })
  return NextResponse.json(user, { status: 201 })
}
```

## Error Boundary Pattern

```typescript
interface IErrorBoundaryState {
  hasError: boolean
  error: Error | null
}

export class ErrorBoundary extends React.Component<
  { children: React.ReactNode },
  IErrorBoundaryState
> {
  state: IErrorBoundaryState = { hasError: false, error: null }

  static getDerivedStateFromError(error: Error): IErrorBoundaryState {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Error boundary caught:', error, errorInfo)
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-fallback">
          <h2>Something went wrong</h2>
          <p>{this.state.error?.message}</p>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      )
    }

    return this.props.children
  }
}
```

**Remember**: Prefer TanStack Start for new projects; use Next.js patterns when working in the App Router.