# 🐳 Frontend Dockerfile Notes
### Jerney Blog Platform — Multi-Stage Docker Build 

---

## What This Dockerfile Does

Builds the React frontend into a **tiny, secure, production-ready Nginx container** using a two-stage build.

```
Stage 1 (build)      Stage 2 (production)
─────────────────    ────────────────────
node:20-alpine       nginx:1.27-alpine
Install deps         Copy ONLY the built files
npm run build   ───► /usr/share/nginx/html
~400MB               ~25MB  ✅
```

The final image contains **zero Node.js, zero source code, zero dev dependencies** — just Nginx serving static files.

---

## Multi-Stage Build — The Core Concept

Without multi-stage builds, your production image would contain Node.js, all `node_modules`, source files, test files — everything used to *build* the app. That's wasteful and a security risk.

Multi-stage solves this by using **separate `FROM` statements**. Each stage is a fresh environment. You cherry-pick only what you need from previous stages using `COPY --from=`.

```dockerfile
FROM node:20-alpine AS build       # Stage 1 — named "build"
...build stuff...

FROM nginx:1.27-alpine AS production  # Stage 2 — fresh slate
COPY --from=build /app/dist ...    # Pull ONLY the output from stage 1
```

The final image is built from the **last `FROM`** only. Stage 1 is discarded entirely.

---

## Stage 1 — Build Stage (Line by Line)

```dockerfile
FROM node:20-alpine AS build
```
- `node:20-alpine` — Node.js 20 on Alpine Linux. Alpine is a minimal Linux distro (~5MB vs ~150MB for Ubuntu). Used here because we only need it temporarily to run the build.
- `AS build` — names this stage so Stage 2 can reference it with `--from=build`.

---

```dockerfile
RUN apk update && apk upgrade --no-cache
```
- `apk` — Alpine's package manager (like `apt` for Ubuntu).
- `apk update` — refreshes the package index.
- `apk upgrade` — installs security patches for any vulnerable OS packages in the base image.
- `--no-cache` — doesn't store the package index on disk. Keeps the layer smaller.

> **Why patch in build stage?** The base image `node:20-alpine` may have been published weeks ago with known CVEs. Patching at build time ensures you're not shipping vulnerable OS packages. Trivy (from the CI/CD pipeline) would catch these if you skipped this step.

---

```dockerfile
WORKDIR /app
```
Sets the working directory. All subsequent commands run from `/app`. Creates the directory if it doesn't exist. Cleaner than running `RUN mkdir /app && cd /app`.

---

```dockerfile
COPY package.json package-lock.json* ./
RUN npm ci && npm cache clean --force
COPY . .
```

This ordering is **deliberate and important** — it's the **dependency caching trick**.

Docker builds images layer by layer. Each instruction creates a layer. If a layer's input hasn't changed, Docker reuses the cached layer instead of rebuilding it.

```
Layer 1: COPY package.json package-lock.json  ← only changes when deps change
Layer 2: RUN npm ci                            ← cached if layer 1 didn't change ✅
Layer 3: COPY . .                              ← changes on every code edit
Layer 4: RUN npm run build                     ← reruns only if layer 3 changed
```

If you did `COPY . .` first, every single code change would invalidate the npm install layer — re-downloading all packages every build. Separating the `package.json` copy means `npm ci` is only re-run when dependencies actually change.

`npm cache clean --force` — removes npm's download cache after installing. No point keeping it in the image layer.

---

```dockerfile
RUN npm run build
```
Runs the build script from `package.json` (typically `vite build` or `react-scripts build`). Outputs compiled, minified static files to `/app/dist`.

---

## Stage 2 — Production Stage (Line by Line)

```dockerfile
FROM nginx:1.27-alpine AS production
```
Completely fresh base image. Everything from Stage 1 is gone. This image starts with only Nginx — no Node, no source code, no `node_modules`.

`1.27-alpine` — pins to a specific Nginx version. Never use `latest` in production — it's non-reproducible and breaks unpredictably.

---

```dockerfile
RUN rm -rf /etc/nginx/conf.d/default.conf /usr/share/nginx/html/*
```
Removes Nginx's default config and default HTML (the "Welcome to nginx!" page). You're replacing both with your own config and built app. Removing the defaults prevents accidental exposure of the default page.

---

