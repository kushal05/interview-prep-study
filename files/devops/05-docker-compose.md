# Docker Compose

> **TL;DR:** Compose is your local production replica. One YAML file describes services, networks, volumes, and dependencies — `docker compose up` brings the stack online. Use it for dev, integration tests, and tiny single-host prod (not k8s replacement).

## File Anatomy

```yaml
# compose.yaml (preferred name; docker-compose.yml also works)
name: myapp                          # project name (defaults to dir)

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        NODE_ENV: production
      cache_from:
        - myapp/api:latest
    image: myapp/api:${TAG:-latest}
    container_name: myapp-api        # avoid in prod — breaks scale
    restart: unless-stopped
    ports:
      - "8080:3000"                  # host:container
    environment:
      DATABASE_URL: postgres://app:${DB_PASSWORD}@db:5432/app
      REDIS_URL: redis://redis:6379
      LOG_LEVEL: info
    env_file:
      - .env                          # loaded into container env
    depends_on:
      db:
        condition: service_healthy    # wait for healthcheck, not just start
      redis:
        condition: service_started
    networks: [backend]
    volumes:
      - ./uploads:/app/uploads
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 10s
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./db/init:/docker-entrypoint-initdb.d:ro
    networks: [backend]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--save", "60", "1", "--loglevel", "warning"]
    volumes:
      - redisdata:/data
    networks: [backend]

  nginx:
    image: nginx:1.27-alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on: [api]
    networks: [backend, frontend]

networks:
  backend:
    driver: bridge
  frontend:
    driver: bridge

volumes:
  pgdata:
  redisdata:
```

## CLI Cheat Sheet

```bash
docker compose up -d                 # start in background
docker compose up --build            # rebuild images first
docker compose down                  # stop + remove containers
docker compose down -v               # also remove named volumes (DESTRUCTIVE)

docker compose ps                    # status
docker compose logs -f api           # follow one service
docker compose logs --tail 100 -f    # all services

docker compose exec api sh           # shell into running service
docker compose run --rm api npm test # one-off command in a new container

docker compose restart api           # restart one
docker compose pull                  # pull all images
docker compose config                # validate + show effective config

docker compose --profile dev up      # only services tagged with profile "dev"
docker compose up --scale api=3      # run 3 replicas (no load balancer)
```

## Service Discovery

Each service is reachable from other services by its **service name** as DNS. Above, `api` connects to postgres as `db:5432`. No `localhost`, no IP. Compose creates a DNS entry per network.

```yaml
environment:
  DATABASE_URL: postgres://app:pw@db:5432/app   # "db" resolves to db container
```

## `depends_on` and the Race Condition

```yaml
depends_on:
  db:
    condition: service_healthy        # waits for DB healthcheck to pass
```

`condition: service_started` (default) only waits for the container to be running — the DB may still be initializing. **Always** use `service_healthy` for stateful deps. If the image has no healthcheck, define one in your compose file.

## Environment Variables

Three layers:

1. `environment:` in compose — explicit overrides.
2. `env_file:` — bulk loading from `.env`.
3. Shell env at the time of `docker compose up` — used for **interpolation** in the YAML itself (`${TAG:-latest}`).

```yaml
# Interpolation
image: myapp/api:${TAG:-latest}      # ${VAR:-default}, ${VAR:?error if unset}

# Passing host env into container
environment:
  - AWS_REGION                       # value taken from host's $AWS_REGION
```

**Gotcha:** `.env` (in project root) is used for *interpolation*. It does **not** get into containers unless you also list it under `env_file:`.

## Profiles

Skip services unless a profile is active. Useful for dev-only services (mailhog, adminer).

```yaml
services:
  api:
    image: myapp/api
  adminer:
    image: adminer
    profiles: ["dev"]                # not started unless --profile dev
    ports: ["8081:8080"]
```

```bash
docker compose up                    # api only
docker compose --profile dev up      # api + adminer
```

## Multiple Compose Files (compose merging)

Layered config — base + overrides.

