---
id: "frontend/nextjs-hydration-mismatch"
type: "gotcha"
title: "Next.js hydration mismatch when rendering date/time or browser-only values on server"
domain: "frontend"
tags: ["nextjs", "react", "hydration", "ssr", "date", "browser", "window"]
stack: ["next@>=13", "react@>=18"]
severity: "hard-error"
score: 50
uses: 0
votes: {up: 0, down: 0}
added: "2026-04-13"
updated: "2026-04-13"
fresh: true
ttl_days: 730
supersedes: null
superseded_by: null
related: []
---

# Next.js hydration mismatch when rendering date/time or browser-only values on server

## Symptom
```
Error: Hydration failed because the initial UI does not match what was rendered on the server.
```
Appears on pages that render `new Date()`, `Date.now()`, `Math.random()`, `window.*`, `localStorage.*`, or `navigator.*`. Works in dev sometimes (React suppresses in strict mode) then fails in production or after switching between pages.

## Cause
Next.js renders components on the server first, then re-renders on the client ("hydration"). If any rendered output differs between the two passes — a timestamp, a random ID, a window property that doesn't exist server-side — React throws and the page is broken.

## Fix

**For date/time values — use `useEffect` to set client-only state:**
```tsx
function LiveClock() {
  const [time, setTime] = useState<string | null>(null)

  useEffect(() => {
    // Only runs on client, after first server-matched render
    setTime(new Date().toLocaleTimeString())
    const id = setInterval(() => setTime(new Date().toLocaleTimeString()), 1000)
    return () => clearInterval(id)
  }, [])

  if (!time) return <span>--:--:--</span>   // matches server output
  return <span>{time}</span>
}
```

**For browser-only APIs — use `suppressHydrationWarning` or dynamic import with `ssr: false`:**
```tsx
// components/ClientOnly.tsx
export function ClientOnly({ children }: { children: React.ReactNode }) {
  const [mounted, setMounted] = useState(false)
  useEffect(() => setMounted(true), [])
  if (!mounted) return null
  return <>{children}</>
}

// Usage
<ClientOnly>
  <ThemeToggle />  {/* reads localStorage */}
</ClientOnly>
```

Or for a whole component:
```tsx
// Dynamic import disables SSR for this component
const Map = dynamic(() => import('./Map'), { ssr: false })
```

## Never Do
```tsx
// Renders different string on server vs client → hydration error
function Header() {
  return <div>Last seen: {new Date().toLocaleString()}</div>
}

// window doesn't exist on server → crashes during SSR, not just hydration
function Theme() {
  const theme = window.localStorage.getItem('theme') ?? 'light'
  return <div className={theme}>...</div>
}
```

## Scope
Affects all Next.js rendering modes (Pages Router and App Router) on any page that isn't `'use client'`-only with `ssr: false`. In App Router, `'use client'` directive still runs on the server for initial HTML — the bug persists unless you add the `ClientOnly` wrapper or `dynamic(..., { ssr: false })`.
