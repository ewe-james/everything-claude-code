---
name: zustand-patterns
description: Zustand state management patterns — store creation, slices, middleware (persist, devtools, immer), and Context+Reducer for scoped state.
origin: EWE
---

# Zustand Patterns

Client-side state management with Zustand for React applications.

## When to Activate

- Creating or modifying a Zustand store
- Choosing between Zustand, Context, or useState for state management
- Adding persist, devtools, or immer middleware to a store
- Splitting a large store into slices
- Implementing scoped state with Context + useReducer

## Related Skills

- **frontend-patterns** — Component composition, TanStack Query (server state), hooks

---

## Store Creation

```tsx
import { create } from 'zustand'

interface IAuthStore {
  user: TUser | null
  setUser: (user: TUser) => void
  logout: () => void
}

export const useAuthStore = create<IAuthStore>((set) => ({
  user: null,
  setUser: (user) => set({ user }),
  logout: () => set({ user: null }),
}))
```

Usage — select only what you need to avoid unnecessary re-renders:

```tsx
// ✅ Select specific field
const user = useAuthStore((s) => s.user)

// ❌ Avoid — subscribes to entire store
const { user, setUser } = useAuthStore()
```

---

## Slice Pattern (Large Stores)

Split a store into domain slices, then combine:

```tsx
// slices/authSlice.ts
export interface IAuthSlice {
  user: TUser | null
  setUser: (user: TUser) => void
  logout: () => void
}

export const createAuthSlice: StateCreator<TAppStore, [], [], IAuthSlice> = (set) => ({
  user: null,
  setUser: (user) => set({ user }),
  logout: () => set({ user: null }),
})

// slices/uiSlice.ts
export interface IUISlice {
  sidebarOpen: boolean
  toggleSidebar: () => void
}

export const createUISlice: StateCreator<TAppStore, [], [], IUISlice> = (set) => ({
  sidebarOpen: true,
  toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
})

// store.ts
type TAppStore = IAuthSlice & IUISlice

export const useAppStore = create<TAppStore>()((...args) => ({
  ...createAuthSlice(...args),
  ...createUISlice(...args),
}))
```

---

## Middleware

### Persist (localStorage / sessionStorage)

```tsx
import { persist } from 'zustand/middleware'

export const useSettingsStore = create<ISettingsStore>()(
  persist(
    (set) => ({
      theme: 'light' as 'light' | 'dark',
      locale: 'en',
      setTheme: (theme) => set({ theme }),
      setLocale: (locale) => set({ locale }),
    }),
    {
      name: 'settings-storage',       // localStorage key
      partialize: (s) => ({ theme: s.theme, locale: s.locale }), // only persist these fields
    },
  ),
)
```

### Devtools

```tsx
import { devtools } from 'zustand/middleware'

export const useMarketStore = create<IMarketStore>()(
  devtools(
    (set) => ({
      markets: [],
      setMarkets: (markets) => set({ markets }, false, 'setMarkets'),
    }),
    { name: 'MarketStore' },
  ),
)
```

### Combining Middleware

```tsx
export const useStore = create<IStore>()(
  devtools(
    persist(
      (set) => ({ /* ... */ }),
      { name: 'app-storage' },
    ),
    { name: 'AppStore' },
  ),
)
```

---

## Context + Reducer (Scoped State)

Use when state is scoped to a component subtree rather than global:

```tsx
interface IState {
  markets: TMarket[]
  selectedMarket: TMarket | null
}

type TAction =
  | { type: 'SET_MARKETS'; payload: TMarket[] }
  | { type: 'SELECT_MARKET'; payload: TMarket }

function reducer(state: IState, action: TAction): IState {
  switch (action.type) {
    case 'SET_MARKETS':
      return { ...state, markets: action.payload }
    case 'SELECT_MARKET':
      return { ...state, selectedMarket: action.payload }
    default:
      return state
  }
}

const MarketContext = createContext<{
  state: IState
  dispatch: Dispatch<TAction>
} | undefined>(undefined)

export function MarketProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(reducer, { markets: [], selectedMarket: null })
  return (
    <MarketContext.Provider value={{ state, dispatch }}>
      {children}
    </MarketContext.Provider>
  )
}

export function useMarkets() {
  const ctx = useContext(MarketContext)
  if (!ctx) throw new Error('useMarkets must be used within MarketProvider')
  return ctx
}
```

---

## When to Use What

| Scenario | Solution |
|----------|----------|
| Global client state (auth, theme, UI flags) | **Zustand** |
| Server/async data (API responses) | **TanStack Query** (see frontend-patterns) |
| Scoped state shared within a component subtree | **Context + useReducer** |
| Local component state | **useState** |
| Complex form state | **React Hook Form** (see shadcn skill) |
