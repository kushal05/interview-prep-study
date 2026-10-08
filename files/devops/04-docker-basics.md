# Docker Basics

> **TL;DR:** Images are read-only stacks of layers; containers are running instances. Optimize for layer cache, slim base, single concern per image, non-root user, and clear ENTRYPOINT vs CMD semantics.

## Mental Model

- **Image:** immutable, content-addressable stack of filesystem layers + metadata. Built from a Dockerfile.
- **Container:** an image plus a thin writable layer, plus a process tree in isolated namespaces (PID, NET, MNT, UTS, IPC, USER) and cgroups for resource limits.
- **Registry:** a server that stores images (Docker Hub, ECR, GHCR, Artifactory).

```bash
docker images
docker ps                           # running containers
docker ps -a                        # all (including exited)
docker run --rm -it ubuntu:22.04 bash
docker exec -it <container> bash    # attach a shell to a running container
docker logs -f <container>          # follow logs
docker stats                        # live resource usage
docker inspect <container>          # full JSON state
```

## Dockerfile — Every Instruction

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:20-alpine AS deps

# Metadata
LABEL org.opencontainers.image.source="https://github.com/me/app"
LABEL org.opencontainers.image.version="1.4.0"

# Build args (compile-time, available only during build)
ARG NODE_ENV=production
ENV NODE_ENV=$NODE_ENV              # ENV persists in the image

WORKDIR /app

# Copy manifest *before* source so node_modules layer is cached
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Now copy source — changes here don't bust node_modules layer
COPY src ./src
COPY public ./public

# Non-root user (security)
RUN addgroup -S app && adduser -S app -G app
USER app

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget -qO- http://localhost:3000/health || exit 1

# CMD = default command, overridable. ENTRYPOINT = always runs.
ENTRYPOINT ["node"]
CMD ["server.js"]
```

### ENTRYPOINT vs CMD — the eternal interview question

| Form | What runs |
|------|-----------|
| `ENTRYPOINT ["node"]` + `CMD ["server.js"]` | `node server.js` (and `docker run img foo.js` runs `node foo.js`) |
| Only `CMD ["node", "server.js"]` | `node server.js` (and `docker run img bash` overrides entirely) |
| Only `ENTRYPOINT ["node", "server.js"]` | `node server.js` always; can't be overridden without `--entrypoint` |

**Pattern:** `ENTRYPOINT` = fixed binary; `CMD` = default arguments.

### Exec vs Shell Form

```dockerfile
CMD ["node", "server.js"]     # exec form — node is PID 1, signals delivered
CMD node server.js            # shell form — /bin/sh -c "node server.js", sh is PID 1
```

Exec form is almost always what you want. Shell form breaks SIGTERM handling.

## Layer Caching — The Speed Dial

Each instruction is a layer. Docker reuses a layer if the instruction text + the input files haven't changed.

```dockerfile
# WRONG — every code change reinstalls all deps
COPY . .
RUN npm ci

# RIGHT — deps layer only invalidates when package*.json changes
COPY package*.json ./
RUN npm ci
COPY . .
```

Order from least-changing to most-changing:
1. `FROM`
2. system packages (`apt-get install`)
3. language deps (`npm ci`, `pip install`, `go mod download`)
4. source code
5. config

### BuildKit cache mounts (huge speedup)

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci
COPY . .
```

The cache survives across builds, not stored in the image. Works for `apt`, `go mod`, `pip`, etc.

## Multi-Stage Builds

Single image, multiple `FROM`s. Copy only what you need into the final stage.

```dockerfile
# Stage 1: build
FROM golang:1.22-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./cmd/server

# Stage 2: runtime (tiny)
FROM gcr.io/distroless/static-debian12
COPY --from=builder /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

Result: a ~10MB image with no shell, no package manager, no CVEs in unused tools.

## .dockerignore — Don't Ship the World

```dockerignore
.git
.gitignore
node_modules
dist
.env
.env.*
*.md
.vscode
.idea
Dockerfile
docker-compose*.yml
coverage
.npm
.cache
```

Without this, your build context includes `.git` (could be hundreds of MB) and your `node_modules` get COPYed before being overwritten by `npm ci`.

## Base Image Choices

| Base | Size | Use when |
|------|------|----------|
| `node:20` (debian) | ~1GB | You need glibc + apt — easy debugging |
| `node:20-slim` | ~250MB | Smaller debian, fewer tools |
| `node:20-alpine` | ~150MB | musl libc — fine for pure JS, breaks native modules |
| `gcr.io/distroless/nodejs20` | ~120MB | Production — no shell, no apt, just runtime |
| `scratch` | 0 bytes | Static binaries (Go, Rust) |

**Alpine gotcha:** `musl` differs from `glibc`. Some npm packages with native bindings (sharp, bcrypt) fail or are slower. Test on alpine before committing.

## Resource Limits

```bash
docker run -d \
    --name api \
    --memory=512m \
    --memory-swap=512m \           # disable swap (= memory)
    --cpus=1.5 \
    --pids-limit=200 \
    --restart=on-failure:5 \
    --read-only \
    --tmpfs /tmp:rw,size=64m \
    -p 8080:3000 \
    myapp:latest
