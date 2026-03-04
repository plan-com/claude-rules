# Frontend Developer Role — React Native Stack — Claude Rules

You are assisting a **Frontend Developer working with React Native** at Plan.com. Follow the general frontend rules in `roles/frontend/CLAUDE.md` plus these React Native-specific rules.

## Stack

- **Framework**: React Native 0.73+ (New Architecture preferred)
- **Language**: TypeScript (strict mode)
- **CLI/Build**: Expo (managed or bare) or React Native CLI — follow the project's established choice
- **Navigation**: React Navigation (preferred) or Expo Router
- **State Management**: Redux Toolkit, Zustand, or TanStack Query — follow the project's established choice
- **Styling**: StyleSheet API, NativeWind (Tailwind for RN), or styled-components/native — follow the project's choice
- **Testing**: Jest, React Native Testing Library, Detox or Maestro for E2E
- **Linting**: ESLint with `@react-native/eslint-config`, TypeScript parser
- **Formatting**: Prettier
- **CI/CD**: Jenkins builds on Cluster-D-Platform, distributed via internal channels (TestFlight, Firebase App Distribution, or self-hosted)

## Rules

### React Native Standards

- Use functional components with hooks exclusively — no class components.
- Follow the Rules of Hooks: only call at the top level, only call from React functions.
- Target both iOS and Android unless the project specifies otherwise.
- Test on both platforms regularly — never assume behavior is identical.
- Use the New Architecture (Fabric renderer, TurboModules) when the project supports it.
- Prefer cross-platform solutions — use `Platform.select` or `Platform.OS` only for genuine platform differences.

### Project Structure

- Co-locate related files: component, styles, tests, and types together.
- Separate platform-specific files with `.ios.tsx` / `.android.tsx` suffixes only when behavior genuinely diverges.
- Keep navigation configuration centralized — define all routes and params with TypeScript types.
- Organize by feature/domain, not by file type (avoid flat `components/`, `screens/`, `hooks/` at root level for large projects).

### Component Patterns

- Keep components small and focused — extract when a component handles multiple concerns.
- Use `FlatList` or `FlashList` for lists — never map arrays into `ScrollView` for dynamic data.
- Always provide `keyExtractor` for list components.
- Handle loading, error, and empty states for every data-fetching screen.
- Use `React.memo` only when profiling shows unnecessary re-renders.
- Implement proper cleanup in `useEffect` (subscriptions, listeners, timers).

### Navigation

- Use React Navigation with typed routes — define `RootStackParamList` and all screen params.
- Use deep linking configuration for all navigable screens.
- Handle navigation state persistence for development (restore screens on reload).
- Use `navigation.goBack()` instead of `navigation.navigate()` to parent — respect the back stack.
- Implement proper screen focus/blur handling for data fetching and cleanup.
- Use `useFocusEffect` from React Navigation for screen-level side effects.

### Styling & Layout

- Use `StyleSheet.create` for all static styles — avoid inline style objects.
- Use Flexbox for all layouts — React Native uses Flexbox by default.
- Use design tokens (from the design system) for colors, spacing, typography, and sizing.
- Respect safe areas: use `SafeAreaView` or `useSafeAreaInsets` for notches and system bars.
- Support both light and dark mode using the system's `useColorScheme` or a theme context.
- Handle different screen sizes — avoid hardcoded pixel values for layout dimensions.
- Use `Dimensions` or `useWindowDimensions` for responsive layouts.

### Native Modules & Platform APIs

- Prefer Expo modules or community libraries over writing custom native code when available.
- When writing native modules, use TurboModules (New Architecture) for better performance.
- Handle platform permissions properly — request at the point of use, explain why, handle denial gracefully.
- Use `react-native-permissions` or Expo's permissions API for a unified permission flow.
- Always check feature availability before using platform APIs (camera, biometrics, geolocation, etc.).

### Performance

- Use `FlashList` (Shopify) over `FlatList` for large or complex lists.
- Avoid unnecessary re-renders — use `useCallback`, `useMemo`, and `React.memo` where profiling shows impact.
- Move heavy computations off the JS thread — use `InteractionManager.runAfterInteractions` or background threads.
- Use `react-native-reanimated` for animations running on the UI thread — avoid `Animated` API for complex animations.
- Optimize images: use appropriate resolutions per screen density, cache with `react-native-fast-image` or Expo Image.
- Minimize bridge traffic (Classic Architecture) — batch calls, avoid frequent JS-to-native communication.
- Use Hermes engine for faster startup and reduced memory usage.
- Profile with Flipper, React DevTools, or Xcode Instruments on Cluster-A-Dev-Stage.

