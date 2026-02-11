---
name: tauri-v2-react-frontend
description: React frontend patterns for feature-based Tauri v2 apps. Covers type-safe command integration via tauri-specta, TanStack Query data layer, Zustand state management, hook orchestration, and component patterns. Use when implementing React features that coordinate with Rust backend via typed IPC contracts.
---

# Tauri v2 React Frontend Architecture

Frontend patterns for building React features in Tauri v2 desktop apps with type-safe IPC, proper state management, and enforced architectural patterns.

## When to Apply

- Implementing React features in `src/features/`
- Creating hooks that orchestrate Tauri commands
- Managing state with TanStack Query and Zustand
- Building components that consume typed backend data
- Working on UI while backend agent handles Rust side

## Critical Frontend Rules

### 1. Type-Safe Commands (Never Raw invoke)

All Rust commands are typed via tauri-specta. Import from generated bindings.

```typescript
// ❌ WRONG - String-based invoke
import { invoke } from '@tauri-apps/api/core'
const prefs = await invoke<AppPreferences>('load_preferences')

// ✅ RIGHT - Type-safe commands
import { commands } from '@/lib/tauri-bindings'
const result = await commands.loadPreferences()
if (result.status === 'ok') {
  console.log(result.data.theme) // Fully typed!
}
```

**What you get:**
- Compile-time type checking
- IDE autocomplete for all commands
- Refactoring safety (renames propagate)
- Auto-generated from Rust code

---

### 2. Service Layer (Plain Async Functions)

Services wrap Tauri commands. NO classes, just plain exported functions.

```typescript
// features/preferences/services/preferencesService.ts

import { commands, unwrapResult } from '@/lib/tauri-bindings'
import type { AppPreferences } from '@/lib/tauri-bindings'

// ✅ RIGHT - Plain async function
export async function loadPreferences(): Promise<AppPreferences> {
  return unwrapResult(await commands.loadPreferences())
}

export async function savePreferences(prefs: AppPreferences): Promise<void> {
  unwrapResult(await commands.savePreferences(prefs))
}
```

