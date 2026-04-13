---
id: "testing/vitest-mock-module-hoisting"
type: "gotcha"
title: "Vitest vi.mock() calls are hoisted to top of file — factory closures cannot reference local variables"
domain: "testing"
tags: ["vitest", "mock", "vi.mock", "hoisting", "jest", "module"]
stack: ["vitest@>=0.30"]
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
summary: "vi.mock() is hoisted before variable declarations — factory closures see TDZ, returning undefined. Fix: inline vi.fn() in factory, use vi.mocked() on the import, or use vi.hoisted() to declare variables that survive the hoisting transform."
---

# Vitest vi.mock() calls are hoisted to top of file — factory closures cannot reference local variables

## Symptom
```
ReferenceError: Cannot access 'mockFn' before initialization
```
Or the mock silently returns `undefined` instead of the expected value. Happens when the `vi.mock()` factory function references a `const`/`let` declared in the test file.

## Cause
Vitest (like Jest) hoists all `vi.mock()` calls to the **top of the file** before any imports — before `const`, `let`, or `describe` blocks are initialized. A variable declared at line 10 doesn't exist yet when the mock factory at line 2 (post-hoisting) executes.

```typescript
// What you write:
const myMock = vi.fn()
vi.mock('./service', () => ({ doWork: myMock }))

// What Vitest actually executes:
vi.mock('./service', () => ({ doWork: myMock }))  // hoisted — myMock is TDZ
const myMock = vi.fn()                             // runs after
```

## Fix

**Option A — Use `vi.fn()` inline in the factory (most common):**
```typescript
vi.mock('./service', () => ({
  doWork: vi.fn().mockResolvedValue({ ok: true })
}))

// Access via import in tests
import { doWork } from './service'

it('calls doWork', async () => {
  await runHandler()
  expect(doWork).toHaveBeenCalledOnce()
})
```

**Option B — Use `vi.mocked()` to type-cast the import:**
```typescript
vi.mock('./service')  // auto-mocks entire module

import { doWork } from './service'
const mockDoWork = vi.mocked(doWork)

beforeEach(() => {
  mockDoWork.mockResolvedValue({ ok: true })
})
```

**Option C — Use `vi.importMock()` for dynamic overrides:**
```typescript
it('handles error', async () => {
  const { doWork } = await vi.importMock<typeof import('./service')>('./service')
  vi.mocked(doWork).mockRejectedValue(new Error('fail'))

  await expect(runHandler()).rejects.toThrow('fail')
})
```

**Option D — `vi.hoisted()` to declare variables that survive hoisting:**
```typescript
const { myMock } = vi.hoisted(() => ({ myMock: vi.fn() }))

vi.mock('./service', () => ({ doWork: myMock }))

// myMock is now accessible in both the factory and test body
```

## Never Do
```typescript
const returnValue = { data: 'test' }  // declared normally

// Factory closes over returnValue — but it hasn't been initialized yet
vi.mock('./api', () => ({
  fetchData: vi.fn().mockResolvedValue(returnValue)  // undefined at hoisting time
}))
```

## Scope
Affects Vitest and Jest identically — both hoist `vi.mock()` / `jest.mock()` via Babel/SWC transform. The `vi.hoisted()` escape hatch is Vitest-specific (Vitest ≥ 0.31). Jest equivalent is putting the variable in a `let` and assigning in `beforeEach`.
