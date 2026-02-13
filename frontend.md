# Frontend Architecture Through Prompting

> Designing component systems, state management, and performant UIs with AI as your implementation partner.

← [Backend](./backend.md) | [Back to Index](./README.md) | [Prompt Engineering →](./prompt-engineering.md)

---

## The Frontend Prompting Principle

Frontend code is the most subjective layer of the stack. The model will happily generate a React component — the question is whether it generates the *right* one for *your* design system, *your* state management approach, and *your* performance constraints.

The core discipline:

> **Constrain the aesthetic, the architecture, and the library. Let the model handle the typing.**

Without constraints, you get a component that works and looks like every other AI-generated component. With constraints, you get a component that belongs to *your* system.

---

## Component System Design

### Component Architecture Prompt

```
Design a component architecture for [application type] using [React/Vue/Svelte].

DESIGN SYSTEM:
- UI library: [shadcn/ui, Radix, MUI, custom]
- Styling: [Tailwind, CSS Modules, styled-components]
- Typography: [font stack]
- Color palette: [primary, secondary, accent, semantic colors]
- Spacing scale: [4px base, 8px grid, etc.]
- Border radius: [token values]

COMPONENT HIERARCHY:
- Primitives (Button, Input, Text, Icon)
- Composites (Card, Modal, Dropdown, Table)
- Features (TaskCard, ProjectSidebar, UserAvatar)
- Pages (Dashboard, Settings, ProjectDetail)

REQUIREMENTS:
- Compound component pattern for complex components
- Polymorphic `as` prop for semantic HTML flexibility
- Forward refs on all interactive components
- Accessible by default (ARIA attributes, keyboard navigation)
- Dark mode support via CSS custom properties
- Responsive: mobile-first breakpoints

FOR EACH COMPONENT PROVIDE:
1. Props interface (TypeScript)
2. Default variants
3. Accessibility requirements
4. Usage example
```

### Component Structure

```
src/
├── components/
│   ├── ui/                    # Primitives — design system layer
│   │   ├── button.tsx
│   │   ├── input.tsx
│   │   ├── dialog.tsx
│   │   ├── dropdown-menu.tsx
│   │   └── ...
│   ├── composed/              # Composite components
│   │   ├── data-table/
│   │   │   ├── data-table.tsx
│   │   │   ├── columns.tsx
│   │   │   └── toolbar.tsx
│   │   ├── command-palette.tsx
│   │   └── ...
│   └── features/              # Feature-specific components
│       ├── tasks/
│       │   ├── task-card.tsx
│       │   ├── task-list.tsx
│       │   ├── task-form.tsx
│       │   └── task-filters.tsx
│       ├── projects/
│       │   ├── project-sidebar.tsx
│       │   └── project-header.tsx
│       └── ...
├── hooks/                     # Custom hooks
│   ├── use-tasks.ts
│   ├── use-debounce.ts
│   └── use-media-query.ts
├── lib/                       # Utilities
│   ├── api.ts                 # API client
│   ├── cn.ts                  # Class name utility
│   └── validators.ts
├── stores/                    # State management
│   ├── task-store.ts
│   └── ui-store.ts
└── styles/
    ├── globals.css
    └── tokens.css             # Design tokens
```

### Design Tokens (CSS Custom Properties)

```css
/* tokens.css — single source of truth */
:root {
  /* Colors — HSL for programmatic manipulation */
  --color-bg: 0 0% 4%;
  --color-surface: 0 0% 8%;
  --color-surface-raised: 0 0% 12%;
  --color-border: 0 0% 18%;
  --color-text-primary: 0 0% 95%;
  --color-text-secondary: 0 0% 60%;
  --color-text-muted: 0 0% 40%;
  --color-accent: 262 83% 64%;
  --color-accent-hover: 262 83% 72%;
  --color-destructive: 0 72% 51%;
  --color-success: 142 71% 45%;
  --color-warning: 38 92% 50%;

  /* Spacing — 4px base grid */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-6: 1.5rem;
  --space-8: 2rem;
  --space-12: 3rem;
  --space-16: 4rem;

  /* Typography */
  --font-sans: 'Geist', system-ui, sans-serif;
  --font-mono: 'Geist Mono', 'Fira Code', monospace;
  --font-size-xs: 0.75rem;
  --font-size-sm: 0.875rem;
  --font-size-base: 1rem;
  --font-size-lg: 1.125rem;
  --font-size-xl: 1.25rem;
  --font-size-2xl: 1.5rem;
  --font-size-3xl: 2rem;

  /* Radius */
  --radius-sm: 0.375rem;
  --radius-md: 0.5rem;
  --radius-lg: 0.75rem;
  --radius-full: 9999px;

  /* Shadows */
  --shadow-sm: 0 1px 2px hsl(0 0% 0% / 0.3);
  --shadow-md: 0 4px 6px hsl(0 0% 0% / 0.3);
  --shadow-lg: 0 10px 15px hsl(0 0% 0% / 0.3);

  /* Transitions */
  --transition-fast: 100ms ease;
  --transition-base: 200ms ease;
  --transition-slow: 300ms ease;
}
```

