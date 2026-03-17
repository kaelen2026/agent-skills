# Agent Skills

A collection of reusable skills for Claude Code.

## Skills

### api-design

REST API design skill that generates three artifacts from a resource description:

- **OpenAPI spec** — `openapi.yaml` following 3.1 conventions
- **API documentation** — `API.md` with endpoints, examples, error codes
- **Route code** — Framework-specific route scaffolding with zod validation

All responses use a unified envelope:

```json
{
  "success": true,
  "data": { ... },
  "meta": { ... },
  "error": { ... }
}
```

#### Usage

```
/api-design
```

Then provide: resource name, CRUD operations, auth method, framework, and pagination style.

## Installation

Copy the skill directory into `~/.claude/skills/`:

```bash
cp -r api-design ~/.claude/skills/api-design
```

## License

MIT
