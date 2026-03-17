# Agent Skills

A collection of reusable skills for Claude Code.

## Shared Conventions

All skills follow these conventions:

- Unified response envelope: `{ success, data?, meta?, error? }`
- `meta.requestId` always present for request tracing
- Required header: `X-Device-Type: web | app | desktop`
- Response properties: **camelCase**
- Error codes: **UPPER_SNAKE_CASE**

## Skills

### api-design

REST API design skill that generates three artifacts from a resource description:

- **OpenAPI spec** — `openapi.yaml` following 3.1 conventions
- **API documentation** — `API.md` with endpoints, examples, error codes
- **Route code** — Framework-specific route scaffolding with zod validation

### error-handling

Error handling patterns and error taxonomy:

- **Error taxonomy** — Operational vs Programmer errors, custom error class hierarchy
- **Handling patterns** — Express/Next.js/Hono/FastAPI error handlers, retry, circuit breaker
- **Logging & monitoring** — Structured logging, alert thresholds, requestId correlation

### auth

Authentication and authorization patterns:

- **JWT patterns** — Access/refresh tokens, device-specific storage, token rotation
- **Authorization** — RBAC role hierarchy, ABAC policies, row-level security
- **Security checklist** — Password hashing, brute force protection, CSRF, common vulnerabilities

### database

Database design, query optimization, and migration strategies:

- **Schema design** — Naming conventions, audit columns, indexing strategy, relationship patterns
- **Query patterns** — ORM examples (Prisma/Drizzle/Knex), N+1 solutions, pagination, transactions
- **Migration patterns** — Zero-downtime migrations, expand-contract, safe column operations

### coding-standards

Universal coding standards for TypeScript, React, and Node.js:

- **TypeScript standards** — Naming, immutability, type safety, async patterns, input validation
- **React patterns** — Component structure, hooks, state management, memoization, lazy loading
- **Testing & quality** — AAA pattern, code smell detection, performance checklist

### frontend-patterns

Frontend development patterns for React, Next.js, and performant UIs:

- **Component patterns** — Composition, compound components, render props, error boundaries
- **Custom hooks** — State management, data fetching, debounce, media query hooks
- **Performance** — Memoization, code splitting, virtualization, form optimization
- **Accessibility & animation** — Keyboard navigation, focus management, ARIA, Framer Motion

### backend-patterns

Backend architecture patterns for scalable server-side applications:

- **Architecture patterns** — Repository / Service / Controller layers, middleware pipeline, dependency injection
- **Database optimization** — N+1 prevention, SELECT minimization, transactions, indexing, connection pooling, pagination
- **Caching strategies** — Redis decorator pattern, Cache-Aside, multi-level caching, HTTP cache headers
- **Resilience patterns** — Rate limiting, background jobs, structured logging, health checks, graceful shutdown

### e2e-testing

Playwright E2E testing patterns:

- **Page Object Model** — Base class, page classes, auth fixtures, storageState reuse
- **Test patterns** — Wait strategies, flaky test fixes, API mocking, critical flow testing
- **Config & CI/CD** — Playwright config, artifact management, GitHub Actions, sharding

### security-review

Security review and OWASP Top 10 compliance:

- **Secrets & input** — Secret management, zod validation, file upload, data sanitization
- **Injection & XSS** — SQL injection prevention, CSP, CSRF protection, rate limiting
- **Auth & access** — Token storage, authorization checks, RLS, security testing
- **Deployment checklist** — Pre-deployment security verification

### git-workflow

Git branching, commit conventions, and release management:

- **Branching strategy** — GitHub Flow, branch naming, PR template, protection rules
- **Commit conventions** — Conventional Commits, commitlint, husky + lint-staged
- **Release workflow** — Semantic Versioning, changelog generation, hotfix flow

### ci-cd

CI/CD pipeline design and deployment automation:

- **Pipeline design** — GitHub Actions workflows, caching, parallel jobs, monorepo CI
- **Deployment strategies** — Blue-green, canary, rolling, feature flags
- **Environment management** — Tier config, secret management, preview deployments

### code-review

Code review process and quality checklist:

- **Review checklist** — Correctness, security, performance, maintainability, type safety
- **Review process** — Feedback patterns, approval criteria, PR size guidelines, CODEOWNERS

### performance

Frontend and backend performance optimization:

- **Web Vitals** — LCP, INP, CLS optimization, image/font/bundle optimization
- **Backend performance** — Query optimization, caching layers, compression, pagination
- **Load testing** — k6 scripts, test types, performance budgets, CI integration

### observability

Monitoring, logging, and distributed tracing:

- **Logging** — Structured JSON logging, pino setup, sensitive data filtering
- **Monitoring & alerting** — RED/USE methods, SLO/SLI, dashboard design, alert rules
- **Tracing** — OpenTelemetry, requestId propagation, health checks, graceful degradation

### documentation

Technical documentation and architecture decision records:

- **ADR template** — Architecture Decision Records with examples
- **Project docs** — README, CONTRIBUTING, CHANGELOG, GitHub templates
- **API docs** — Endpoint documentation, SDK examples, versioning, migration guides

## Installation

Copy a skill directory into `~/.claude/skills/`:

```bash
cp -r api-design ~/.claude/skills/api-design
cp -r error-handling ~/.claude/skills/error-handling
cp -r auth ~/.claude/skills/auth
cp -r database ~/.claude/skills/database
cp -r coding-standards ~/.claude/skills/coding-standards
cp -r frontend-patterns ~/.claude/skills/frontend-patterns
cp -r backend-patterns ~/.claude/skills/backend-patterns
cp -r e2e-testing ~/.claude/skills/e2e-testing
cp -r security-review ~/.claude/skills/security-review
cp -r git-workflow ~/.claude/skills/git-workflow
cp -r ci-cd ~/.claude/skills/ci-cd
cp -r code-review ~/.claude/skills/code-review
cp -r performance ~/.claude/skills/performance
cp -r observability ~/.claude/skills/observability
cp -r documentation ~/.claude/skills/documentation
cp -r tdd-workflow ~/.claude/skills/tdd-workflow
```

## License

MIT