```

Without `--memory`, a leaking container can take down the host. Always set limits in prod.

## Networking

```bash
docker network create app-net
docker run -d --network app-net --name db postgres:16
docker run -d --network app-net --name api -p 8080:3000 myapp
# api can reach db at hostname "db" on port 5432
```

Networks:
- **bridge** (default) — NAT'd, containers see each other by IP only on the same network.
- **host** — shares host's network stack (Linux only). No isolation, no port mapping needed.
- **none** — fully isolated.
- **overlay** — multi-host (Swarm).

## Volumes & Bind Mounts

```bash
# Named volume — managed by Docker, survives container removal
docker volume create pgdata
docker run -v pgdata:/var/lib/postgresql/data postgres

# Bind mount — host path mapped in
docker run -v $(pwd)/src:/app/src node

# Anonymous tmpfs — RAM-backed
docker run --tmpfs /tmp myapp
```

Volumes for state (DBs), bind mounts for dev (live code), tmpfs for secrets-in-memory.

## Inspect & Debug

```bash
docker inspect <container> | jq '.[0].State'
docker logs --tail 200 -f <container>
docker exec -it <container> sh

# Why did it die?
docker ps -a --filter "name=api" --format "table {{.Names}}\t{{.Status}}\t{{.RunningFor}}"
docker inspect <container> --format='{{.State.ExitCode}} {{.State.Error}}'

# Disk usage
docker system df
docker system prune -a --volumes   # nuke everything unused
```

## Security Hardening Checklist

- Run as non-root (`USER app`).
- Drop capabilities: `--cap-drop=ALL --cap-add=NET_BIND_SERVICE` (only if you need <1024).
- Read-only root FS: `--read-only` + `--tmpfs /tmp`.
- No `--privileged` (kernel namespace breakout).
- Pin image versions by digest: `FROM node@sha256:abc...` (not `node:latest`).
- Scan images: `trivy image myapp:1.0`, `docker scout cves myapp:1.0`.
- Sign images: `cosign sign ghcr.io/me/app:1.0`.

## Interview Questions

**Q: Why multi-stage builds?**
A: Final image only contains runtime artifacts — no build tools (compilers, npm, dev deps). Smaller, faster pulls, fewer CVEs, no source code in image.

**Q: ENTRYPOINT vs CMD?**
A: ENTRYPOINT = fixed binary that runs. CMD = default arguments to ENTRYPOINT, overridable by `docker run img <args>`. Use both: `ENTRYPOINT ["node"]` + `CMD ["server.js"]`.

**Q: How do you minimize the size of a Docker image?**
A: Multi-stage build, distroless or alpine base, combine RUN commands (`apt-get install && rm -rf /var/lib/apt/lists/*`), clean caches in same layer, `.dockerignore`, copy only what's needed.

**Q: A code change rebuilds everything. Why?**
A: Layer cache busted because `COPY . .` runs before `RUN npm ci`. Fix: copy `package*.json` first, install deps, then copy the rest.

**Q: How do signals work in containers?**
A: PID 1 inside the container receives signals from `docker stop` (SIGTERM, then SIGKILL after grace). If you use shell form (`CMD npm start`), `sh` is PID 1 and doesn't forward signals — node never gets SIGTERM. Use exec form or `tini` (`--init`).

**Q: Difference between volume and bind mount?**
A: Volumes are Docker-managed (`/var/lib/docker/volumes/...`), portable, backupable via Docker. Bind mounts map an arbitrary host path — useful in dev, fragile in prod (host path may not exist).

## Common Pitfalls

- `COPY . .` before deps install — busts cache on every code change.
- Shell-form CMD — signals don't reach your app; graceful shutdown fails.
- `FROM node:latest` — non-reproducible builds; pin to a digest or at least `node:20.11.1-alpine`.
- Running as root — combined with mounted volume, container can chown host files.
- Forgetting `.dockerignore` — `.git` and `node_modules` bloat builds 10x.
- `apt-get install` without `--no-install-recommends` and without cleaning `/var/lib/apt/lists/*` in the same RUN — image bloat + cached package lists go stale.

## Related

- [05-docker-compose.md](05-docker-compose.md)
- [06-kubernetes-fundamentals.md](06-kubernetes-fundamentals.md)
- [21-security-devsecops.md](21-security-devsecops.md)
- [../nodejs/23-deployment-and-docker.md](../nodejs/23-deployment-and-docker.md)