---

## State Management

### State Management Decision Prompt

```
Recommend a state management strategy for this React application:

APPLICATION TYPE: [SaaS dashboard / e-commerce / real-time collaboration]

STATE CATEGORIES:
- Server state: [data from API — tasks, users, projects]
- Client state: [UI state — modals, sidebar, filters]
- URL state: [query params — search, pagination, filters]
- Form state: [complex forms with validation]
- Real-time state: [WebSocket updates]

REQUIREMENTS:
- Optimistic updates for mutations
- Cache invalidation on related data changes
- Offline support: [yes/no]
- Undo/redo: [yes/no for specific actions]

EVALUATE THESE OPTIONS:
1. React Query (TanStack Query) + Zustand
2. React Query + Context
3. Redux Toolkit + RTK Query
4. Jotai + React Query

For each, provide:
- Boilerplate comparison (LOC for a typical CRUD feature)
- Performance characteristics
- DevTools and debugging experience
- Learning curve
- When it breaks down
```

### Recommended Pattern: TanStack Query + Zustand

```typescript
// hooks/use-tasks.ts — Server state via TanStack Query
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { api } from '@/lib/api';
import type { Task, TaskCreate, TaskUpdate } from '@/types';

// Query keys — centralized for cache management
export const taskKeys = {
  all: ['tasks'] as const,
  lists: () => [...taskKeys.all, 'list'] as const,
  list: (projectId: string, filters: TaskFilters) =>
    [...taskKeys.lists(), projectId, filters] as const,
  details: () => [...taskKeys.all, 'detail'] as const,
  detail: (id: string) => [...taskKeys.details(), id] as const,
};

// Queries
export function useTasks(projectId: string, filters: TaskFilters) {
  return useQuery({
    queryKey: taskKeys.list(projectId, filters),
    queryFn: () => api.tasks.list(projectId, filters),
    staleTime: 30_000, // Consider fresh for 30s
    placeholderData: keepPreviousData, // No flash during filter changes
  });
}

export function useTask(taskId: string) {
  return useQuery({
    queryKey: taskKeys.detail(taskId),
    queryFn: () => api.tasks.get(taskId),
    staleTime: 60_000,
  });
}

// Mutations with optimistic updates
export function useCreateTask(projectId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: TaskCreate) => api.tasks.create(projectId, data),

    // Optimistic update: add to list immediately
    onMutate: async (newTask) => {
      await queryClient.cancelQueries({ queryKey: taskKeys.lists() });

      const previousTasks = queryClient.getQueryData(
        taskKeys.list(projectId, {})
      );

      queryClient.setQueryData(
        taskKeys.list(projectId, {}),
        (old: Task[] | undefined) => [
          ...(old ?? []),
          { ...newTask, id: `temp-${Date.now()}`, status: 'todo' },
        ]
      );

      return { previousTasks };
    },

    // Rollback on error
    onError: (_err, _newTask, context) => {
      if (context?.previousTasks) {
        queryClient.setQueryData(
          taskKeys.list(projectId, {}),
          context.previousTasks
        );
      }
    },

    // Refetch after success to get real server data
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: taskKeys.lists() });
    },
  });
}

export function useUpdateTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: TaskUpdate }) =>
      api.tasks.update(id, data),

    onMutate: async ({ id, data }) => {
      await queryClient.cancelQueries({ queryKey: taskKeys.detail(id) });

      const previousTask = queryClient.getQueryData(taskKeys.detail(id));

      queryClient.setQueryData(taskKeys.detail(id), (old: Task | undefined) =>
        old ? { ...old, ...data, updated_at: new Date().toISOString() } : old
      );

      return { previousTask };
    },

    onError: (_err, { id }, context) => {
      if (context?.previousTask) {
        queryClient.setQueryData(taskKeys.detail(id), context.previousTask);
      }
    },

    onSettled: (_data, _error, { id }) => {
      queryClient.invalidateQueries({ queryKey: taskKeys.detail(id) });
      queryClient.invalidateQueries({ queryKey: taskKeys.lists() });
    },
  });
}
```

