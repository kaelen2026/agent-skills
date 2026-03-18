# Agent Skills

A collection of reusable skills for Claude Code.

## Shared Conventions

All skills follow these conventions:

- Unified response envelope: `{ success, data?, meta?, error? }` — defined in [api-design](api-design/SKILL.md), other skills reference it
- `meta.requestId` always present for request tracing
- Required header: `X-Device-Type: web | app | desktop`
- Response properties: **camelCase**
- Error codes: **UPPER_SNAKE_CASE**
- Structured logging standards — defined in [observability](observability/SKILL.md), other skills reference it
- Rate limiting implementation — owned by [backend-patterns](backend-patterns/SKILL.md), audited by security-review/performance

## Ownership Matrix

Cross-cutting concerns have a single owner to avoid duplication:

| Concern | Owner (defines) | References (audits/uses) |
|---------|----------------|--------------------------|
| Response envelope | api-design | auth, backend-patterns, error-handling |
| Structured logging | observability | backend-patterns, error-handling, security-review |
| N+1 query (schema/index) | database | backend-patterns (app layer), performance (audit) |
| Rate limiting (impl) | backend-patterns | security-review (audit), auth (auth endpoints) |
| Auth security (httpOnly/CSRF) | auth | security-review (general OWASP audit) |
| Pre-commit hooks | git-workflow | — |

## Skills (18)

### prd

Product requirements full lifecycle — Phase 1: interactive PRD writing (user interviews + codebase exploration + deep module design). Phase 2: decompose PRD into vertical-slice implementation specs.

### planning

Implementation planning — feature specs, implementation plans, architecture design templates.

### grill-me

Universal probing skill — decision-tree traversal questioning to uncover requirement blind spots, expose hidden assumptions, and eliminate ambiguity.

### api-design

REST API design skill that generates three artifacts from a resource description:

- **OpenAPI spec** — `openapi.yaml` following 3.1 conventions
- **API documentation** — `API.md` with endpoints, examples, error codes
- **Route code** — Framework-specific route scaffolding with zod validation

### error-handling

Error handling patterns and error taxonomy:

- **Error taxonomy** — Operational vs Programmer errors, custom error class hierarchy
- **Handling patterns** — Express/Next.js/Hono/FastAPI error handlers, retry, circuit breaker

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
- **Database optimization** — N+1 prevention (app layer), SELECT minimization, transactions, connection pooling
- **Caching strategies** — Redis decorator pattern, Cache-Aside, multi-level caching, HTTP cache headers
- **Resilience patterns** — Rate limiting (implementation owner), background jobs, health checks, graceful shutdown

### e2e-testing

Playwright E2E testing patterns:

- **Page Object Model** — Base class, page classes, auth fixtures, storageState reuse
- **Test patterns** — Wait strategies, flaky test fixes, API mocking, critical flow testing
- **CI/CD integration** — Parallel execution, sharding, retry configuration, artifact collection

### tdd-workflow

Test-driven development workflow enforcing 80%+ coverage:

- **Unit testing** — Vitest/Jest patterns, mocking, snapshot testing
- **Integration testing** — API testing, database testing, service testing
- **E2E testing** — Critical path coverage, smoke tests

### security-review

Security audit skill covering OWASP Top 10:

- **Secrets & input** — Environment variables, zod validation, file upload security
- **Injection & XSS** — Parameterized queries, HTML sanitization, CSP headers
- **Auth & access** — httpOnly cookies (audits, impl → auth), RLS, CORS
- **Deployment checklist** — Rate limiting (audits, impl → backend-patterns), dependency audit, HTTPS

### code-review

Universal code review skill for all languages and frameworks.

### git-workflow

Git workflow management — branching strategy, commit conventions, PR process, release workflow, worktree parallel development, and pre-commit hook setup (Husky + lint-staged + Biome).

### ci-cd

CI/CD pipeline design, deployment strategies, and environment management.

### observability

Observability skill — logging (owner), monitoring, alerting, distributed tracing best practices:

- **Logging** — Structured JSON, requestId propagation, PII filtering (authoritative source)
- **Monitoring** — RED/USE metrics, SLO/SLI, dashboards
- **Alerting** — Severity levels, routing, runbooks
- **Tracing** — OpenTelemetry, span attributes, context propagation

### performance

Performance optimization — Web Vitals, backend performance, load testing, performance budgets:

- **Frontend** — LCP, INP, CLS, bundle size, image/font optimization
- **Backend** — P95 latency, caching, index optimization, compression
- **Infrastructure** — CDN, connection pooling, async operations
- **Process** — Performance budgets, Lighthouse CI

### documentation

Project documentation — ADR, README, API docs, changelog templates and workflows.

## Installation

Symlink skill directories into `~/.claude/skills/` (source updates are reflected automatically):

```bash
mkdir -p ~/.claude/skills

# From the repo root
for skill in */; do
  [ -f "$skill/SKILL.md" ] && ln -sf "$(pwd)/$skill" ~/.claude/skills/
done
```

## License

MIT
