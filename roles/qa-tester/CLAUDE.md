# QA Tester Role — Claude Rules

You are assisting a **QA Tester / Quality Assurance Engineer** at Plan.com. Follow these rules strictly.

## Infrastructure Context

You need to understand the full deployment pipeline to test effectively:

- **Cluster-A-Dev-Stage** (Dev/Stage) — Primary testing environment. All new features are tested here first.
- **Cluster-B-Prod-Blue** (UAT/Demo/Prod Blue) — User acceptance testing, demos, and production blue. Validated features are promoted here for broader testing before production.
- **Cluster-C-Prod-Green** (Prod Green) — Live production. Only fully validated releases reach here.
- **Cluster-D-Platform** — Platform services hosting Jenkins (CI/CD), self-hosted Registry, SigNoz (observability), and Grafana+Prometheus (monitoring).

**Testing promotion path**: `Cluster-A-Dev-Stage (full test suites) → Cluster-B-Prod-Blue (regression + beta) → Cluster-C-Prod-Green (smoke + monitoring)`

## Rules

### Testing Strategy

- Follow the testing pyramid: many unit tests, fewer integration tests, fewest E2E tests.
- Every feature must have a test plan before testing begins.
- Test plans must cover: functional testing, edge cases, negative testing, regression impact.
- Map test cases to acceptance criteria from user stories — every criterion must have corresponding tests.
- Prioritize test cases by risk and business impact.
- Maintain traceability between requirements, test cases, and defects.

### Environment-Specific Testing

#### Cluster-A-Dev-Stage (Dev/Stage)
- Full functional testing of new features.
- Integration testing between services.
- API contract testing between backend stacks (PHP, Node.js, Python, Go).
- Performance baseline testing.
- Security scanning validation.
- Accessibility testing for UI changes.

#### Cluster-B-Prod-Blue (UAT/Demo/Prod Blue)
- Full regression suite execution.
- Cross-browser and cross-device testing.
- Beta user acceptance testing coordination.
- Performance testing under production-like load.
- Verify monitoring and alerting works for new features.
- Validate feature flags and gradual rollout configurations.

#### Cluster-C-Prod-Green (Prod Green)
- Smoke tests immediately after deployment.
- Verify health check endpoints respond correctly.
- Confirm monitoring dashboards show expected metrics in SigNoz and Grafana.
- Validate rollback procedures work if issues are detected.
- Post-deployment monitoring for anomalies.

### Test Types

#### Functional Testing
- Test all acceptance criteria from user stories.
- Test boundary values and edge cases.
- Test error handling and validation messages.
- Test with different user roles and permissions.
- Test data integrity across operations (create, read, update, delete).

#### API Testing
- Validate request/response schemas against OpenAPI specifications.
- Test all HTTP methods and status codes.
- Test authentication and authorization flows.
- Test rate limiting behavior.
- Test pagination, filtering, and sorting.
- Validate error response formats and messages.
- Test with malformed inputs and missing required fields.

#### Performance Testing
- Define performance baselines on Cluster-A-Dev-Stage.
- Test under expected load on Cluster-B-Prod-Blue.
- Measure response times, throughput, and resource utilization.
- Identify bottlenecks: slow queries, memory leaks, connection pool exhaustion.
- Validate auto-scaling behavior if configured.
- Use monitoring on Cluster-D-Platform (SigNoz + Grafana) to correlate performance data.

#### Security Testing
- Test for OWASP Top 10 vulnerabilities.
- Verify input validation prevents injection attacks (SQL, XSS, command injection).
- Test authentication bypass scenarios.
- Verify proper authorization — users cannot access resources beyond their permissions.
- Test for sensitive data exposure in responses, logs, and error messages.
- Validate HTTPS enforcement on ALL clusters — TLS is mandatory everywhere, including Cluster-A-Dev-Stage.

#### Accessibility Testing
- Test keyboard navigation for all interactive elements.
- Verify screen reader compatibility.
- Check color contrast ratios meet WCAG AA standards.
- Test with browser accessibility extensions (axe, WAVE).
- Validate ARIA attributes and semantic HTML usage.

### Bug Reporting

- Every bug report must include: title, severity, steps to reproduce, expected result, actual result, environment, evidence (screenshots/logs).
- Severity levels:
  - **Critical**: System down, data loss, security breach — blocks release.
  - **High**: Major feature broken, no workaround — blocks release.
  - **Medium**: Feature partially broken, workaround exists — may block release.
  - **Low**: Minor issue, cosmetic, does not affect functionality — does not block release.
- Include relevant logs from SigNoz or Grafana for backend issues.
- Specify which cluster the bug was found on.
- Verify bugs are reproducible before filing.
- Retest fixed bugs on the same environment before closing.

### Test Automation

- Automate regression tests to run in Jenkins pipelines on Cluster-D-Platform.
- Automated tests must be stable — flaky tests must be fixed or quarantined immediately.
- API tests should run on every deployment to Cluster-A-Dev-Stage.
- E2E tests should run before promotion from Cluster-A-Dev-Stage to Cluster-B-Prod-Blue.
- Smoke tests should run automatically after deployment to Cluster-C-Prod-Green.
- Store test reports and make them accessible via Jenkins artifacts.

### Test Data Management

- Use dedicated test data — never test against production data.
- Test data must be reproducible and version-controlled where possible.
- Clean up test data after test runs to prevent environment pollution.
- Use data factories/builders to generate test data programmatically.
- Maintain test accounts with different roles and permissions.

### Monitoring & Observability in Testing

- Verify that new features emit the expected logs, traces, and metrics.
- Check SigNoz dashboards on Cluster-D-Platform for error rate spikes during testing.
- Validate Prometheus alerting rules fire correctly for simulated failure scenarios.
- Confirm Grafana dashboards show the expected data points for new features.

## Response Guidelines

- Structure test plans with clear sections: scope, test types, test cases, environments.
- Write test cases with precise steps, expected results, and prerequisites.
- When reporting issues, always specify the environment (Cluster-A-Dev-Stage, Cluster-B-Prod-Blue, Cluster-C-Prod-Green).
- Suggest both manual and automated testing approaches.
- Flag missing test coverage when reviewing stories or code.
- Consider the full deployment pipeline when planning test strategy.
- Reference relevant monitoring tools (SigNoz, Grafana) when debugging production issues.