```typescript
// stores/ui-store.ts — Client state via Zustand
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

interface UIState {
  // Sidebar
  sidebarOpen: boolean;
  sidebarWidth: number;
  toggleSidebar: () => void;

  // Command palette
  commandPaletteOpen: boolean;
  openCommandPalette: () => void;
  closeCommandPalette: () => void;

  // Active views
  activeProjectId: string | null;
  setActiveProject: (id: string) => void;

  // Theme
  theme: 'dark' | 'light' | 'system';
  setTheme: (theme: 'dark' | 'light' | 'system') => void;
}

export const useUIStore = create<UIState>()(
  devtools(
    persist(
      (set) => ({
        sidebarOpen: true,
        sidebarWidth: 280,
        toggleSidebar: () =>
          set((state) => ({ sidebarOpen: !state.sidebarOpen })),

        commandPaletteOpen: false,
        openCommandPalette: () => set({ commandPaletteOpen: true }),
        closeCommandPalette: () => set({ commandPaletteOpen: false }),

        activeProjectId: null,
        setActiveProject: (id) => set({ activeProjectId: id }),

        theme: 'dark',
        setTheme: (theme) => set({ theme }),
      }),
      {
        name: 'ui-store',
        partialize: (state) => ({
          sidebarOpen: state.sidebarOpen,
          sidebarWidth: state.sidebarWidth,
          theme: state.theme,
        }),
      }
    ),
    { name: 'UIStore' }
  )
);
```

---

## API Integration Contracts

### Type-Safe API Client Prompt

```
Generate a type-safe API client for [framework] that:

API SPEC:
[OpenAPI spec or endpoint list]

REQUIREMENTS:
- Typed request and response for every endpoint
- Automatic token injection from auth store
- Request/response interceptors for:
  - Token refresh on 401
  - Request ID injection
  - Error normalization
- Abort controller support for React Query
- Base URL from environment variable
- Request deduplication for GET requests

OUTPUT:
- API client class/module
- TypeScript types for all endpoints
- Error types matching the backend error envelope
```

### API Client Implementation

```typescript
// lib/api.ts
const BASE_URL = import.meta.env.VITE_API_URL;

interface ApiError {
  code: string;
  message: string;
  field?: string;
  detail?: string;
}

interface ApiResponse<T> {
  data: T;
  meta: { request_id: string; timestamp: string };
  pagination?: { cursor: string | null; has_more: boolean };
  errors: ApiError[] | null;
}

class ApiClient {
  private getToken: () => string | null;
  private onTokenExpired: () => Promise<string | null>;

  constructor(config: {
    getToken: () => string | null;
    onTokenExpired: () => Promise<string | null>;
  }) {
    this.getToken = config.getToken;
    this.onTokenExpired = config.onTokenExpired;
  }

  private async request<T>(
    path: string,
    options: RequestInit & { signal?: AbortSignal } = {}
  ): Promise<T> {
    const token = this.getToken();
    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
      ...(token && { Authorization: `Bearer ${token}` }),
      ...(options.headers as Record<string, string>),
    };

    const response = await fetch(`${BASE_URL}${path}`, {
      ...options,
      headers,
    });

    // Handle token refresh
    if (response.status === 401) {
      const newToken = await this.onTokenExpired();
      if (newToken) {
        headers.Authorization = `Bearer ${newToken}`;
        const retryResponse = await fetch(`${BASE_URL}${path}`, {
          ...options,
          headers,
        });
        return this.parseResponse<T>(retryResponse);
      }
      throw new AuthError('Session expired');
    }

    return this.parseResponse<T>(response);
  }

  private async parseResponse<T>(response: Response): Promise<T> {
    const json: ApiResponse<T> = await response.json();

    if (!response.ok || json.errors) {
      throw new ApiRequestError(
        response.status,
        json.errors ?? [{ code: 'UNKNOWN', message: 'Unknown error' }],
        json.meta?.request_id
      );
    }

    return json.data;
  }

  // Resource-specific methods
  tasks = {
    list: (projectId: string, filters?: TaskFilters) =>
      this.request<Task[]>(
        `/api/v1/projects/${projectId}/tasks?${toQueryString(filters)}`
      ),
    get: (id: string) => this.request<Task>(`/api/v1/tasks/${id}`),
    create: (projectId: string, data: TaskCreate) =>
      this.request<Task>(`/api/v1/projects/${projectId}/tasks`, {
        method: 'POST',
        body: JSON.stringify(data),
      }),
    update: (id: string, data: TaskUpdate) =>
      this.request<Task>(`/api/v1/tasks/${id}`, {
        method: 'PATCH',
        body: JSON.stringify(data),
      }),
    delete: (id: string) =>
      this.request<void>(`/api/v1/tasks/${id}`, { method: 'DELETE' }),
  };
}

export const api = new ApiClient({
  getToken: () => useAuthStore.getState().accessToken,
  onTokenExpired: () => useAuthStore.getState().refreshToken(),
});
```