**Service rules:**
- Live in `features/<n>/services/`
- Export plain async functions only
- ONLY call `commands.*` from tauri-bindings
- Handle Result types (see next section)
- No business logic (that's in Rust or hooks)

---

### 3. Result Type Handling

Rust commands return `Result<T, E>`. tauri-specta types this as:

```typescript
type Result<T, E> = 
  | { status: 'ok'; data: T }
  | { status: 'error'; error: E }
```

**Pattern 1: Manual handling (for UI feedback)**

```typescript
const handleSave = async () => {
  const result = await commands.savePreferences(preferences)
  
  if (result.status === 'error') {
    toast.error('Failed to save', { description: result.error })
    return
  }
  
  toast.success('Preferences saved!')
}
```

**Pattern 2: unwrapResult (for data fetching)**

```typescript
import { unwrapResult } from '@/lib/tauri-bindings'

// In TanStack Query - throws on error
const { data } = useQuery({
  queryKey: ['preferences'],
  queryFn: async () => unwrapResult(await commands.loadPreferences()),
})
```

**When to use:**
- **Manual `if/else`**: Event handlers, mutations with toast
- **unwrapResult**: TanStack Query queryFn (error propagates)

---

### 4. Hook Layer (TanStack Query)

Hooks wrap services and provide caching, loading, error states.

```typescript
// features/preferences/hooks/usePreferences.ts

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { loadPreferences, savePreferences } from '../services/preferencesService'

export function usePreferences() {
  const queryClient = useQueryClient()
  
  const { data, isLoading, error } = useQuery({
    queryKey: ['preferences'],
    queryFn: loadPreferences,
    staleTime: 60000,
  })
  
  const saveMutation = useMutation({
    mutationFn: savePreferences,
    onSuccess: (_, variables) => {
      queryClient.setQueryData(['preferences'], variables)
    },
  })
  
  return {
    preferences: data,
    isLoading,
    savePreferences: saveMutation.mutate,
    isSaving: saveMutation.isPending,
  }
}
```

**Hook responsibilities:**
- Wrap service functions with TanStack Query
- Handle loading/error states automatically
- Implement optimistic updates
- Manage cache invalidation
- Return clean API for components

---

### 5. State Management Onion

**Three layers - choose based on data characteristics:**

```
┌─────────────────────────────────────┐
│ TanStack Query                      │  ← Persistent (backend)
│ - Templates, preferences            │
│ - Cached, refetchable              │
└─────────────────────────────────────┘
            ↓
┌─────────────────────────────────────┐
│ Zustand (Global)                    │  ← Transient UI
│ - Sidebar visibility                │
│ - Real-time progress               │
│ - High-frequency updates           │
└─────────────────────────────────────┘
            ↓
┌─────────────────────────────────────┐
│ useState                            │  ← Component-local
│ - Form inputs, toggles              │
│ - Single component only            │
└─────────────────────────────────────┘
```

**Decision tree:**
1. **Fetched from backend?** → TanStack Query
2. **Shared but not persisted?** → Zustand
3. **Single component only?** → useState

---

### 6. Zustand Selector Pattern (CRITICAL)

**NEVER destructure Zustand stores.** Always use selectors.

```typescript
// ❌ WRONG - Causes render cascades (caught by ast-grep)
const { leftSidebarVisible, setValue } = useUIStore()

// ✅ RIGHT - Selector syntax (only re-renders when value changes)
const leftSidebarVisible = useUIStore(state => state.leftSidebarVisible)
const setValue = useUIStore(state => state.setValue)

// ✅ RIGHT - Use getState() in callbacks
const handleToggle = () => {
  const { leftSidebarVisible, setLeftSidebarVisible } = useUIStore.getState()
  setLeftSidebarVisible(!leftSidebarVisible)
}
```

**Why:** Destructuring subscribes to ALL store changes. Selectors prevent unnecessary re-renders.

**Enforcement:** `no-zustand-destructuring` ast-grep rule catches violations.

---

### 7. Feature Isolation (Barrel Exports)

Features export via `index.ts`. Outside code NEVER imports internal paths.

```typescript
// features/preferences/index.ts
export { PreferencesDialog } from './components/PreferencesDialog'
export { usePreferences } from './hooks/usePreferences'

// ✅ RIGHT - Import from barrel
import { PreferencesDialog } from '@/features/preferences'

// ❌ WRONG - Deep import (caught by ast-grep)
import { PreferencesDialog } from '@/features/preferences/components/PreferencesDialog'
```

**Feature structure:**
```
features/<feature-name>/
├── components/      # React components
├── hooks/           # TanStack Query hooks
├── services/        # IPC wrappers (plain functions)
├── store/           # Zustand (if needed)
├── types/           # Feature types
├── __tests__/       # Tests
└── index.ts         # PUBLIC API
```

---

### 8. Component Call Chain

Components → Hooks → Services → Commands. Never skip layers.

```typescript
// ❌ WRONG - Component calls command
function PreferencesDialog() {
  const handleSave = async () => {
    await commands.savePreferences(prefs) // NO!
  }
}

// ✅ RIGHT - Component → Hook → Service → Command
function PreferencesDialog() {
  const { savePreferences } = usePreferences()
  
  const handleSave = () => {
    savePreferences(prefs) // Hook handles everything
  }
}
```

**Enforcement:** `no-direct-ipc-in-components` ast-grep rule.

---

### 9. Event Listeners (Backend → Frontend)

React listens to events emitted by Rust backend.

```typescript
// features/processing/hooks/useProcessing.ts

import { TauriEvents } from '@/lib/services/tauriEvents'

export function useProcessing() {
  const setProgress = useProcessingStore(state => state.setProgress)
  
  useEffect(() => {
    const unlisten = TauriEvents.listen('operation:progress', (payload) => {
      setProgress(payload)
    })
    
    // CRITICAL: Always cleanup
    return () => { unlisten.then(fn => fn()) }
  }, [setProgress])
}
```

**Event patterns:**
- `TauriEvents.listen()` for Rust → React events
- `FeatureEvents.listen()` for React → React cross-feature events
- ALWAYS return cleanup function from useEffect
- Store unlisten function, call in cleanup

---

## Complete Feature Example

**Service layer:**
```typescript
// features/templates/services/templateService.ts
import { commands, unwrapResult } from '@/lib/tauri-bindings'

export async function listTemplates(): Promise<Template[]> {
  return unwrapResult(await commands.listTemplates())
}

export async function saveTemplate(template: Template): Promise<void> {
  unwrapResult(await commands.saveTemplate(template))
}
```

**Hook with optimistic updates:**
```typescript
// features/templates/hooks/useTemplates.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'

export function useTemplates() {
  const queryClient = useQueryClient()
  
  const { data: templates, isLoading } = useQuery({
    queryKey: ['templates'],
    queryFn: listTemplates,
  })
  
  const saveMutation = useMutation({
    mutationFn: saveTemplate,
    onMutate: async (newTemplate) => {
      await queryClient.cancelQueries({ queryKey: ['templates'] })
      const previous = queryClient.getQueryData(['templates'])
      
      // Optimistic update
      queryClient.setQueryData(['templates'], (old: Template[]) => 
        [...old, newTemplate]
      )
      
      return { previous }
    },
    onError: (err, vars, context) => {
      // Rollback on error
      queryClient.setQueryData(['templates'], context?.previous)
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['templates'] })
    },
  })
  
  return {
    templates,
    isLoading,
    saveTemplate: saveMutation.mutate,
  }
}
```

**Component:**
```typescript
// features/templates/components/TemplateList.tsx
export function TemplateList() {
  const { templates, isLoading, saveTemplate } = useTemplates()
  
  if (isLoading) return <Spinner />
  
  return (
    <div>
      {templates?.map(t => (
        <TemplateCard key={t.id} template={t} />
      ))}
      <Button onClick={() => saveTemplate(newTemplate)}>
        Save
      </Button>
    </div>
  )
}
```

---

## Common Frontend Mistakes

### ❌ Mistake 1: Raw invoke
```typescript
// WRONG
await invoke('my_command', { param: value })

// RIGHT
await commands.myCommand(value)
```

### ❌ Mistake 2: Zustand destructuring
```typescript
// WRONG (caught by ast-grep)
const { value } = useMyStore()

// RIGHT
const value = useMyStore(state => state.value)
```

### ❌ Mistake 3: Deep feature imports
```typescript
// WRONG (caught by ast-grep)
import { X } from '@/features/my-feature/components/X'

// RIGHT
import { X } from '@/features/my-feature'
```

### ❌ Mistake 4: Component calls command
```typescript
// WRONG (caught by ast-grep)
await commands.doSomething() // In component

// RIGHT
const { doSomething } = useMyFeature()
```

### ❌ Mistake 5: Missing event cleanup
```typescript
// WRONG - Memory leak
useEffect(() => {
  TauriEvents.listen('event', handler)
}, [])

// RIGHT
useEffect(() => {
  const unlisten = TauriEvents.listen('event', handler)
  return () => { unlisten.then(fn => fn()) }
}, [])
```

### ❌ Mistake 6: Backend data in Zustand
```typescript
// WRONG - Use TanStack Query for backend data
const data = useMyStore(state => state.backendData)

// RIGHT
const { data } = useQuery({ queryKey: ['data'], queryFn: loadData })
```

---

## Backend Coordination

**While you work on React features, here's what the backend agent is doing:**

### Backend Agent's Responsibilities

1. **Writing Rust commands** with `#[tauri::command]` + `#[specta::specta]`
2. **Defining error types** as discriminated unions
3. **Emitting events** for real-time updates to frontend
4. **Registering commands** in `bindings.rs`
5. **Running `cargo test export_bindings`** to regenerate TypeScript bindings

### Your Integration Points

**After backend agent regenerates bindings:**
- New commands appear in `src/lib/tauri-bindings.ts` (auto-generated)
- Import commands: `import { commands } from '@/lib/tauri-bindings'`
- Types are available: `import type { MyData } from '@/lib/tauri-bindings'`

**When you need a new command:**
- Specify the Rust signature in coordination with backend agent
- Wait for bindings regeneration
- Then implement your service/hook/component

**Event contracts:**
- Coordinate event names and payload types
- Backend emits: `app.emit("event-name", payload)`
- You listen: `TauriEvents.listen('event-name', handler)`

### Shared Architectural Principles

**Feature parity:** Frontend `src/features/my-feature/` mirrors backend `src-tauri/src/features/my_feature/`

**Type contracts:** All types flow from Rust → TypeScript via tauri-specta

**Result pattern:** Backend returns `Result<T, E>`, you handle both cases

**Event naming:** Use kebab-case with feature prefix: `processing:progress`, `template:saved`

**Parallel work:** You can work in parallel once IPC contract (command signatures) is agreed upon

---

## Verification Before Commit

```bash
npm run verify:architecture  # Feature structure, barrel exports
npm run ast:lint             # Pattern enforcement
npm run typecheck            # TypeScript compilation
npm run test:run             # Frontend tests
npm run check:all            # Full quality gate
```

**What gets verified:**
- ✅ Barrel exports used (no deep imports)
- ✅ No `invoke()` in components
- ✅ No Zustand destructuring
- ✅ TypeScript compiles
- ✅ Tests pass

---

## Key References

- **Full architecture:** `docs/developer/architecture-guide.md`
- **State patterns:** `docs/developer/state-management.md`
- **Example feature:** `src/features/example-feature/`
- **Quick ref:** `AGENTS.md`

---

## Frontend Implementation Checklist

- ✅ Commands from `@/lib/tauri-bindings` (not raw invoke)
- ✅ Services are plain async functions
- ✅ Hooks wrap services with TanStack Query
- ✅ Result types handled properly
- ✅ State in correct layer (useState/Zustand/TanStack Query)
- ✅ Zustand uses selectors (no destructuring)
- ✅ Feature exports via barrel
- ✅ Components use hooks (not services directly)
- ✅ Events cleaned up in useEffect
- ✅ Architecture verification passes
