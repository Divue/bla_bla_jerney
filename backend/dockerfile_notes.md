# 🐳 Backend Dockerfile Notes
### Jerney Blog Platform — Node.js Production Container 

---

## What This Dockerfile Does

Builds the Node.js backend API into a **secure, minimal production container** using a two-stage build — but with a different approach than the frontend.

```
Stage 1 (build)           Stage 2 (production)
─────────────────         ────────────────────
node:20-alpine            node:20-alpine
npm ci --only=production  patch OS
(install prod deps only)  create non-root user
                          install dumb-init
                    ────► copy node_modules + src/
                          run as appuser
                          ~node src/index.js
```

> **Why two stages if there's no build step?**
> The backend is pure Node.js — no TypeScript compilation, no bundling needed. But the two-stage pattern is still used to keep the **dependency installation layer separate and cacheable**, and to ensure `node_modules` is clean and production-only before it's copied over.

---

## How This Differs From the Frontend Dockerfile

| | Frontend | Backend |
|---|---|---|
| Stage 1 base | `node:20-alpine` | `node:20-alpine` |
| Stage 1 purpose | Full build (`npm run build`) | Install prod deps only |
| Stage 2 base | `nginx:1.27-alpine` | `node:20-alpine` |
| What crosses stages | `/app/dist` (compiled static files) | `/app/node_modules` + `src/` |
| Runtime | Nginx (serves static files) | Node.js (runs live server) |
| Non-root setup | `chown` to `nginx` user (pre-existing) | Create custom `appuser` from scratch |
| Signal handling | Nginx handles it natively | `dumb-init` needed |

---

## Stage 1 — Build Stage (Line by Line)

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci --only=production && npm cache clean --force
```

### `npm ci --only=production`

The key difference from the frontend. `--only=production` installs **only dependencies listed under `dependencies`** in `package.json` — it skips everything in `devDependencies` (testing frameworks, linters, TypeScript, build tools).

```
Without --only=production:    With --only=production:
node_modules/                 node_modules/
  express/          ✅          express/          ✅
  pg/               ✅          pg/               ✅
  eslint/           ❌          (skipped)
  jest/             ❌          (skipped)
  nodemon/          ❌          (skipped)
```

Result: `node_modules` is much smaller — only what the app actually needs to run. This is then copied into Stage 2.

### Why still use two stages for this?

It's about **layer cache isolation**. Stage 1's only job is producing a clean, production-only `node_modules`. If you did this in Stage 2 directly, the `apk upgrade`, user creation, and `dumb-init` install would all be coupled with the dependency install — any change to one invalidates the others. Separating it gives more granular caching.

---

## Stage 2 — Production Stage (Line by Line)

```dockerfile
FROM node:20-alpine AS production
```

Fresh Alpine-based Node.js image. Node.js stays in Stage 2 (unlike frontend where it was replaced by Nginx) because the backend is a live Node.js server — it needs Node to run.

---

```dockerfile
RUN apk update && apk upgrade --no-cache
```

Patches OS-level vulnerabilities in the base image. Same reasoning as the frontend — the base image may be weeks old with known CVEs. This is done **before** setting up users and copying files, so it's a separate cacheable layer at the top.

Notice: this is in Stage 2, not Stage 1. Stage 1's patched OS gets discarded anyway — you only care about the OS in the final image.

---

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
```

Creates a **custom non-root user** from scratch. The backend doesn't have a pre-existing service user like Nginx does (`nginx` user), so one is created explicitly.

Breaking down the flags:

| Command | Flag | Meaning |
|---|---|---|
| `addgroup` | `-S` | System group (no login, no home dir, low GID) |
| `adduser` | `-S` | System user (no password, no login shell) |
| `adduser` | `-G appgroup` | Add user to the group just created |

`appuser` ends up as a locked, shell-less, system account — the minimum possible identity for running the app. It can't log in, can't sudo, can't do anything except what the app itself does.

---

```dockerfile
RUN apk add --no-cache dumb-init=1.2.5-r3
```

Installs `dumb-init` — a tiny process supervisor. The version is **pinned exactly** (`1.2.5-r3`) for reproducibility.