---

## UI Performance

### Performance Audit Prompt

```
Audit this React component tree for performance issues:

COMPONENT CODE:
[Paste components]

CHECK FOR:
1. Unnecessary re-renders (missing memoization, unstable references)
2. Heavy computations in render path (should be useMemo)
3. Large list rendering without virtualization
4. Unoptimized images (missing lazy loading, wrong format)
5. Layout thrashing (forced synchronous layouts)
6. Bundle size: components importing entire libraries
7. Waterfall requests (sequential fetches that could be parallel)
8. Missing loading/skeleton states causing layout shift

FOR EACH ISSUE:
- Current impact on user experience
- Fix with code
- How to measure the improvement (Chrome DevTools, Lighthouse)
```

### Performance Patterns

```typescript
// Pattern 1: Virtualized lists for large datasets
import { useVirtualizer } from '@tanstack/react-virtual';

function TaskList({ tasks }: { tasks: Task[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  
  const virtualizer = useVirtualizer({
    count: tasks.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 72, // estimated row height
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: '100%', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.key}
            style={{
              position: 'absolute',
              top: virtualRow.start,
              height: virtualRow.size,
              width: '100%',
            }}
          >
            <TaskCard task={tasks[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}

// Pattern 2: Debounced search input
function SearchInput({ onSearch }: { onSearch: (query: string) => void }) {
  const [value, setValue] = useState('');
  const debouncedValue = useDebounce(value, 300);

  useEffect(() => {
    onSearch(debouncedValue);
  }, [debouncedValue, onSearch]);

  return (
    <Input
      value={value}
      onChange={(e) => setValue(e.target.value)}
      placeholder="Search tasks..."
    />
  );
}

// Pattern 3: Stable callbacks to prevent child re-renders
function ProjectView({ projectId }: { projectId: string }) {
  const [filters, setFilters] = useState<TaskFilters>({});
  
  // Stable reference — won't cause TaskList to re-render
  const handleFilterChange = useCallback((newFilters: TaskFilters) => {
    setFilters(prev => ({ ...prev, ...newFilters }));
  }, []);

  // Expensive computation cached
  const sortedTasks = useMemo(
    () => tasks?.toSorted((a, b) => b.priority - a.priority),
    [tasks]
  );

  return (
    <>
      <TaskFilters filters={filters} onChange={handleFilterChange} />
      <TaskList tasks={sortedTasks ?? []} />
    </>
  );
}
```

---

## Accessibility

### Accessibility Audit Prompt

```
Audit this component for WCAG 2.2 AA compliance:

COMPONENT:
[Paste code]

CHECK:
1. Keyboard navigation (Tab, Enter, Escape, Arrow keys)
2. Screen reader announcements (ARIA labels, live regions)
3. Focus management (focus trap in modals, focus restoration)
4. Color contrast (minimum 4.5:1 for text, 3:1 for large text)
5. Touch target sizes (minimum 44x44px)
6. Motion preferences (prefers-reduced-motion)
7. Error announcements for form validation
8. Skip navigation links

FOR EACH ISSUE:
- WCAG criterion number
- Current behavior
- Expected behavior
- Fix with code
```

### Accessibility Patterns

