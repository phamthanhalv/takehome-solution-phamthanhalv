## Diagnosis Process

### Configuration Review
- Examine `docker-compose.yml` to understand service architecture
- Review nginx configuration in `nginx/conf.d/default.conf`
- Review API code in `api/src/index.js`

### Port Mismatch Discovery
- Test the endpoints
```bash
curl http://localhost:8080/
Welcome to the platform

curl http://localhost:8080/api/users
502 Bad Gateway
```
- nginx proxies to port 3001 (`proxy_pass http://api:3001`)
- API listens on port 3000 (`app.listen(3000, ...)`)
- Port mismatch causing api requests to fail

### Dependency Analysis
- Simple `depends_on` with no health checks
- Might cause race conditions during startup -> failed api requests during startup

### No restart policy discovery
- Without a restart policy, a crashed container stays down permanently, any single crash makes the platform inaccessible until manual intervention.

### Unused db init script
- `postgres/init.sql` script exists to set `max_connections = 20` but wasn't mounted to the postgres container

### No persistent volumes
- `postgres_data` and `redis_data` volumes were not declared
---

## Fixes

### Fix 1: Correct nginx port configuration
**File**: `nginx/conf.d/default.conf`
```diff
- proxy_pass http://api:3001;
+ proxy_pass http://api:3000;`
```

### Fix 2: Add health checks, proper dependencies, volumes, and restart policies
```yaml
services:
  nginx:
    image: nginx:1.25
    ports:
      - "8080:80"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
    depends_on:
      api:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/"]
      interval: 5s
      timeout: 3s
      retries: 3
      start_period: 5s

  api:
    build: ./api
    environment:
      - DB_HOST=postgres
      - REDIS_HOST=redis
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/status"]
      interval: 5s
      timeout: 3s
      retries: 3
      start_period: 10s

  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s

  redis:
    image: redis:7
    volumes:
      - redis_data:/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3
      start_period: 5s

  volumes:
    postgres_data:
    redis_data:
```

```bash
location /status {
    proxy_pass http://api:3000/status;   # expose health endpoint through nginx
}
```

### Fix 4: Mount db init script
```yaml
postgres:
  volumes:
    - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
```
---

## Monitoring & Alerts to Add

| Signal | Tool | Alert condition |
|---|---|---|
| HTTP 5xx rate | Prometheus + Alertmanager | > 1% of requests over 1 min |
| API response latency p99 | Prometheus | > 500 ms |
| Postgres connections | `pg_stat_activity` exporter | connections > 80% of max |
| Container restarts | Docker daemon metrics | restart count > 0 in 5 min |
| Redis memory | Redis exporter | > 90% `maxmemory` |
---

## How to Prevent This in Production

1. IaC review checklist — PR review checklist that flags missing
   `restart`, `healthcheck`... before any compose file is merged.

2. Staging smoke tests — a simple `curl` loop against all endpoints after
   `docker compose up` catches 502s before production.

3. Persistent volumes declared from day 0 — treat stateful containers the
   same as physical disks: always named volumes, always backed up.