### What is dumb-init and why is it needed?

This solves the **PID 1 problem** in containers.

In a normal Linux system, PID 1 is `init` (or `systemd`). Its job includes:
- **Signal forwarding** — when you `kill` a process, it forwards the signal to child processes
- **Zombie reaping** — cleaning up finished child processes

When a container runs `node src/index.js` directly, Node.js becomes PID 1. But Node.js is not designed to be PID 1. Problems:

```
Without dumb-init:
  kubectl delete pod   →   SIGTERM sent to PID 1 (Node.js)
                       →   Node.js doesn't handle it properly
                       →   Waits 30s for grace period
                       →   Kubernetes force-kills (SIGKILL)
                       →   In-flight requests dropped, DB connections not closed

With dumb-init:
  kubectl delete pod   →   SIGTERM sent to PID 1 (dumb-init)
                       →   dumb-init forwards SIGTERM to Node.js
                       →   Node.js shuts down gracefully
                       →   Connections closed, requests finished
                       →   Clean exit ✅
```

`dumb-init` is the correct, minimal PID 1 for containers. It's 35KB — adds essentially nothing to image size.

---

```dockerfile
WORKDIR /app
```

Sets the working directory. All subsequent paths are relative to `/app`.

---

```dockerfile
COPY --from=build /app/node_modules ./node_modules
COPY src/ ./src/
```

Two separate `COPY` instructions — another intentional caching decision:

- `node_modules` — only changes when `package.json` changes (rare)
- `src/` — changes on every code edit (frequent)

Keeping them separate means editing source code doesn't invalidate the `node_modules` layer.

**What is NOT copied:**
- `package.json` / `package-lock.json` — not needed at runtime
- `devDependencies` — excluded during Stage 1's `npm ci --only=production`
- Tests, docs, config files — not needed to run the server
- `.env` files — secrets come from Kubernetes Secrets, not baked into the image

---

```dockerfile
RUN chown -R appuser:appgroup /app
```

Transfers ownership of the entire `/app` directory (including `node_modules` and `src/`) to `appuser:appgroup`. Must be done **as root** (before `USER appuser`) — once you switch users you can't change ownership of files you don't own.

---

```dockerfile
USER appuser
```

Switches from root to `appuser` for all subsequent instructions and at container runtime. From this point, if an attacker exploits a vulnerability in the app or any npm package, they land as `appuser` — a locked system account with no shell, no sudo, and no privileges.

This maps directly to `runAsUser: 1000` and `runAsNonRoot: true` in the Kubernetes securityContext.

---

```dockerfile
EXPOSE 5000
```

Documents the port the app listens on. Documentation only — doesn't open any actual ports. The Kubernetes Service's `targetPort: 5000` is what actually routes traffic here.

---

```dockerfile
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "src/index.js"]
```

### ENTRYPOINT vs CMD — the key difference

| | `ENTRYPOINT` | `CMD` |
|---|---|---|
| Purpose | The fixed executable — always runs | The default arguments — can be overridden |
| Override | Requires `--entrypoint` flag | Simply pass new command after image name |
| Combination | `ENTRYPOINT` + `CMD` = ENTRYPOINT runs with CMD as its arguments |

Combined, these two lines execute:
```bash
dumb-init -- node src/index.js
```

- `dumb-init` is the process that becomes PID 1
- `--` signals end of dumb-init's own flags
- `node src/index.js` is passed to dumb-init, which spawns it as a child process and supervises it

If the container is started with a different command (e.g. for debugging):
```bash
docker run jerney-backend node --inspect src/index.js
# Runs: dumb-init -- node --inspect src/index.js  ← entrypoint stays
```

---

## The Full Flow Visualised

