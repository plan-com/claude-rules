# Frontend Developer Role — React Stack — Claude Rules

You are assisting a **Frontend Developer working with React** at Plan.com. Follow the general frontend rules in `roles/frontend/CLAUDE.md` plus these React-specific rules.

## Stack

- **Library**: React 18+
- **Language**: TypeScript (strict mode)
- **Meta-Framework**: Next.js or Vite — follow the project's established choice
- **State Management**: Redux Toolkit, Zustand, or TanStack Query — follow the project's established choice
- **Styling**: Tailwind CSS, CSS Modules, or styled-components — follow the project's established choice
- **Routing**: React Router (Vite) or Next.js App Router / Pages Router — follow the project's choice
- **Testing**: Vitest or Jest, React Testing Library, Playwright or Cypress for E2E
- **Linting**: ESLint with `eslint-plugin-react`, `eslint-plugin-react-hooks`, TypeScript parser
- **Formatting**: Prettier
- **Containerized**: Docker multi-stage builds (Node.js build + nginx runtime), deployed to Plan.com swarm clusters

## Rules

### React Standards

- Use functional components exclusively — no class components in new code.
- Use hooks for all state and side effect management.
- Follow the Rules of Hooks: only call at the top level, only call from React functions.
- Prefer named exports over default exports for better refactoring and IDE support.
- Keep components small and focused — extract when a component exceeds ~150 lines or handles multiple concerns.
- Co-locate related files: component, styles, tests, and types in the same directory.

### Component Patterns

- Use composition over prop drilling — leverage `children`, render props, or compound components.
- Lift state only as high as needed — prefer local state with `useState` and `useReducer`.
- Use React Context sparingly — only for truly global concerns (theme, auth, locale). Never for frequently changing data.
- Implement error boundaries around route segments and critical UI sections.
- Always handle loading, error, and empty states — never assume data is present.
- Use `Suspense` and lazy loading for route-level code splitting.

### Hooks

- Extract reusable logic into custom hooks with clear naming (`useAuth`, `usePagination`, `useDebounce`).
- Use `useCallback` and `useMemo` only when profiling shows unnecessary re-renders — do not apply prematurely.
- Use `useRef` for mutable values that should not trigger re-renders (timers, DOM refs, previous values).
- Use `useEffect` only for synchronization with external systems — not for derived state or event handling.
- Clean up all side effects in `useEffect` return functions (subscriptions, timers, abort controllers).
- Avoid `useEffect` chains — if one effect depends on another, reconsider the data flow.

### State Management

- **Local state**: `useState` / `useReducer` for component-scoped state.
- **Server state**: TanStack Query (React Query) or SWR for API data — handle caching, refetching, and invalidation through the library.
- **Global client state**: Redux Toolkit or Zustand for state shared across unrelated components. Keep the store minimal.
- Never duplicate server state into global state — let the data-fetching library manage it.
- Normalize complex nested data structures in global stores.
- Use selectors to prevent unnecessary re-renders from store changes.

### Next.js Specific (when applicable)

- Follow the App Router conventions when using Next.js 13+.
- Use Server Components by default — add `'use client'` only when the component needs interactivity, hooks, or browser APIs.
- Use `loading.tsx`, `error.tsx`, and `not-found.tsx` for route segment handling.
- Use Server Actions or Route Handlers for data mutations — not API calls from client components.
- Implement proper metadata with `generateMetadata` for SEO.
- Use `next/image` for automatic image optimization.
- Use `next/link` for client-side navigation.

### Forms

- Use controlled components or a form library (React Hook Form, Formik) — follow project convention.
- Validate with Zod or Yup schemas — share validation schemas with the backend when possible.
- Display inline validation errors next to the relevant field.
- Handle form submission states: idle, submitting, success, error.
- Implement proper focus management: focus the first error field on validation failure.
- Disable submit during submission to prevent double-submits.

### Performance

- Use React DevTools Profiler on Cluster-A-Dev-Stage to identify re-render bottlenecks.
- Implement route-based code splitting with `React.lazy` and `Suspense`.
- Virtualize long lists with `react-window` or `@tanstack/react-virtual`.
- Debounce expensive operations (search inputs, resize handlers).
- Avoid inline object/array/function creation in JSX props — these create new references every render.
- Use `React.memo` only for components that receive stable primitive props but re-render due to parent changes — profile first.

### Testing

- Use React Testing Library — test from the user's perspective, not implementation details.
- Query elements by role, label, placeholder, or text — avoid test IDs unless necessary.
- Test user interactions: clicks, typing, form submissions, navigation.
- Use MSW (Mock Service Worker) to mock API responses in tests.
- Test accessibility with `jest-axe` or `@axe-core/playwright`.
- Write Playwright or Cypress E2E tests for critical user flows.
- Snapshot tests only for stable, presentational components — avoid for interactive components.

### Docker & Deployment

- Multi-stage build: `node:20-alpine` for building, `nginx:alpine` for serving.
- Build output goes to nginx's static directory.
- Configure nginx for SPA routing: `try_files $uri $uri/ /index.html`.
- Set cache headers: immutable long-term cache for hashed assets, `no-cache` for `index.html`.
- Environment configuration injected at runtime via `window.__CONFIG__` or an `/config` endpoint — not baked into the build.
- Use `.dockerignore` to exclude `node_modules`, `.git`, test files, and documentation.

### Logging & Observability

- Integrate Sentry (or equivalent) for production error tracking with React Error Boundaries.
- Track Web Vitals (LCP, INP, CLS) and report to monitoring on Cluster-D-Platform.
- Use OpenTelemetry browser SDK for distributed tracing when applicable.
- Log meaningful client-side errors to a backend logging endpoint — never `console.log` in production.

### Security

- Avoid `dangerouslySetInnerHTML` — if required, sanitize with DOMPurify.
- Never store auth tokens in localStorage — use httpOnly cookies managed by the backend.
- Use CSP headers (via Traefik or nginx) to restrict script sources.
- Sanitize user input before rendering.
- Keep dependencies updated — run `npm audit` regularly.
- Use `rel="noopener noreferrer"` on external links.

## Response Guidelines

- Write idiomatic React with TypeScript and hooks.
- Always include proper loading, error, and empty states.
- Provide React Testing Library tests alongside new components.
- Flag unnecessary re-renders and suggest optimizations when profiling warrants it.
- Consider Server Components vs Client Components when working with Next.js.
- Ensure accessibility in all UI suggestions.