### Offline & Networking

- Use a centralized API client — never call `fetch` directly from components.
- Handle offline state gracefully — show appropriate UI when the device is offline.
- Implement retry logic with exponential backoff for failed network requests.
- Cache API responses locally where appropriate (TanStack Query persistence, MMKV, AsyncStorage).
- Use `NetInfo` to detect connectivity changes and adapt behavior.
- Handle slow network conditions — show progress indicators and allow cancellation.

### Testing

- Use React Native Testing Library — test component behavior from the user's perspective.
- Query elements by role, label, text, or testID — prefer accessible queries.
- Mock native modules in Jest setup (camera, geolocation, etc.).
- Use MSW (Mock Service Worker) for API mocking in tests.
- Write Detox or Maestro E2E tests for critical user flows on both platforms.
- Test on real devices regularly — simulators/emulators do not catch all issues.
- Test accessibility with screen readers (VoiceOver on iOS, TalkBack on Android).

### Build & Distribution

- Use Expo EAS Build or Fastlane for automated build pipelines.
- Jenkins on Cluster-D-Platform triggers builds — artifacts distributed via internal channels.
- Maintain separate build configurations for environments: `dev`, `staging`, `production`.
- Use environment variables via `react-native-config`, Expo's `.env` support, or build-time injection.
- Never hardcode API URLs or secrets in the app bundle — inject at build time from Vault via Jenkins.
- Code signing: manage iOS certificates and Android keystores securely — store in Vault, inject in CI.
- Manage app versioning: auto-increment build numbers in CI, semantic versioning for releases.

### Over-the-Air (OTA) Updates

- Use Expo Updates or CodePush for JavaScript-only changes — reduces App Store review cycles.
- OTA updates must not change native code — native changes require a full binary release.
- Test OTA updates on Cluster-A-Dev-Stage before pushing to Cluster-B-Prod-Blue or Cluster-C-Prod-Green.
- Implement rollback capability for failed OTA updates.
- Version OTA updates to prevent incompatible JS bundles running on old native shells.

### Accessibility

- All interactive elements must have accessibility labels (`accessibilityLabel`).
- Use `accessibilityRole` and `accessibilityState` to convey element purpose and state.
- Support dynamic font sizes — use relative font scaling, never hardcode font sizes that ignore system settings.
- Ensure minimum touch target size of 44x44pt.
- Test with VoiceOver (iOS) and TalkBack (Android) on real devices.
- Support `prefers-reduced-motion` — disable animations when the user requests it.
- Implement proper focus order for screen readers.

### Logging & Observability

- Use Sentry React Native SDK for crash reporting and error tracking.
- Include device info, OS version, and app version in error reports.
- Track screen views and key user actions for analytics.
- Integrate OpenTelemetry for distributed tracing of API calls from the app.
- Log meaningful errors to a backend endpoint — never `console.log` in production builds.
- Use Grafana dashboards on Cluster-D-Platform to monitor mobile-specific metrics (crash rate, ANR rate, startup time).

### Security

- Never store sensitive data in AsyncStorage or MMKV unencrypted — use the platform's secure storage (Keychain on iOS, Keystore on Android) via `react-native-keychain` or Expo SecureStore.
- Use certificate pinning for API communication in production builds.
- Enable ProGuard (Android) and Bitcode/code stripping (iOS) to obfuscate release builds.
- Validate all deep link parameters — treat them as untrusted input.
- Disable debug features, React DevTools, and Flipper in production builds.
- Use biometric authentication where appropriate — never roll your own.
- Handle sensitive data in memory carefully — clear credentials from state when no longer needed.

## Response Guidelines

- Write idiomatic React Native with TypeScript and hooks.
- Always consider both iOS and Android behavior.
- Include proper loading, error, empty, and offline states.
- Provide React Native Testing Library tests alongside new components.
- Flag performance concerns (JS thread blocking, unnecessary re-renders, heavy bridge traffic).
- Consider accessibility in all UI suggestions — labels, roles, touch targets.
- Note platform differences when they affect the implementation.
