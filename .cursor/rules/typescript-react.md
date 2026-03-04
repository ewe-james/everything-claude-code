---
description: "React best practices and patterns for TSX/JSX, extends TypeScript rules"
globs: ["**/*.tsx", "**/*.jsx", "components/**/*", "hooks/**/*"]
alwaysApply: false
---
# TypeScript/React Best Practices

> This file extends the TypeScript rules with React-specific patterns for components, hooks, state, and UX.

## Component Structure

- Use functional components over class components
- Keep components small and focused
- Extract reusable logic into custom hooks
- Use composition over inheritance
- Implement proper prop types with TypeScript
- Split large components into smaller, focused ones

```tsx
// CORRECT: Functional component, typed props
interface IButtonProps {
  label: string
  onClick: () => void
  variant?: 'primary' | 'secondary'
}

export const Button = ({ label, onClick, variant = 'primary' }: IButtonProps) => (
  <button type="button" className={cn('btn', `btn-${variant}`)} onClick={onClick}>
    {label}
  </button>
)
```

## Hooks

- Follow the Rules of Hooks
- Use custom hooks for reusable logic
- Keep hooks focused and simple
- Use appropriate dependency arrays in useEffect
- Implement cleanup in useEffect when needed
- Avoid nested hooks

```tsx
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay)
    return () => clearTimeout(handler)
  }, [value, delay])

  return debouncedValue
}
```

```tsx
// CORRECT: useEffect with cleanup
useEffect(() => {
  const sub = stream.subscribe(handleMessage)
  return () => sub.unsubscribe()
}, [stream])
```

## State Management

- Use useState for local component state
- Use state management library, Zustand for complex state logic and shared state
- Keep state as close to where it's used as possible
- Avoid prop drilling through proper state management

```tsx
const [isOpen, setIsOpen] = useState(false)
const [query, setQuery] = useState('')
```

```tsx
// Zustand store for shared/complex state
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

## Performance

- Implement proper memoization (useMemo, useCallback)
- Use React.memo for expensive components
- Avoid unnecessary re-renders
- Implement proper lazy loading
- Use proper key props in lists
- Profile and optimize render performance

```tsx
const MemoizedItem = React.memo(function Item({ id, name, onSelect }: IItemProps) {
  return <li onClick={() => onSelect(id)}>{name}</li>
})

const sortedList = useMemo(() => [...items].sort(byDate), [items])
const handleSubmit = useCallback(() => submit(formData), [formData])
```

## Forms

- Use controlled components for form inputs
- Use react-hook-form or tanstack form with zod to implement proper form validation
- Handle form submission states properly using react-hook-form or tanstack form
- Show appropriate loading and error states
- Implement proper accessibility for forms

```tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

type TFormValues = z.infer<typeof schema>

export const LoginForm = () => {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<TFormValues>({ resolver: zodResolver(schema) })

  const onSubmit = async (data: TFormValues) => {
    await submitLogin(data)
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        type="email"
        {...register('email')}
        aria-invalid={!!errors.email}
        aria-describedby={errors.email ? 'email-error' : undefined}
      />
      {errors.email && (
        <span id="email-error" role="alert">{errors.email.message}</span>
      )}
      <input
        type="password"
        {...register('password')}
        aria-invalid={!!errors.password}
        aria-describedby={errors.password ? 'password-error' : undefined}
      />
      {errors.password && (
        <span id="password-error" role="alert">{errors.password.message}</span>
      )}
      <button type="submit" disabled={isSubmitting}>Submit</button>
    </form>
  )
}
```

## Error Handling

- Implement Error Boundaries
- Handle async errors properly
- Show user-friendly error messages
- Implement proper fallback UI
- Log errors appropriately
- Handle edge cases gracefully

```tsx
interface IProps {
  children: React.ReactNode
  fallback?: React.ReactNode
}

export class ErrorBoundary extends React.Component<IProps, { hasError: boolean }> {
  state = { hasError: false }

  static getDerivedStateFromError() {
    return { hasError: true }
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error('ErrorBoundary caught:', error, info.componentStack)
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? <div role="alert">Something went wrong.</div>
    }
    return this.props.children
  }
}
```

## Testing

- Write unit tests for components
- Implement integration tests for complex flows
- Use React Testing Library
- Test user interactions
- Test error scenarios
- Implement proper mock data

```tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { Button } from './Button'

it('calls onClick when clicked', async () => {
  const onClick = vi.fn()
  render(<Button label="Submit" onClick={onClick} />)
  await userEvent.click(screen.getByRole('button', { name: 'Submit' }))
  expect(onClick).toHaveBeenCalledOnce()
})
```

## Accessibility

- Use semantic HTML elements
- Implement proper ARIA attributes
- Ensure keyboard navigation
- Test with screen readers
- Handle focus management
- Provide proper alt text for images

```tsx
<nav aria-label="Main">
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>

<button
  type="button"
  onClick={handleClose}
  onKeyDown={(e) => e.key === 'Escape' && handleClose()}
  aria-expanded={isOpen}
  aria-controls="menu-id"
>
  Menu
</button>

<img src={src} alt={alt} />
```

## Code Organization

- Group related components together
- Use proper file naming conventions
- Implement proper directory structure
- Keep styles close to components
- Use proper imports/exports
- Document complex component logic

```tsx
// components/UserAvatar.tsx — one main component per file, named export
interface IUserAvatarProps {
  src: string
  alt: string
  size?: 'sm' | 'md' | 'lg'
}

export const UserAvatar = ({ src, alt, size = 'md' }: IUserAvatarProps) => (
  <img src={src} alt={alt} className={cn('avatar', `avatar-${size}`)} />
)
```

## Lists and Keys

Use stable, unique keys (e.g. id); avoid index as key when list can reorder.

```tsx
{items.map((item) => (
  <MemoizedItem key={item.id} id={item.id} name={item.name} onSelect={onSelect} />
))}
```