```dockerfile
COPY nginx.conf /etc/nginx/conf.d/default.conf
```
Installs your custom Nginx config. This file typically configures:
- Which port to listen on (8080, not 80 — because non-root can't bind to ports below 1024)
- SPA routing: redirect all 404s to `index.html` so React Router works
- Response headers (cache control, security headers)
- Gzip compression

---

```dockerfile
COPY --from=build /app/dist /usr/share/nginx/html
```
The multi-stage magic. Copies the compiled static files from Stage 1's `/app/dist` into Stage 2's web root. This is the **only thing** that crosses the stage boundary.

`/usr/share/nginx/html` is where Nginx serves static files from by default.

---

```dockerfile
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    chown -R nginx:nginx /var/cache/nginx && \
    chown -R nginx:nginx /var/log/nginx && \
    touch /var/run/nginx.pid && \
    chown -R nginx:nginx /var/run/nginx.pid
```

Grants the `nginx` user ownership of every directory it needs to write to at runtime:

| Directory | Why nginx needs it |
|---|---|
| `/usr/share/nginx/html` | Serve static files from here |
| `/var/cache/nginx` | Nginx writes temporary cache files here |
| `/var/log/nginx` | Nginx writes access and error logs here |
| `/var/run/nginx.pid` | Nginx writes its process ID here on startup |

By default these are owned by `root`. The next step switches the user to `nginx` — so these permissions must be set first, otherwise Nginx will crash at startup with "permission denied".

---

```dockerfile
USER nginx
```
Switches the running user from `root` to `nginx` (a non-privileged user). Everything after this line and at container runtime runs as `nginx`.

This is a **critical security practice**. If an attacker exploits a vulnerability in Nginx, they land as the `nginx` user — not root. They can't install packages, modify system files, or escape the container as easily.

> This maps directly to the `runAsUser: 101` and `runAsNonRoot: true` settings in the Kubernetes securityContext — K8s enforces this at the Pod level too.

---

```dockerfile
EXPOSE 8080
```
Documents that the container listens on port 8080. This is **documentation only** — it doesn't actually open any ports. The actual port binding happens in the Kubernetes Service (`targetPort: 8080`).

Port 8080 (not 80) because non-root users can't bind to privileged ports below 1024 on Linux.

---

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```
The command that runs when the container starts.

- `nginx` — starts the Nginx web server
- `-g "daemon off;"` — tells Nginx to run in the **foreground** instead of daemonising (running as a background process)

This is required in containers. Docker/Kubernetes tracks processes by watching PID 1. If Nginx daemonises, PID 1 exits and the container stops — even though Nginx is technically still running in the background. `daemon off` keeps it as PID 1.

---

## The Full Flow Visualised

```
docker build -f frontend/Dockerfile .
                    │
         ┌──────────▼──────────┐
         │   STAGE 1: build    │
         │   node:20-alpine    │
         │                     │
         │  patch OS           │
         │  npm ci             │
         │  npm run build      │
         │       │             │
         │  /app/dist/ ────────┼──┐
         └─────────────────────┘  │ COPY --from=build
                                  │
         ┌──────────▼──────────┐  │
         │ STAGE 2: production │◄─┘
         │ nginx:1.27-alpine   │
         │                     │
         │  remove defaults    │
         │  install nginx.conf │
         │  chown → nginx user │
         │  USER nginx         │
         │  EXPOSE 8080        │
         │  CMD nginx          │
         └─────────────────────┘
                    │
            Final image ~25MB
       (no Node, no source, no deps)
```

---

## Key Security Decisions Summary

| Decision | Why |
|---|---|
| Alpine base images | Minimal attack surface — fewer packages = fewer CVEs |
| `apk upgrade` at build time | Patches OS vulnerabilities in the base image |
| Multi-stage build | Production image has zero build tools — nothing to exploit |
| Pinned versions (`node:20`, `nginx:1.27`) | Reproducible builds — no surprise breaking changes |
| `chown` before `USER nginx` | Pre-grant write permissions so non-root Nginx can run |
| `USER nginx` | Principle of least privilege — no root in production |
| Port 8080 not 80 | Non-root processes can't bind ports below 1024 |
| `daemon off` | Keeps process in foreground as PID 1 — required for containers |

---


**Q: What is a multi-stage Docker build and why use it?**
> A multi-stage build uses multiple `FROM` statements in one Dockerfile. Each stage is isolated — you can use heavy build tools in early stages and copy only the output to a minimal final image. The result is a smaller, more secure production image. In Jerney's frontend, Stage 1 uses Node.js to build the React app, but the final image is pure Nginx (~25MB) with zero Node.js.

**Q: Why is layer ordering important in a Dockerfile?**
> Docker caches layers. If a layer's inputs haven't changed, it's reused from cache — making builds much faster. The trick is to order instructions from least-changing to most-changing. In Jerney, `package.json` is copied and `npm ci` runs before `COPY . .` — so dependency installation is cached as long as `package.json` doesn't change, even when source files do.

**Q: Why does the container run on port 8080 instead of port 80?**
> Linux doesn't allow non-root processes to bind to ports below 1024 (privileged ports). Since the container runs as the `nginx` user (non-root) for security, it must use port 8080. The Kubernetes Service maps external port 80 to the container's port 8080 with `targetPort: 8080`.

**Q: What does `daemon off` mean in the CMD?**
> By default, Nginx backgrounds itself (daemonises) after starting. In containers, the orchestrator (Docker/Kubernetes) tracks the process via PID 1. If PID 1 exits, the container is considered stopped. `daemon off` keeps Nginx running as PID 1 in the foreground so the container stays alive as long as Nginx is running.

**Q: Why use `USER nginx` and what does it protect against?**
> Running as root inside a container means if an attacker exploits a vulnerability in Nginx, they get root access inside the container — making container escape or lateral movement much easier. Running as `nginx` (a non-privileged user) limits the blast radius. Even if compromised, the attacker can't install packages, modify system files, or access other users' data.