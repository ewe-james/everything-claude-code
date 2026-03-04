---
description: "TypeScript patterns extending common rules"
globs: ["**/*.ts", "**/*.tsx", "**/*.js", "**/*.jsx"]
alwaysApply: false
---
# TypeScript/JavaScript Patterns

> This file extends the common patterns rule with TypeScript/JavaScript specific content.

## Project Structure

### For frontend project
src/
├── app/                    # Next.js app router pages (if using Next.js)
├── routes/                 # TanStack Start and TanStack Router router pages (if using TanStack Start or TanStack Router)
├── features/               # Feature-based modules
├── components/             # Shared/common components
├── hooks/                  # Shared custom hooks
├── lib/                    # Third-party library configurations
├── stores/                 # Global state management
├── types/                  # Shared TypeScript types
├── utils/                  # Utility functions
└── constants/              # Application constants

- Feature-local vs global: Each feature may have its own components, hooks, stores, types, utils, and constants. If a given module is used by more than one feature, move it to the global components, hooks, stores, types, utils, or constants (as appropriate).
- Unit tests for functions and hooks: Use TDD for single functions and hooks. Add unit tests in a __tests__ folder next to the function or hook file. Name the test file after the function or hook, e.g. formatNumber.spec.ts.
- Testable core logic: Keep core logic (especially money/numeric calculations) in testable functions or hooks so it can be covered by unit tests.

### For backend project
src/
├── modules/
│   ├── rewards/
│   │   ├── rewards.module.ts
│   │   ├── rewards.controller.ts       (GET/POST /rewards/*)
│   │   ├── rewards.service.ts
│   │   ├── dto/
│   │   │   └── dispatch-event.dto.ts
│   │   └── repositories/
│   │       ├── user-rewards.repository.ts
├── db/
│   ├── drizzle.module.ts
│   ├── drizzle.provider.ts
│   └── schema/
│       ├── index.ts
│       ├── profiles.schema.ts
├── blockchain/
│   ├── blockchain.module.ts
│   └── blockchain.client.ts
├── common/                    # Shared across modules
│   ├── filters/               # Exception filters (e.g. GrpcExceptionFilter)
│   ├── guards/                # Auth/authorization guards
│   ├── interceptors/          # Logging, metrics, auth
│   ├── pipes/                 # Validation pipes
│   └── decorators/
├── proto/                     # gRPC（if use）
│   └── market.proto
└── types/ 
├── app.module.ts
└── main.ts
drizzle/
└── migrations/
    ├── 0001_schema.sql
    └── 0002_seed.sql
drizzle.config.ts

## API Response Format

```typescript
interface ApiResponse<T> {
  success: boolean
  data?: T
  error?: string
  meta?: {
    total: number
    page: number
    limit: number
  }
}
```

## Custom Hooks Pattern

```typescript
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay)
    return () => clearTimeout(handler)
  }, [value, delay])

  return debouncedValue
}
```

## Repository Pattern

```typescript
interface Repository<T> {
  findAll(filters?: Filters): Promise<T[]>
  findById(id: string): Promise<T | null>
  create(data: CreateDto): Promise<T>
  update(id: string, data: UpdateDto): Promise<T>
  delete(id: string): Promise<void>
}
```
