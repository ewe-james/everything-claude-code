---
name: framer-motion-patterns
description: Framer Motion animation patterns — enter/exit transitions, list animations, layout animations, page transitions, and reusable motion variants.
origin: EWE
---

# Framer Motion Patterns

Animation patterns for React applications using Framer Motion.

## When to Activate

- Adding enter/exit animations to components
- Animating list additions and removals
- Implementing layout animations (shared layout, reordering)
- Building page transitions in TanStack Router or Next.js
- Creating reusable motion variants

## Related Skills

- **frontend-patterns** — Component composition, performance optimization

---

## Core Concepts

### Animate + AnimatePresence

`AnimatePresence` enables exit animations when components unmount.

```tsx
import { motion, AnimatePresence } from 'framer-motion'

export function Toast({ message, visible }: { message: string; visible: boolean }) {
  return (
    <AnimatePresence>
      {visible && (
        <motion.div
          initial={{ opacity: 0, y: 20 }}
          animate={{ opacity: 1, y: 0 }}
          exit={{ opacity: 0, y: 20 }}
          transition={{ duration: 0.2 }}
          className="toast"
        >
          {message}
        </motion.div>
      )}
    </AnimatePresence>
  )
}
```

### Reusable Variants

Define animation states as objects, reuse across components:

```tsx
const fadeSlideUp = {
  initial: { opacity: 0, y: 20 },
  animate: { opacity: 1, y: 0 },
  exit:    { opacity: 0, y: -10 },
}

// Usage
<motion.div {...fadeSlideUp} transition={{ duration: 0.3 }}>
  {children}
</motion.div>
```

---

## List Animations

Use `key` on each `motion` element so AnimatePresence tracks additions/removals:

```tsx
export function AnimatedList({ items }: { items: TItem[] }) {
  return (
    <AnimatePresence initial={false}>
      {items.map(item => (
        <motion.div
          key={item.id}
          initial={{ opacity: 0, height: 0 }}
          animate={{ opacity: 1, height: 'auto' }}
          exit={{ opacity: 0, height: 0 }}
          transition={{ duration: 0.2 }}
        >
          <ItemCard item={item} />
        </motion.div>
      ))}
    </AnimatePresence>
  )
}
```

### Staggered Children

```tsx
const container = {
  animate: { transition: { staggerChildren: 0.05 } },
}

const child = {
  initial: { opacity: 0, y: 10 },
  animate: { opacity: 1, y: 0 },
}

export function StaggeredGrid({ items }: { items: TItem[] }) {
  return (
    <motion.div variants={container} initial="initial" animate="animate" className="grid grid-cols-3 gap-4">
      {items.map(item => (
        <motion.div key={item.id} variants={child}>
          <Card item={item} />
        </motion.div>
      ))}
    </motion.div>
  )
}
```

---

## Modal / Overlay Animation

```tsx
export function Modal({ isOpen, onClose, children }: IModalProps) {
  return (
    <AnimatePresence>
      {isOpen && (
        <>
          <motion.div
            className="fixed inset-0 bg-black/50"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            onClick={onClose}
          />
          <motion.div
            className="fixed inset-0 flex items-center justify-center"
            initial={{ opacity: 0, scale: 0.95 }}
            animate={{ opacity: 1, scale: 1 }}
            exit={{ opacity: 0, scale: 0.95 }}
            transition={{ type: 'spring', damping: 25, stiffness: 300 }}
          >
            {children}
          </motion.div>
        </>
      )}
    </AnimatePresence>
  )
}
```

---

## Layout Animations

### Auto-Layout with `layout`

Adding `layout` makes elements animate smoothly when their position or size changes:

```tsx
export function FilterableList({ items, filter }: { items: TItem[]; filter: string }) {
  const filtered = items.filter(i => i.category === filter || filter === 'all')

  return (
    <div className="flex flex-wrap gap-4">
      <AnimatePresence>
        {filtered.map(item => (
          <motion.div
            key={item.id}
            layout
            initial={{ opacity: 0, scale: 0.8 }}
            animate={{ opacity: 1, scale: 1 }}
            exit={{ opacity: 0, scale: 0.8 }}
            transition={{ layout: { type: 'spring', damping: 20 } }}
          >
            <Card item={item} />
          </motion.div>
        ))}
      </AnimatePresence>
    </div>
  )
}
```

### Shared Layout Animation (LayoutGroup)

Animate an element between two positions (e.g. active tab indicator):

```tsx
import { LayoutGroup } from 'framer-motion'

export function TabBar({ tabs, activeTab, onSelect }: ITabBarProps) {
  return (
    <LayoutGroup>
      <div className="flex gap-2">
        {tabs.map(tab => (
          <button key={tab.id} onClick={() => onSelect(tab.id)} className="relative px-4 py-2">
            {tab.label}
            {activeTab === tab.id && (
              <motion.div
                layoutId="active-tab"
                className="absolute inset-0 rounded-md bg-primary/10"
                transition={{ type: 'spring', damping: 25, stiffness: 300 }}
              />
            )}
          </button>
        ))}
      </div>
    </LayoutGroup>
  )
}
```

---

## Page Transitions

### With TanStack Router

```tsx
// components/AnimatedOutlet.tsx
import { useRouterState, Outlet } from '@tanstack/react-router'

export function AnimatedOutlet() {
  const { location } = useRouterState()

  return (
    <AnimatePresence mode="wait">
      <motion.div
        key={location.pathname}
        initial={{ opacity: 0, x: 20 }}
        animate={{ opacity: 1, x: 0 }}
        exit={{ opacity: 0, x: -20 }}
        transition={{ duration: 0.2 }}
      >
        <Outlet />
      </motion.div>
    </AnimatePresence>
  )
}
```

### With Next.js App Router

```tsx
// app/template.tsx — runs on every navigation
'use client'

export default function Template({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
      transition={{ duration: 0.2 }}
    >
      {children}
    </motion.div>
  )
}
```

Note: Next.js App Router doesn't support exit animations natively since the outgoing page unmounts immediately. Use `template.tsx` for enter-only transitions.

---

## Performance Tips

- Use `layout` sparingly — measuring layout is expensive on large DOM trees
- Prefer `opacity` and `transform` (x, y, scale, rotate) — these are GPU-composited and don't trigger reflow
- Set `initial={false}` on `AnimatePresence` to skip mount animation on first render
- Use `will-change: transform` via Tailwind `will-change-transform` for heavy animations
