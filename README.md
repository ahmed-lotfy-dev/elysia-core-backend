# Elysia Core Backend (Skill)

Scaffold a minimal, production-ready Bun + Elysia backend with Better Auth, Drizzle ORM, Postgres, MCP endpoint, OpenAPI docs, CORS, and security defaults. No business logic, just the base infrastructure.

## Install

```bash
npx skills add ahmed-lotfy-dev/elysia-core-backend --skill elysia-core-backend
```

## What This Skill Does

- Generates a clean Elysia server scaffold using the official generator
- Adds Better Auth with Drizzle adapter and minimal auth schema
- Sets up Drizzle ORM with Postgres
- Adds MCP plugin with a sample `health` tool
- Enables OpenAPI docs at `/docs`
- Configures CORS allowlist, CSRF protection, and security headers
- Provides baseline routes: `/`, `/health`, `/api/csrf`, `/api/auth/*`, `/mcp`

## File Structure It Produces

```
server/
  src/
    index.ts
    auth.ts
    middleware/
      auth-middleware.ts
      logger.ts
      csrf.ts
      security-headers.ts
    db/
      db.ts
      auth-schema.ts
    mcp/
      index.ts
      tools/
        health.ts
    routes/
      health.ts
    lib/
      env.ts
  drizzle.config.ts
  drizzle/
  package.json
  tsconfig.json
  .env
```

## Requirements

- Bun runtime
- Postgres database
- Better Auth secrets and provider keys

## Environment Variables

```
DATABASE_URL=postgres://user:pass@host:5432/db
BETTER_AUTH_SECRET=your-generated-secret
BETTER_AUTH_URL=http://localhost:8000
GITHUB_CLIENT_ID=your-github-client-id
GITHUB_CLIENT_SECRET=your-github-client-secret
FRONTEND_URL=http://localhost:3000
NODE_ENV=development
```

## Default Scripts

```
"scripts": {
  "dev": "bun run --watch src/index.ts",
  "build": "bun build src/index.ts --outdir dist --target bun",
  "start": "bun dist/index.js",
  "db:generate": "drizzle-kit generate",
  "db:migrate": "drizzle-kit migrate",
  "db:studio": "drizzle-kit studio"
}
```

## Usage Example

Ask Codex something like:

- "Scaffold a Bun + Elysia backend with Better Auth, Drizzle, Postgres, MCP, and OpenAPI. Use the Elysia Core Backend skill."
- "Add Better Auth + Drizzle + MCP scaffold to a new Elysia server."

## Notes

- The MCP server must be mounted via `elysia-mcp` plugin, not a raw `McpServer` instance.
- CSRF is enforced for non-auth routes when using cookie auth.

## License

MIT