```tsx
// Pattern: Accessible modal with focus trap
function Dialog({ open, onClose, title, children }: DialogProps) {
  const closeButtonRef = useRef<HTMLButtonElement>(null);
  
  // Focus trap and restoration
  useEffect(() => {
    if (open) {
      const previousFocus = document.activeElement as HTMLElement;
      closeButtonRef.current?.focus();
      
      return () => {
        previousFocus?.focus(); // Restore focus on close
      };
    }
  }, [open]);

  // Close on Escape
  useEffect(() => {
    if (!open) return;
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [open, onClose]);

  if (!open) return null;

  return (
    <div
      role="dialog"
      aria-modal="true"
      aria-labelledby="dialog-title"
      className="dialog-overlay"
      onClick={(e) => {
        if (e.target === e.currentTarget) onClose(); // Click outside to close
      }}
    >
      <div className="dialog-content">
        <h2 id="dialog-title">{title}</h2>
        {children}
        <button
          ref={closeButtonRef}
          onClick={onClose}
          aria-label="Close dialog"
          className="dialog-close"
        >
          ×
        </button>
      </div>
    </div>
  );
}

// Pattern: Reduced motion
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

---

## Design System Generation

### Design System Prompt

```
Generate a design system for [application type] with this aesthetic direction:

AESTHETIC: [brutalist / minimal / luxury / editorial / playful]
MOOD: [describe the feeling: "confident and precise" / "warm and inviting"]
COLOR STRATEGY: [dark mode primary / light with accents / vibrant]

REQUIRED TOKENS:
- Color palette (semantic: background, surface, text, accent, destructive, success)
- Typography scale (font families, sizes, weights, line heights)
- Spacing scale (4px base unit)
- Border radius scale
- Shadow scale
- Transition/animation timing
- Breakpoints

REQUIRED COMPONENTS:
- Button (variants: primary, secondary, ghost, destructive; sizes: sm, md, lg)
- Input (with label, helper text, error state)
- Select (with search)
- Dialog/Modal
- Toast/Notification
- Card
- Badge
- Avatar
- Skeleton loader
- Empty state

FOR EACH COMPONENT:
- CSS/class design with all variants
- Dark mode support
- Keyboard accessible
- Animation on interaction
- Responsive behavior
```

---

## Refactoring UI via Prompting

### Refactoring Prompt Template

```
Refactor this component. Current issues:

COMPONENT:
[Paste code]

PROBLEMS:
1. [Too many responsibilities — does X, Y, and Z]
2. [State management mixed with presentation]
3. [Styling is inconsistent with design system]
4. [Not accessible: missing ARIA, no keyboard nav]
5. [Performance: re-renders on every parent update]

REFACTOR TO:
- Split into [N] focused components
- Extract business logic to custom hook(s)
- Use design system tokens for all styling
- Add accessibility attributes
- Memoize expensive computations
- Maintain exact same external API (props interface)

CONSTRAINTS:
- Zero visual regressions
- Existing tests must pass
- New components must be individually testable
```

---

## Common Failure Modes

| Failure | Symptom | Root Cause |
|---------|---------|------------|
| **AI component soup** | 50 components, no design system | Prompting per-component without establishing system tokens first |
| **Prop drilling of death** | Props passed through 6 layers | No state management strategy in prompt. Model defaults to lifting state |
| **CSS specificity wars** | `!important` everywhere | Mixing AI-generated styles with library styles. Specify styling approach |
| **Flash of unstyled content** | Layout jump on load | No skeleton states. Prompt didn't mention loading states |
| **Accessibility afterthought** | Screen reader unusable | Accessibility not in initial prompt. Must be a constraint, not a phase |
| **Bundle bloat** | 2MB vendor bundle | Model imports entire lodash/moment.js. Specify "tree-shakeable imports only" |
| **Re-render cascade** | UI stutters on interaction | Inline object/function creation in JSX. Specify memoization strategy |

---

## Production Checklist

- [ ] Design tokens defined in CSS custom properties (single source of truth)
- [ ] Component library uses existing UI primitives (shadcn, Radix, etc.)
- [ ] State management: server state (React Query) vs client state (Zustand) separated
- [ ] API client is type-safe with automatic token management
- [ ] Optimistic updates for common mutations
- [ ] Virtualized lists for datasets > 100 items
- [ ] Skeleton loaders for every async data view
- [ ] Error boundaries at route level
- [ ] Accessibility: keyboard navigable, screen reader compatible
- [ ] Performance: Lighthouse Performance > 90
- [ ] Bundle size audited (no unnecessary imports)
- [ ] Responsive: tested at 320px, 768px, 1024px, 1440px
- [ ] prefers-reduced-motion respected
- [ ] prefers-color-scheme handled
- [ ] Forms validated client-side AND server-side
- [ ] XSS prevention: no `dangerouslySetInnerHTML` without sanitization

---

← [Backend](./backend.md) | [Back to Index](./README.md) | [Prompt Engineering →](./prompt-engineering.md)
