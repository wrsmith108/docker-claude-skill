---
name: docker
description: Container-based development for isolated, reproducible environments. Use when running npm commands, installing packages, executing code, or managing project dependencies. Trigger phrases include "npm install", "run the build", "start the server", "install package", or any code execution request.
---

# Docker Development Skill

Execute all package installations and code execution inside Docker containers. This keeps the host machine clean and ensures consistent environments across projects.

## ⛔ ENFORCEMENT - READ FIRST

**BLOCKED COMMANDS (never run these on host):**
```
npm install, npm run, npm test, npm exec
npx <anything>
node <script>
tsx <script>
bun <script>
```

**REQUIRED PREFIX for all Node.js commands:**
```bash
docker exec <container-name> <command>
```

**Note:** Use `docker exec` WITHOUT the `-t` flag in Claude Code (non-interactive terminal):
```bash
# ✅ Correct (works in Claude Code)
docker exec my-project-dev-1 npm run build

# ❌ Wrong (fails with "not a TTY")
docker exec -it my-project-dev-1 npm run build
```

## Core Principle

**NEVER run `npm`, `node`, `npx`, `tsx`, or project scripts directly on the host machine.**

Instead, use `docker exec` or ensure the container is running the dev server.

## Pre-Flight Check (MANDATORY)

**Before running ANY npm/node command, Claude Code MUST verify the container is running.**

Run this check first:

```bash
docker ps --filter "name=<project-name>" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

**Expected output:**
```
NAMES                   STATUS          PORTS
my-project-dev-1        Up X minutes    0.0.0.0:3000->3000/tcp
```

**If container is NOT running:**
```bash
# Navigate to project root first
cd /path/to/project

# Start container
docker-compose --profile dev up dev -d

# Verify it started
docker ps --filter "name=<project-name>"
```

**If container shows "Exited":**
```bash
# Check why it exited
docker logs <container-name> --tail 20

# Remove and restart
docker-compose --profile dev down
docker-compose --profile dev up dev -d
```

## Quick Reference

### Check Container Status

```bash
# List running containers for current project
docker ps --filter "name=<project-name>"

# Check container logs
docker logs <container-name> --tail 50

# Check if dev server is responding
curl -s http://localhost:<port> > /dev/null && echo "Server running" || echo "Server not running"
```

### Start/Stop Containers

```bash
# Start development container (from project root)
docker-compose --profile dev up dev -d

# Stop container
docker-compose --profile dev down

# Restart container
docker-compose --profile dev restart dev

# Rebuild after Dockerfile changes
docker-compose --profile dev up dev -d --build
```

### Execute Commands Inside Container

```bash
# Install a package
docker exec <container-name> npm install <package-name>

# Install dev dependency
docker exec <container-name> npm install -D <package-name>

# Run tests
docker exec <container-name> npm test

# Run type checking
docker exec <container-name> npm run typecheck

# Run linting
docker exec <container-name> npm run lint

# Run build
docker exec <container-name> npm run build

# Open shell inside container
docker exec -it <container-name> /bin/sh

# Run any arbitrary command
docker exec <container-name> <command>
```

## When to Use Docker exec

| Operation | Use docker exec? | Reason |
|-----------|------------------|--------|
| `npm install` | ✅ Yes | Packages install in container |
| `npm run dev` | ❌ No | Already running via docker-compose |
| `npm test` | ✅ Yes | Tests run in container environment |
| `npm run build` | ✅ Yes | Build happens in container |
| `git` commands | ❌ No | Git runs on host (manages files) |
| File editing | ❌ No | Volume mount syncs automatically |
| Database migrations | ✅ Yes | Uses container's Node environment |

## Container Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  HOST (macOS/Linux/Windows)                                 │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Docker Container (my-project-dev-1)                │   │
│  │                                                     │   │
│  │  Node 20 Alpine                                     │   │
│  │  └── node_modules/ (container-only)                 │   │
│  │  └── Dev server (port 3000)                         │   │
│  │                                                     │   │
│  │  Volume Mounts:                                     │   │
│  │  └── .:/app (source code sync)                      │   │
│  │  └── node_modules:/app/node_modules (persist deps)  │   │
│  └─────────────────────────────────────────────────────┘   │
│                         │                                   │
│                         ▼                                   │
│                   Port mapped                               │
│                         │                                   │
│                         ▼                                   │
│              http://localhost:<port>                        │
└─────────────────────────────────────────────────────────────┘
```

## Volume Mount Behavior

The `docker-compose.yml` mounts the project directory into the container:

```yaml
volumes:
  - .:/app                           # Source code (synced)
  - /app/node_modules                # Dependencies (container-only)
```

**What this means:**
- Source code changes on host are immediately visible in container
- `node_modules/` in container is separate from any on host
- Hot reload works automatically

## Troubleshooting

### Container Not Running

```bash
# Check if container exists
docker ps -a --filter "name=<project-name>"

# If exited, check why
docker logs <container-name>

# Restart
docker-compose --profile dev up dev -d
```

### Port Already in Use

```bash
# Find what's using the port
lsof -i :<port>

# Kill the process or change port in docker-compose.yml
```

### Module Not Found Errors

```bash
# Rebuild container with fresh dependencies
docker-compose --profile dev down
docker-compose --profile dev build --no-cache dev
docker-compose --profile dev up dev -d
```

### File Changes Not Reflecting

```bash
# Check volume mounts
docker inspect <container-name> | grep -A 10 "Mounts"

# Restart container
docker-compose --profile dev restart dev
```

## Project Configuration Template

When setting up a new project, configure these values:

| Setting | Example Value |
|---------|---------------|
| Container name | `my-project-dev-1` |
| Port | 3000, 4321, 5173, etc. |
| Node version | 20 (Alpine) |
| Dev command | `npm run dev -- --host 0.0.0.0` |
| Volume mounts | `.:/app`, named volume for `node_modules` |

### Environment Variables

Required env vars are loaded from `.env` file via docker-compose.

If a command needs a specific env var:
```bash
docker exec -e MY_VAR=value <container-name> <command>
```

## Best Practices

1. **Always check container status** before running commands
2. **Use `docker exec`** for all npm/node operations
3. **Let volume mounts** handle file syncing (no manual copying)
4. **Rebuild image** after changing `package.json` or `Dockerfile`
5. **Check logs** if something isn't working

## Integration with Claude Code

When Claude Code needs to:

| Task | Action |
|------|--------|
| Install dependency | `docker exec <container> npm install <pkg>` |
| Run tests | `docker exec <container> npm test` |
| Check types | `docker exec <container> npm run typecheck` |
| Build project | `docker exec <container> npm run build` |
| Start dev server | Container already runs it via docker-compose |
| Edit files | Edit directly (volume mount syncs) |
| Git operations | Run on host (not in container) |

## Sample docker-compose.yml

```yaml
version: '3.8'

services:
  dev:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: ${COMPOSE_PROJECT_NAME:-my-project}-dev-1
    ports:
      - "${DEV_PORT:-3000}:3000"
    volumes:
      - .:/app
      - node_modules:/app/node_modules
    environment:
      - NODE_ENV=development
    command: npm run dev -- --host 0.0.0.0
    profiles:
      - dev

volumes:
  node_modules:
```

---

*Last updated: December 2025*