```bash
# compose.yaml         (shared)
# compose.override.yml (auto-loaded for dev)
# compose.prod.yml     (explicit for prod)

docker compose up                                 # base + override (dev)
docker compose -f compose.yaml -f compose.prod.yml up   # base + prod
```

```yaml
# compose.override.yml
services:
  api:
    build:
      target: dev                    # multi-stage target for dev
    volumes:
      - ./src:/app/src               # live reload
    command: npm run dev
    ports:
      - "9229:9229"                  # debugger
```

```yaml
# compose.prod.yml
services:
  api:
    image: ghcr.io/me/myapp:${TAG}   # no build, use registry
    restart: always
    deploy:
      resources:
        limits: { cpus: "2", memory: 1G }
```

## Healthchecks — Be Specific

```yaml
healthcheck:
  test: ["CMD", "curl", "-fsS", "http://localhost:3000/health"]
  interval: 30s
  timeout: 3s
  retries: 3
  start_period: 30s                  # grace period before failures count
```

- Use `CMD` (exec form) — not `CMD-SHELL` unless you need a shell pipeline.
- Set `start_period` for slow-booting apps (JVM, big migrations).
- The container shows `(healthy)` in `docker ps` once healthcheck passes.

## Networks — Segmenting Traffic

```yaml
services:
  db:
    networks: [backend]              # only backend can reach db
  api:
    networks: [backend, frontend]    # gateway between them
  nginx:
    networks: [frontend]             # public-facing only
```

Compose auto-creates networks, prefixed with project name. Use this to enforce that your DB is unreachable from outside.

## Volumes — Persist State

```yaml
volumes:
  pgdata:                            # named volume, lives in /var/lib/docker
  uploads:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/nas/uploads       # mount NAS path
```

Mount in service:

```yaml
services:
  db:
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./backups:/backups:ro        # host path read-only
```

`docker compose down` does NOT delete volumes. `docker compose down -v` does — be careful.

## CI Patterns

```bash
# Spin up infra for integration tests
docker compose -f compose.test.yaml up -d --wait
npm test
docker compose -f compose.test.yaml down -v
```

`--wait` (compose v2.17+) blocks until healthchecks pass — no `sleep 30` needed.

## Interview Questions

**Q: When would you use Docker Compose vs Kubernetes?**
A: Compose for single-host (laptop, small VPS, CI). K8s for production multi-host, auto-scaling, rolling updates, self-healing. Compose is great for dev + integration tests — you can use the *same* compose file your devs use to run tests in CI.

**Q: How do services in compose find each other?**
A: Each service is registered as a DNS name on its network(s). `api` resolves to the api container(s). No IP plumbing needed.

**Q: `depends_on` doesn't actually wait for my DB to be ready. Why?**
A: Default condition is `service_started`, which means "container is running" — but Postgres takes seconds to accept connections. Use `condition: service_healthy` plus a healthcheck.

**Q: How do you scale a service?**
A: `docker compose up --scale api=3`. But compose doesn't load-balance — you'd need an nginx in front, or move to Swarm / K8s for real scaling.

**Q: Where does `.env` go?**
A: `.env` next to `compose.yaml` is read for **interpolating** `${VARS}` in the YAML. To pass variables into containers, use `env_file:` or `environment:`.

**Q: What's the difference between `docker compose run` and `docker compose exec`?**
A: `exec` runs a command in an already-running container. `run` creates a *new* container from the service definition (useful for one-shots like `npm test`).

## Common Pitfalls

- Forgetting `service_healthy` — your API starts before the DB is accepting connections, crashes once.
- Not pinning image versions (`postgres:latest`) — non-reproducible local environments.
- Mounting source into a Node container without excluding `node_modules` — host's modules clobber the container's, often with wrong arch.
- `docker compose down -v` in prod — wipes named volumes.
- Mixing host networking with service-name DNS — they're mutually exclusive on Linux.

## Related

- [04-docker-basics.md](04-docker-basics.md)
- [06-kubernetes-fundamentals.md](06-kubernetes-fundamentals.md)
- [10-ci-cd-fundamentals.md](10-ci-cd-fundamentals.md)
- [../nodejs/23-deployment-and-docker.md](../nodejs/23-deployment-and-docker.md)
