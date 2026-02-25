# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Moltworker is a Cloudflare Worker that runs [OpenClaw](https://github.com/openclaw/openclaw) in a Cloudflare Sandbox container. It proxies HTTP/WebSocket requests to the OpenClaw gateway, provides an admin UI for device management, and optionally persists data to R2.

**Stack:** Hono (web framework), React 19 (admin UI), Vite (bundling), Vitest (testing), oxlint/oxfmt (linting/formatting), Cloudflare Workers + Sandbox Containers.

## Commands

```bash
npm test                # Run unit tests (vitest)
npm run test:watch      # Tests in watch mode
npm run test:coverage   # Tests with coverage report
npm run typecheck       # TypeScript strict check (tsc --noEmit)
npm run lint            # Lint with oxlint
npm run lint:fix        # Auto-fix lint issues
npm run format          # Format with oxfmt
npm run format:check    # Check formatting
npm run build           # Vite build (worker + client)
npm run start           # Local dev with wrangler dev
npm run deploy          # Build + deploy to Cloudflare
```

Run a single test file: `npx vitest run src/gateway/env.test.ts`

## Architecture

```
Browser → Cloudflare Worker (src/index.ts)
            │
            ├─ Public routes (no auth): /sandbox-health, /api/status, /cdp/*
            ├─ Protected routes (CF Access JWT):
            │   ├─ /_admin/* → React SPA (device management)
            │   ├─ /api/admin/* → Device management API
            │   └─ /debug/* → Debug endpoints (when DEBUG_ROUTES=true)
            └─ Catch-all → Proxy to OpenClaw gateway in sandbox container (port 18789)
```

The middleware chain in `src/index.ts` is order-dependent: request logging → sandbox init → **public routes** → config validation → CF Access auth → **protected routes** → catch-all proxy.

### Key Components

- **`src/index.ts`** — Main Hono app, route mounting, WebSocket proxying with message transformation
- **`src/gateway/`** — Container process lifecycle (`process.ts`), environment variable mapping (`env.ts`), R2 mounting (`r2.ts`), backup sync (`sync.ts`)
- **`src/auth/`** — CF Access JWT verification and Hono middleware
- **`src/routes/`** — Route handlers (api, admin-ui, public, debug, cdp)
- **`src/client/`** — React admin UI (built by Vite, excluded from unit tests)
- **`Dockerfile`** — Container image: `cloudflare/sandbox` base + Node 22 + rclone + OpenClaw
- **`start-openclaw.sh`** — Container startup: R2 restore → onboard → config patch → launch gateway
- **`wrangler.jsonc`** — Worker + container + R2 + browser binding config

### Environment Variable Flow

Worker secrets (set via `wrangler secret put`) are mapped to container env vars in `src/gateway/env.ts` via `buildEnvVars()`. Some are renamed: `MOLTBOT_GATEWAY_TOKEN` → `OPENCLAW_GATEWAY_TOKEN`, `DEV_MODE` → `OPENCLAW_DEV_MODE`.

## Testing

Tests use Vitest with colocated test files (`src/**/*.test.ts`). Client code (`src/client/`) is excluded.

Shared test utilities in `src/test-utils.ts`: `createMockEnv()`, `createMockProcess()`, `createMockSandbox()`, `suppressConsole()`.

Pattern: mock the sandbox and its processes using vitest's `vi.fn()`, then assert on behavior.

## Code Conventions

- TypeScript strict mode; explicit types on function signatures
- Hono route handlers should be thin — extract logic to `gateway/` or `routes/` modules
- Use Hono's `c.json()`, `c.html()` for responses
- Legacy naming: `clawdbot` → `openclaw` (both supported in process detection for backward compat)

## Important Gotchas

- **CLI commands are slow:** OpenClaw CLI takes 10-15s. Always use `waitForProcess(proc, 20000)`.
- **WebSocket locally broken:** `wrangler dev` can't properly proxy WebSocket through sandbox. Deploy to Cloudflare for full testing.
- **R2 is backup/restore, not live-mount:** Data syncs every 5 minutes via cron in the container. Use `rsync -r --no-times` (s3fs doesn't support timestamps).
- **Docker cache bust:** When changing `start-openclaw.sh`, bump the version comment in the Dockerfile.
- **Token injection:** CF Access redirects strip query params, so the worker injects `?token=` server-side for WebSocket connections.
- **Process status unreliable:** `proc.status` may not update immediately. Verify success via expected output/logs, not status.

## Local Development

```bash
npm install
cp .dev.vars.example .dev.vars   # Edit with your ANTHROPIC_API_KEY
npm run start
```

Set `DEV_MODE=true` in `.dev.vars` to skip CF Access auth and device pairing. Set `DEBUG_ROUTES=true` for `/debug/*` endpoints.
