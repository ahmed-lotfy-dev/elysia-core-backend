# Elysia Backend Scaffold Guide (Better Auth + Drizzle + Postgres + MCP)

This document is a reusable scaffold guide and AI prompt to generate a minimal Elysia backend with Better Auth, Drizzle ORM, Postgres, MCP, OpenAPI, and CORS. It contains no domain-specific business logic.

## Goal

Scaffold a clean Elysia backend with:

- Bun runtime
- Better Auth (auth routes + sessions)
- Drizzle ORM
- Postgres
- MCP endpoint
- OpenAPI docs
- CORS (front-end origin allowlist)
- Security defaults (CORS allowlist, CSRF for non-auth routes, security headers)

No application logic, only base infrastructure.

## Official Scaffold Steps (Required)

Always start with the official Elysia generator and Bun installs. Do not hand-roll the project.

1. `bun create elysia server`
2. `cd server`
3. `bun add @elysiajs/cors @elysiajs/openapi better-auth drizzle-orm pg @modelcontextprotocol/sdk elysia-mcp logixlysia zod`
4. `bun add -d drizzle-kit typescript`
5. `touch drizzle.config.ts` and add the config below
6. Add Drizzle scripts to `package.json` (see Package Scripts section)
7. Generate a `BETTER_AUTH_SECRET` and set it in `.env`: `openssl rand -hex 32`
8. Add social provider env vars (see Environment Variables section)
9. Generate migrations: `bun run db:generate`
10. Run migrations: `bun run db:migrate`
11. Start dev server: `bun run dev`
12. Configure security env vars (see Security Defaults section)

Then apply the file structure and code below.

## Folder Structure

```
server/
  src/
    index.ts
    auth.ts
    middleware/
      auth-middleware.ts
      logger.ts
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

## Dependencies

Install (via `bun add`, not by manually editing `package.json`):

- elysia
- @elysiajs/cors
- @elysiajs/openapi
- better-auth
- drizzle-orm
- drizzle-kit
- pg
- @modelcontextprotocol/sdk
- elysia-mcp
- logixlysia
- zod
- node:crypto (builtin, used by CSRF middleware)

## Minimal DB Layer

**src/db/db.ts**

```ts
import { Pool } from "pg";
import { drizzle } from "drizzle-orm/node-postgres";

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

export const db = drizzle(pool);
```

**src/db/auth-schema.ts**

```ts
import { pgTable, text, timestamp, boolean } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: text("id").primaryKey(),
  name: text("name"),
  email: text("email").unique(),
  emailVerified: boolean("email_verified"),
  image: text("image"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});

export const sessions = pgTable("sessions", {
  id: text("id").primaryKey(),
  token: text("token").notNull(),
  userId: text("user_id").notNull(),
  expiresAt: timestamp("expires_at").notNull(),
});

export const accounts = pgTable("accounts", {
  id: text("id").primaryKey(),
  userId: text("user_id").notNull(),
  provider: text("provider").notNull(),
  providerAccountId: text("provider_account_id").notNull(),
});

export const verificationTokens = pgTable("verification_tokens", {
  identifier: text("identifier").notNull(),
  token: text("token").notNull(),
  expiresAt: timestamp("expires_at").notNull(),
});
```

## Better Auth Setup

**src/auth.ts**

```ts
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { db } from "./db/db";
import {
  users,
  sessions,
  accounts,
  verificationTokens,
} from "./db/auth-schema";

export const auth = betterAuth({
  database: drizzleAdapter(db, {
    provider: "pg",
    schema: { users, sessions, accounts, verificationTokens },
  }),
  socialProviders: {
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    },
  },
});
```

## Drizzle Config

**drizzle.config.ts**

```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/db/auth-schema.ts",
  out: "./drizzle",
  dialect: "postgresql",
  dbCredentials: {
    connectionString: process.env.DATABASE_URL ?? "",
  },
});
```

## MCP Setup

**src/mcp/index.ts**

```ts
import { Elysia } from "elysia";
import { mcp } from "elysia-mcp";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