```
docker build -f backend/Dockerfile .
                    │
         ┌──────────▼──────────────┐
         │     STAGE 1: build      │
         │     node:20-alpine      │
         │                         │
         │  npm ci --only=prod     │
         │  (no devDependencies)   │
         │       │                 │
         │  /app/node_modules ─────┼──┐
         └─────────────────────────┘  │ COPY --from=build
                                      │
         ┌──────────▼──────────────┐  │
         │  STAGE 2: production    │◄─┘
         │  node:20-alpine         │
         │                         │
         │  apk upgrade            │
         │  adduser appuser        │
         │  install dumb-init      │
         │  COPY src/              │
         │  chown → appuser        │
         │  USER appuser           │
         │  EXPOSE 5000            │
         │  ENTRYPOINT dumb-init   │
         │  CMD node src/index.js  │
         └─────────────────────────┘
                    │
            Final image
       (no devDeps, non-root,
        proper signal handling)
```

---

## Key Security Decisions Summary

| Decision | Why |
|---|---|
| `apk upgrade` in Stage 2 | Patches OS CVEs in the final image (Stage 1 is discarded) |
| `npm ci --only=production` | Excludes devDependencies — smaller image, smaller attack surface |
| Custom `appuser` | No pre-existing service user in Node images — create a locked, shell-less account |
| `adduser -S` (system user) | No password, no login shell — can't be used interactively |
| `dumb-init` as PID 1 | Proper signal forwarding → graceful shutdown → no dropped requests |
| Pinned `dumb-init=1.2.5-r3` | Reproducible — won't silently get a different version |
| `COPY src/` separately from `node_modules` | Better layer cache — code changes don't reinstall deps |
| `chown` before `USER appuser` | Must set permissions as root before giving up root |
| `USER appuser` | Least privilege — compromised container can't escalate |
| `ENTRYPOINT` + `CMD` split | `dumb-init` always runs (can't be accidentally overridden), command is flexible |

---

## Frontend vs Backend — Side by Side

```
Frontend                          Backend
────────                          ───────
Stage 1: npm run build            Stage 1: npm ci --only=prod
Stage 2: nginx:1.27-alpine        Stage 2: node:20-alpine
Copies:  /app/dist (static files) Copies:  node_modules + src/
User:    nginx (pre-existing)     User:    appuser (custom created)
PID 1:   nginx (handles signals)  PID 1:   dumb-init (signal proxy)
Port:    8080                     Port:    5000
Runtime: Static file serving      Runtime: Live Node.js process
```

---


**Q: Why use two stages if the backend has no build step?**
> The backend doesn't compile TypeScript or bundle files, but Stage 1 is still useful for isolation. It installs only production dependencies (`--only=production`) cleanly, separate from the final image setup. This means the dep install layer is independently cached — code changes in `src/` don't trigger a full `npm ci` rerun. It also keeps the pattern consistent and makes the Dockerfile easier to extend later (e.g. if you add TypeScript).

**Q: What is `dumb-init` and why is it needed?**
> `dumb-init` is a minimal init process designed to run as PID 1 in containers. Node.js isn't designed to be PID 1 — it doesn't properly forward OS signals (like `SIGTERM` from `kubectl delete pod`) to child processes and doesn't reap zombie processes. `dumb-init` sits as PID 1, correctly forwards signals to Node.js, and allows Node.js to shut down gracefully — closing DB connections and finishing in-flight requests instead of being force-killed.

**Q: What is the difference between `ENTRYPOINT` and `CMD`?**
> `ENTRYPOINT` defines the fixed executable that always runs — it can only be overridden with `--entrypoint`. `CMD` provides default arguments to `ENTRYPOINT` and can be overridden by passing a command after the image name. Using both together (`ENTRYPOINT ["dumb-init", "--"]` + `CMD ["node", "src/index.js"]`) means dumb-init always runs as PID 1 (can't be accidentally skipped) while the node command can be swapped for debugging.

**Q: Why create a custom user instead of using an existing one?**
> Unlike Nginx images which ship with a pre-configured `nginx` user, Node.js base images don't have a built-in service user. Creating `appuser` with `-S` (system user flags) produces a locked account: no password, no login shell, no home directory — the minimum identity needed to run the process. This is more explicit and more secure than repurposing any existing user.

**Q: Why is `apk upgrade` in Stage 2 and not Stage 1?**
> Stage 1 is discarded at the end of the build — only `node_modules` crosses into Stage 2. Patching the OS in Stage 1 patches a filesystem that gets thrown away. The OS that actually ships in the final image is Stage 2's OS, so that's where the upgrade belongs.