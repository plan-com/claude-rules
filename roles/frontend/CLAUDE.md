# Frontend Developer Role — Claude Rules

You are assisting a **Frontend Developer** at Plan.com. Follow these rules strictly.

## Infrastructure Context

Frontend applications at Plan.com are containerized and deployed across Docker Swarm clusters:

- **Cluster-A-Dev-Stage** (Dev/Stage) — **Cluster-B-Prod-Blue** (UAT/Demo/Prod Blue) — **Cluster-C-Prod-Green** (Prod Green)
- Traffic enters via **Traefik** reverse proxy on each swarm
- Monitoring via **SigNoz** and **Grafana+Prometheus** on **Cluster-D-Platform**
- Images built by **Jenkins** and pushed to the **self-hosted Registry** on **Cluster-D-Platform**

For stack-specific rules, see the subdirectories: `react/`, `react-native/`.

## Rules

### General Frontend Principles

- Write clean, component-based, and maintainable code.
- Follow the project's established framework and conventions — do not introduce new frameworks without team consensus.
- Keep components small, focused, and reusable.
- Separate presentation logic from business logic.
- Use TypeScript with strict mode for all frontend projects.
- Prefer composition over inheritance for component patterns.

### Code Quality

- Use ESLint with TypeScript parser and project-specific rules.
- Use Prettier for consistent formatting.
- No `any` types — use `unknown` with type guards when dealing with dynamic data.
- No `@ts-ignore` or `@ts-expect-error` in production code.
- No `console.log` in production — use proper logging utilities.
- No inline styles unless dynamically computed — use CSS modules, Tailwind, or styled-components per project convention.

### Component Architecture

- Follow atomic design or the project's established component hierarchy.
- Keep state as local as possible — lift only when necessary.
- Use proper state management (Redux, Zustand, Pinia, etc.) for global/shared state.
- Handle loading, error, and empty states for every data-fetching component.
- Use error boundaries to prevent cascading failures.
- Implement proper cleanup in component unmount (subscriptions, timers, event listeners).

### API Integration

- Use a centralized API client — never call `fetch` or `axios` directly from components.
- Define API response types with TypeScript interfaces.
- Handle all HTTP error states gracefully — show meaningful error messages to users.
- Implement request cancellation for abandoned navigations.
- Use optimistic updates where appropriate for better UX.
- Cache API responses where appropriate (React Query, SWR, TanStack Query, etc.).

### Performance

- Lazy-load routes and heavy components with code splitting.
- Optimize images: use appropriate formats (WebP, AVIF), responsive sizes, and lazy loading.
- Minimize bundle size — analyze with bundle analyzer regularly.
- Use `useMemo`, `useCallback`, or framework equivalents where profiling shows unnecessary re-renders.
- Avoid layout thrashing — batch DOM reads and writes.
- Implement virtualization for long lists.
- Set performance budgets and monitor them in CI.

### Accessibility (a11y)

- Use semantic HTML elements (`nav`, `main`, `article`, `button`, etc.).
- All interactive elements must be keyboard accessible.
- All images must have meaningful `alt` text (or empty `alt=""` for decorative images).
- Use ARIA attributes only when native HTML semantics are insufficient.
- Maintain sufficient color contrast ratios (WCAG AA minimum).
- Test with screen readers and keyboard navigation.
- Include skip navigation links.

### Testing

- Write unit tests for utility functions and hooks.
- Write component tests with Testing Library (React, Vue, etc.) — test behavior, not implementation.
- Write integration tests for critical user flows.
- Use MSW (Mock Service Worker) for API mocking in tests.
- Test accessibility with `jest-axe` or `@axe-core/playwright`.
- Visual regression testing for critical UI components where applicable.

### Docker & Deployment

- Use multi-stage Docker builds: build stage (Node.js for compilation) and runtime stage (nginx/caddy for static serving).
- Base images from the **self-hosted Registry on Cluster-D-Platform**.
- Configure the web server (nginx) for SPA routing (fallback to `index.html`).
- Set proper cache headers: long-term for hashed assets, no-cache for `index.html`.
- Environment-specific configuration should be injected at runtime (via `window.__CONFIG__` or environment endpoint), not build time.
- Traefik handles TLS termination — the frontend container serves plain HTTP internally.

### Logging & Observability

- Implement error tracking (Sentry or equivalent) for production errors.
- Use structured logging for client-side errors sent to the backend.
- Integrate OpenTelemetry browser SDK for distributed tracing where applicable.
- Track Web Vitals (LCP, FID, CLS) and report to monitoring.
- Use Grafana dashboards on Cluster-D-Platform to monitor frontend performance metrics.

### Security

- Sanitize any user-generated content rendered in the DOM — prevent XSS.
- Use Content Security Policy (CSP) headers via Traefik or nginx.
- Never store sensitive data in localStorage — use httpOnly cookies for auth tokens.
- Validate user input on the client for UX, but never trust it — backend validates authoritatively.
- Use `rel="noopener noreferrer"` for external links with `target="_blank"`.
- Keep dependencies updated — run `npm audit` regularly.

### Git & CI/CD

- Feature branches must pass linting, type checking, and tests in Jenkins before merge.
- Build artifacts are Docker images pushed to the self-hosted Registry.
- Preview deployments on Cluster-A-Dev-Stage for PRs when configured.
- Follow promotion path: `Cluster-A-Dev-Stage → Cluster-B-Prod-Blue → Cluster-C-Prod-Green`.

## Response Guidelines

- Write idiomatic, typed frontend code following the project's framework conventions.
- Always consider accessibility in UI suggestions.
- Include component tests alongside new components.
- Flag performance concerns (large bundles, excessive re-renders).
- Consider responsive design for all layout suggestions.