const mcpPlugin = new Elysia({ name: "mcp-server" })
  .onRequest(({ request }) => {
    // Fix for clients that don't send proper Accept headers (e.g., OpenCode, some MCP clients)
    // The MCP SDK requires both application/json and text/event-stream
    const acceptHeader = request.headers.get("Accept") ?? "";
    const hasJson = acceptHeader.includes("application/json");
    const hasSse = acceptHeader.includes("text/event-stream");
    if (!hasJson || !hasSse) {
      const newHeaders = new Headers(request.headers);
      newHeaders.set("Accept", "application/json, text/event-stream");
      Object.defineProperty(request, "headers", {
        value: newHeaders,
        writable: false,
      });
    }
  })
  .use(
    mcp({
      stateless: true,
      enableJsonResponse: true,
      serverInfo: {
        name: "Base MCP",
        version: "1.0.0",
      },
      capabilities: {
        tools: {},
        resources: {},
        prompts: {},
        logging: {},
      },
      setupServer: async (server: McpServer) => {
        server.registerTool(
          "health",
          {
            description: "Returns API health status",
            inputSchema: z.object({}),
          },
          async () => ({
            content: [{ type: "text", text: "ok" }],
          }),
        );
      },
    })
  );

export { mcpPlugin };
```

**Important**: Do not `.use(new McpServer(...))` directly. `McpServer` is **not** an Elysia plugin and will throw:
`TypeError: undefined is not an object (evaluating 'plugin.promisedModules.size')`.
Always wrap it with the `elysia-mcp` plugin as shown above and `.use(mcpPlugin)`.

**Client Compatibility Note**: The `stateless: true` and `enableJsonResponse: true` options make the MCP server compatible with HTTP-based MCP clients that may not support full SSE streaming. The `onRequest` handler patches the Accept header to ensure compatibility with clients like OpenCode that only send `Accept: application/json`.

## Elysia App Bootstrap

**src/index.ts**

```ts
import { Elysia } from "elysia";
import { cors } from "@elysiajs/cors";
import { openapi } from "@elysiajs/openapi";
import logixlysia from "logixlysia";
import { auth } from "./auth";
import { mcpPlugin } from "./mcp";
import { logger } from "./middleware/logger";
import { csrfProtection } from "./middleware/csrf";
import { securityHeaders } from "./middleware/security-headers";

const app = new Elysia()
  .use(logger)
  .use(securityHeaders)
  .use(
    openapi({
      path: "/docs",
      documentation: { info: { title: "API", version: "1.0.0" } },
    }),
  )
  .use(
    cors({
      origin: (request) => {
        const origin = request.headers.get("origin");
        if (!origin) return true;
        return trustedOrigins.includes(origin);
      },
      credentials: true,
      methods: ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
      allowedHeaders: ["Content-Type", "Authorization", "Cookie", "Accept"],
    }),
  )
  .use(csrfProtection)
  .mount(auth.handler)
  .get("/health", () => ({ status: "ok" }))
  .get("/", () => "Elysia backend running")
  .use(mcpPlugin);

app.listen(process.env.PORT || 8000);
```

## Logger Middleware

**src/middleware/logger.ts**

```ts
import logixlysia from "logixlysia";

export const logger = logixlysia({
  config: {
    showStartupMessage: true,
    startupMessageFormat: "simple",
    timestamp: { translateTime: "yyyy-mm-dd HH:MM:ss.SSS" },
    ip: true,
  },
});
```

## Security Defaults (Required)

Add these two middleware files and wire them into `src/index.ts` as shown above.

**src/lib/env.ts**

```ts
function parseList(value?: string): string[] {
  if (!value) return [];
  return value
    .split(",")
    .map((item) => item.trim())
    .filter(Boolean);
}

export const trustedOrigins = parseList(process.env.FRONTEND_URL);
```

**src/middleware/security-headers.ts**

```ts
import Elysia from "elysia";

const CSP_NON_DOCS =
  "default-src 'none'; frame-ancestors 'none'; base-uri 'none'; form-action 'none'";

export const securityHeaders = new Elysia({ name: "security-headers" }).onAfterHandle(
  ({ request, set }) => {
    set.headers["X-Content-Type-Options"] = "nosniff";
    set.headers["Referrer-Policy"] = "no-referrer";
    set.headers["X-Frame-Options"] = "DENY";
    set.headers["Permissions-Policy"] =
      "camera=(), microphone=(), geolocation=(), payment=(), usb=()";

    const path = new URL(request.url).pathname;
    if (!path.startsWith("/docs")) {
      set.headers["Content-Security-Policy"] = CSP_NON_DOCS;
    }
  },
);
```

**src/middleware/csrf.ts**

```ts
import Elysia from "elysia";
import { randomBytes, timingSafeEqual } from "node:crypto";

