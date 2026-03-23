# MEMORY.md — Long-Term Memory

## About Me
- I am Darcy, managed by Claude (MiniMax-M2.7) in the OpenClaw workspace
- I run as the "claude" agent in OpenClaw

## Mission Control (MC) — Primary Responsibility
I manage the **Mission Control** codebase — a self-hosted dashboard for orchestrating multiple OpenClaw AI agent deployments.

**Location:** `/opt/openclaw/.openclaw/mc-codebase`

**Architecture:**
- Two Docker containers: Next.js 15 frontend (port 3000) + FastAPI backend (port 8000)
- Frontend proxies API calls through `/api/proxy/[...path]` to backend
- Backend maintains WebSocket connections to OpenClaw instances and fans out SSE to browser
- OpenClaw v2 is WebSocket-first; REST API is minimal (mostly `/health`)

**Key Commands:**
- Rebuild/restart MC: `cd /opt/openclaw/.openclaw/mc-codebase && docker compose build && docker compose up -d`
- Restart backend only: `cd /opt/openclaw/.openclaw/mc-codebase && docker compose restart backend`
- Check container health: `docker ps | grep mc-codebase`
- View backend logs: `docker logs mc-codebase-backend-1`

**Auth Flow:**
- POST `/auth/login` with `{ password }` → backend verifies bcrypt hash → issues JWT as httpOnly cookie
- Single admin password (set via `MC_ADMIN_PASSWORD` env var)
- OpenClaw instances require device identity approval: `openclaw devices approve <requestId>`

**Important Context:**
- Backend container can get stuck (marked "unhealthy") causing 500 errors on login
- Restarting the backend container (`docker compose restart backend`) fixes it
- When OpenClaw gateway restarts, MC may need its backend restarted too

**Development:**
- Run `/claude/prime` command when starting new sessions to update codebase understanding
- `make test-backend` for pytest, `make test-frontend` for vitest, `make test-e2e` for playwright

## OpenClaw Gateway
- Running on `claw02` at `ws://127.0.0.1:18789` (local loopback)
- Token auth enabled
- Access via: `http://10.0.0.167:18789/` (LAN) or `https://openclaw.area4.net/` (via proxy)
- Gateway was restarted during troubleshooting at ~14:05 on 2026-03-22

## User
- **Mark Crouch** (spotthegeek, ID: 5376107144)
- Manages OpenClaw deployments across multiple Proxmox LXC containers
- Runs Mission Control to unify management of all agents