const SAFE_METHODS = new Set(["GET", "HEAD", "OPTIONS"]);
const CSRF_COOKIE = "csrf_token";
const CSRF_HEADER = "x-csrf-token";

function parseCookies(header: string | null): Record<string, string> {
  if (!header) return {};
  return header.split(";").reduce<Record<string, string>>((acc, part) => {
    const [rawKey, ...rest] = part.trim().split("=");
    if (!rawKey) return acc;
    acc[rawKey] = decodeURIComponent(rest.join("="));
    return acc;
  }, {});
}

function tokensMatch(a: string, b: string): boolean {
  if (a.length !== b.length) return false;
  return timingSafeEqual(Buffer.from(a), Buffer.from(b));
}

function shouldUseSecureCookies(): boolean {
  return (
    process.env.BETTER_AUTH_URL?.startsWith("https://") &&
    process.env.NODE_ENV === "production"
  );
}

export const csrfProtection = new Elysia({ name: "csrf-protection" })
  .get("/api/csrf", ({ set }) => {
    const token = randomBytes(32).toString("hex");
    const cookieParts = [
      `${CSRF_COOKIE}=${encodeURIComponent(token)}`,
      "Path=/",
      "SameSite=Lax",
      "HttpOnly",
    ];

    if (shouldUseSecureCookies()) {
      cookieParts.push("Secure");
    }

    set.headers["Set-Cookie"] = cookieParts.join("; ");
    set.headers["Cache-Control"] = "no-store";

    return { csrfToken: token };
  })
  .onBeforeHandle(({ request, set }) => {
    if (SAFE_METHODS.has(request.method)) return;

    const url = new URL(request.url);
    if (url.pathname.startsWith("/api/auth") || url.pathname === "/api/csrf") {
      return;
    }

    const authHeader =
      request.headers.get("authorization") || request.headers.get("Authorization");
    if (authHeader?.startsWith("Bearer ")) {
      return;
    }

    const cookies = parseCookies(
      request.headers.get("cookie") || request.headers.get("Cookie"),
    );
    const csrfCookie = cookies[CSRF_COOKIE];
    const csrfHeader = request.headers.get(CSRF_HEADER);

    if (!csrfCookie || !csrfHeader || !tokensMatch(csrfCookie, csrfHeader)) {
      set.status = 403;
      return { error: "Forbidden: CSRF token missing or invalid" };
    }
  });
```

**Frontend note**: For state-changing requests, the client must fetch `/api/csrf` and include `X-CSRF-Token` in non-GET/HEAD/OPTIONS requests.

**Better Auth note**: Better Auth protects its own `/api/auth/*` routes with CSRF/origin checks, but non-auth routes still need CSRF enforcement if cookie auth is used.

## Environment Variables

**.env**

```
DATABASE_URL=postgres://user:pass@host:5432/db
BETTER_AUTH_SECRET=your-generated-secret
BETTER_AUTH_URL=http://localhost:8000
GITHUB_CLIENT_ID=your-github-client-id
GITHUB_CLIENT_SECRET=your-github-client-secret
FRONTEND_URL=http://localhost:3000
NODE_ENV=development
```

## Package Scripts

**package.json**

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

## Run Locally (Required)

1. Set your `.env` values (database + Better Auth + social provider secrets).
2. Generate migrations: `bun run db:generate`
3. Run migrations: `bun run db:migrate`
4. Start dev server: `bun run dev`

## AI Prompt Template

```
Scaffold a Bun + Elysia backend with:
- Better Auth
- Drizzle ORM
- Postgres
- MCP endpoint
- OpenAPI docs
- CORS
No business logic, just base structure.

Create this structure:
server/src/index.ts
server/src/auth.ts
server/src/db/db.ts
server/src/db/auth-schema.ts
server/src/mcp/index.ts
server/drizzle.config.ts
server/.env
server/package.json

Implement:
- Elysia app with /docs, /health, /, /mcp
- Better Auth mounted with Drizzle adapter
- Minimal auth schema: users, sessions, accounts, verificationTokens
- Postgres connection with drizzle
- Drizzle config and db scripts
- Social auth provider config (GitHub example)
- Run steps (generate, migrate, dev)
- Logixlysia logger
- MCP plugin via elysia-mcp
- Security defaults (CORS allowlist, CSRF protection, security headers)

Return full file contents for each file.
```
