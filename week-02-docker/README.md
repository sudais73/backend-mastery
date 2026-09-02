# Docker Mastery Complete Guide (7/7 Checklist)

## Overview
**Goal:** Master Docker completely. Understand images, containers, Dockerfiles, and multi-service apps.

**What you'll learn:** Package any app into a container that runs identically everywhere.

---

## Mastery Checklist (0/7 → 7/7)

### **1. ✅ Explain the difference between an image, a container, and a VM**

#### **The Three Concepts**

| Concept | What it is | Analogy | Storage | Startup | Size |
|---------|-----------|---------|---------|---------|------|
| **Image** | Blueprint/recipe (static) | Class definition | Disk (stored) | N/A | 100MB-1GB |
| **Container** | Running instance (dynamic) | Object instance | RAM (running) | Milliseconds | RAM only |
| **VM** | Full OS virtualization | Entire computer | Disk + RAM | Seconds | Gigabytes |

#### **Key Differences Explained**

**Image:**
```
Think: Your Dockerfile
Result: myapp:1.0 (stored on disk)
Use: Can't run directly, must create container first
Size: 300MB (compressed, shared)
```

**Container:**
```
Think: docker run myapp:1.0
Result: Running process with own filesystem, network, processes
Use: Temporary (dies when stopped)
Size: Only uses RAM for running process
```

**VM:**
```
Think: VirtualBox, VMware
Result: Entire Ubuntu OS running inside hypervisor
Use: Heavy isolation, full OS
Size: 5-10GB, boots in 30+ seconds
```

#### **Visual Comparison**

**Docker Container:**
```
┌─────────────────────────────────┐
│ Your App (Node.js)              │
├─────────────────────────────────┤
│ node_modules + dependencies     │
├─────────────────────────────────┤
│ Node.js runtime                 │
├─────────────────────────────────┤
│ Ubuntu base OS (no kernel)      │ ← Shares host kernel
├─────────────────────────────────┤
│ Linux Kernel (HOST)             │ ← Shared with other containers
└─────────────────────────────────┘
```

**Virtual Machine:**
```
┌─────────────────────────────────┐
│ Your App (Node.js)              │
├─────────────────────────────────┤
│ node_modules + dependencies     │
├─────────────────────────────────┤
│ Node.js runtime                 │
├─────────────────────────────────┤
│ Ubuntu OS (full)                │
├─────────────────────────────────┤
│ Linux Kernel (own)              │ ← Separate from host
├─────────────────────────────────┤
│ Hypervisor (VMware/VirtualBox)  │
├─────────────────────────────────┤
│ Host OS + Hardware              │
└─────────────────────────────────┘
```

#### **Why This Matters**

**Container = faster, lighter**
- Start in milliseconds (no OS boot)
- Use minimal RAM
- Ideal for microservices (run 100s of containers)

**VM = more isolated, heavier**
- Full OS isolation (security)
- Start in seconds (OS boot)
- Run different OSes on one machine

**Interview Answer:**
> "Containers share the host kernel (lightweight, fast), while VMs virtualize hardware and boot a full OS (heavy, slow). Docker uses containers. For our needs, containers are perfect — we get isolation without the overhead."

---

### **2. ✅ Write a Dockerfile from scratch and know when to use CMD vs ENTRYPOINT**

#### **Dockerfile Structure — Every Command Explained**

```dockerfile
# Base image (OS + runtime)
FROM node:18-alpine

# Set working directory inside container
WORKDIR /app

# Copy package files from host → container
COPY package*.json ./

# Install dependencies (creates layer)
RUN npm install

# Copy app source code
COPY . .

# Build (for Next.js, etc.)
RUN npm run build

# Metadata: which port container listens on
EXPOSE 3000

# Environment variables
ENV NODE_ENV=production
ENV DATABASE_URL=postgresql://localhost/db

# Health check (Docker knows if app is healthy)
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

# Default command when container starts
CMD ["npm", "start"]
```

#### **Each Line Explained**

| Line | Purpose | What it does |
|------|---------|-------------|
| `FROM node:18-alpine` | Base image | Start with Node 18 (alpine = minimal, 160MB) |
| `WORKDIR /app` | Set directory | All commands run in /app inside container |
| `COPY package*.json ./` | Copy files | Copy package.json + package-lock.json |
| `RUN npm install` | Execute | Install dependencies (creates layer, cached) |
| `COPY . .` | Copy app | Copy all source files |
| `RUN npm run build` | Build app | Compile/bundle app |
| `EXPOSE 3000` | Document port | Container listens on 3000 (informational) |
| `ENV NODE_ENV=production` | Environment | Set env variable |
| `HEALTHCHECK` | Monitor health | Docker checks if app is alive |
| `CMD ["npm", "start"]` | Default command | Run this when container starts |

#### **CMD vs ENTRYPOINT — Critical Difference**

**CMD** = Default command, can be overridden

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY . .
CMD ["npm", "start"]
```

**Usage:**
```bash
docker run myapp                    # Runs: npm start
docker run myapp npm test           # Runs: npm test (overrides CMD)
```

**ENTRYPOINT** = Fixed entry point, harder to override

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY . .
ENTRYPOINT ["npm", "start"]
```

**Usage:**
```bash
docker run myapp                    # Runs: npm start
docker run myapp npm test           # Runs: npm start npm test (concatenates)
```

#### **When to Use Which**

**Use CMD when:**
- Default behavior can be overridden
- Want flexibility
- Running web servers (might want different command for testing)

**Use ENTRYPOINT when:**
- Command is mandatory
- Building a CLI tool
- Want strict control

**Best Practice: Combination**

```dockerfile
ENTRYPOINT ["node", "/app/server.js"]
CMD ["--port", "3000"]
```

**Result:**
```bash
docker run myapp                           # node /app/server.js --port 3000
docker run myapp --port 8080               # node /app/server.js --port 8080
```

#### **Real Example: Next.js Dockerfile**

```dockerfile
FROM node:18-alpine
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy source
COPY . .

# Build Next.js
RUN npm run build

# Expose port
EXPOSE 3000

# Production start
CMD ["npm", "start"]
```

---

### **3. ✅ Order instructions to take advantage of layer caching**

#### **Layer Caching: The Performance Secret**

**Understanding Layers:**
```dockerfile
FROM node:18-alpine              # Layer 1: ~160MB
WORKDIR /app                     # Layer 2: no size change
COPY package*.json ./            # Layer 3: ~100KB
RUN npm install                  # Layer 4: ~200MB
COPY . .                         # Layer 5: ~2MB (your code)
RUN npm run build                # Layer 6: ~100MB (build output)
```

Each instruction = one layer. **Docker caches layers.**

#### **The Cache Problem: Bad Order**

❌ **BAD ORDER:**
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY . .                         # Copy code first!
RUN npm install                  # Install dependencies
```

**What happens:**
```
You edit app.js (change code)
→ Layer 3 (COPY . .) changed
→ Docker invalidates cache
→ Re-runs RUN npm install (reinstalls everything!)
→ Takes 2 minutes (slow!)
```

#### **The Cache Solution: Good Order**

✅ **GOOD ORDER:**
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./            # Copy dependencies first (changes rarely)
RUN npm install                  # Install (cached if package.json unchanged)
COPY . .                         # Copy code (changes frequently)
RUN npm run build
```

**What happens:**
```
You edit app.js (change code)
→ Layer 1-4 (dependencies) unchanged
→ Docker uses cache for layers 1-4 (instant!)
→ Only re-runs layer 5 (COPY . .) and 6 (npm run build)
→ Takes 10 seconds (fast!)
```

#### **Cache Strategy Principle**

**Order from least-changed to most-changed:**

```dockerfile
FROM node:18-alpine              # Changes: rarely (base OS updates)
WORKDIR /app                     # Changes: never

COPY package*.json ./            # Changes: sometimes (add dependency)
RUN npm install                  # ← Gets cached if package.json unchanged

COPY . .                         # Changes: frequently (edit code)
RUN npm run build                # ← Re-runs often, but dependencies cached
```

#### **Real Build Time Comparison**

**First build (no cache):**
```
FROM ... → 5 seconds
COPY package.json → 2 seconds
RUN npm install → 60 seconds
COPY . . → 2 seconds
RUN npm run build → 30 seconds
Total: 99 seconds
```

**Second build, only code changed (with good order):**
```
Layers 1-4: Cache hit (0 seconds)
COPY . . → 2 seconds (code changed, cache bust)
RUN npm run build → 30 seconds (code changed, re-build)
Total: 32 seconds (3x faster!)
```

**Second build with BAD order (code first):**
```
Layers 1-3: Cache hit (0 seconds)
COPY . . → 2 seconds (code changed, cache bust)
RUN npm install → 60 seconds (re-installs everything!)
RUN npm run build → 30 seconds
Total: 92 seconds (almost full rebuild!)
```

#### **Interview Answer**

> "I order Dockerfile instructions from least-changed to most-changed. Package files and dependencies come first (stable, cached), app code comes last (changes frequently). This way, editing code only re-runs the build step, not the expensive npm install."

---

### **4. ✅ Use multi-stage builds to produce small production images**

#### **The Problem: Build Bloat**

**Single-stage Dockerfile:**
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install                     # 200MB dependencies
COPY . .
RUN npm run build                   # 100MB+ build output
EXPOSE 3000
CMD ["npm", "start"]
```

**Result:**
- Final image: 500MB+
- Includes dev dependencies (testing libs, type definitions)
- Includes build tools (eslint, typescript compiler)
- **All unnecessary for production!**

#### **The Solution: Multi-Stage Builds**

**Two-stage Dockerfile:**

```dockerfile
# ===== STAGE 1: Build =====
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install                     # Install everything (dev + prod)

COPY . .
RUN npm run build                   # Build app (creates .next, dist, etc.)


# ===== STAGE 2: Runtime (Production) =====
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --only=production   # Only production dependencies!

# Copy ONLY built files from builder stage
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public

EXPOSE 3000
CMD ["npm", "start"]
```

#### **What Multi-Stage Does**

**Stage 1 (Builder):**
- Starts fresh with full node image
- Installs ALL dependencies (dev + prod)
- Runs build command
- Result: 1.5GB intermediate image (discarded)

**Stage 2 (Runtime):**
- Starts fresh with minimal node image
- Copies ONLY production dependencies
- Copies ONLY built files (`.next`, `public`)
- Result: 300MB final image (kept and pushed)

**Final image only includes:**
✅ Production dependencies
✅ Built app files
✅ Node runtime
❌ Dev tools (deleted)
❌ Typescript compiler (deleted)
❌ Test files (deleted)

#### **Size Comparison**

```
Single-stage: 500MB (bloated)
Multi-stage: 300MB (optimized)
Improvement: 40% smaller!
```

#### **Next.js Specific Example**

```dockerfile
# Stage 1: Builder
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci                          # Clean install
COPY . .
RUN npm run build                   # Creates .next folder

# Stage 2: Runtime
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production        # Smaller

COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/node_modules ./node_modules

EXPOSE 3000
CMD ["npm", "start"]
```

#### **Commands to Compare Sizes**

```bash
# Build single-stage
docker build -t myapp:single -f Dockerfile.single .
docker images myapp:single
# → REPOSITORY   TAG     SIZE
# → myapp        single  500MB

# Build multi-stage
docker build -t myapp:multi -f Dockerfile.multi .
docker images myapp:multi
# → REPOSITORY   TAG     SIZE
# → myapp        multi   300MB (40% smaller!)
```

#### **Interview Answer**

> "Multi-stage builds separate build and runtime. Stage 1 installs everything and builds the app. Stage 2 copies only production dependencies and built files. This removes dev tools and unnecessary files, cutting final image size by 40-50%. Smaller images = faster pushes, pulls, and deployments."

---

### **5. ✅ Persist data with volumes and connect containers over a network**

#### **The Problem: Data Loss**

**Without volumes:**
```bash
docker run -d postgres:15
# Container runs, creates database
# Then: docker rm postgres-container
# → Database is GONE forever!
```

**Why:** Container filesystem is ephemeral. When container stops, data vanishes.

#### **Solution 1: Named Volumes (Recommended)**

**Create volume:**
```bash
docker volume create mydata
```

**Use in container:**
```bash
docker run -d -v mydata:/var/lib/postgresql/data postgres:15
```

**What happens:**
- Data written to `/var/lib/postgresql/data` inside container
- Actually stored in `/var/lib/docker/volumes/mydata/_data` on host
- Persists even if container is deleted!

**Verify:**
```bash
docker volume ls
docker volume inspect mydata
```

#### **Solution 2: Bind Mounts (Development)**

**Bind host directory to container:**
```bash
docker run -d -v ~/my-data:/var/lib/postgresql/data postgres:15
```

**What happens:**
- `/var/lib/postgresql/data` inside container = `~/my-data` on host
- Changes in either location are synced
- Perfect for development (edit files locally, see changes in container)

#### **Solution 3: docker-compose Volumes**

```yaml
version: '3.9'

services:
  db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - db_data:/var/lib/postgresql/data  # Named volume
    ports:
      - "5432:5432"

  app:
    build: .
    volumes:
      - ./src:/app/src                    # Bind mount for dev
    ports:
      - "3000:3000"

volumes:
  db_data:                                # Define named volume
```

**Commands:**
```bash
docker-compose up                         # Creates volume automatically
docker-compose down                       # Stops containers, keeps volume
docker-compose down -v                    # Stops containers AND deletes volume
```

#### **Container Networking**

**Problem:** Containers run in isolation. How do they talk?

**Solution: docker-compose networks**

```yaml
version: '3.9'

services:
  app:
    build: .
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp
    ports:
      - "3000:3000"
    depends_on:
      - db
    networks:
      - mynet

  db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=pass
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - mynet

volumes:
  db_data:

networks:
  mynet:                                  # Custom network
```

**How it works:**
- `db` container joins `mynet` network
- `app` container joins `mynet` network
- Docker's internal DNS resolves `db` → container IP
- **`app` can connect to `db:5432` directly by name!**

**In code:**
```javascript
// app.js (inside app container)
const db = new Client({
  host: 'db',        // References database service by name!
  port: 5432,
  database: 'myapp',
  user: 'user',
  password: 'pass'
});
```

#### **Real Example: Next.js + PostgreSQL + Redis**

```yaml
version: '3.9'

services:
  app:
    build: .
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/app
      - REDIS_URL=redis://redis:6379
    ports:
      - "3000:3000"
    depends_on:
      - db
      - redis
    volumes:
      - ./src:/app/src              # Dev: sync code
      - /app/node_modules           # Keep dependencies in container
    networks:
      - appnet

  db:
    image: postgres:15
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=app
    volumes:
      - db_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - appnet

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - appnet

volumes:
  db_data:
  redis_data:

networks:
  appnet:
```

#### **Interview Answer**

> "Volumes persist data beyond container lifecycle. Named volumes are best for databases. Bind mounts are best for development (sync code). docker-compose connects containers on a shared network where they communicate by service name (e.g., 'db:5432'). Docker's DNS resolves service names to container IPs automatically."

---

### **6. ✅ Bring up a multi-service app with a single docker compose up**

#### **What docker-compose Does**

Manages multiple containers as one system. One command orchestrates everything.

#### **Full Stack Example: Next.js + PostgreSQL + Redis + Nginx**

```yaml
version: '3.9'

services:
  # Frontend + API (Next.js)
  app:
    build: ./app                          # Build from ./app/Dockerfile
    container_name: myapp-app
    restart: always
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:pass@db:5432/app
      - REDIS_URL=redis://redis:6379
      - NEXTAUTH_SECRET=your-secret-key
      - NEXTAUTH_URL=http://localhost
    ports:
      - "3000:3000"
    depends_on:
      - db
      - redis
    volumes:
      - ./app:/app
      - /app/node_modules
    networks:
      - appnet
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3

  # PostgreSQL Database
  db:
    image: postgres:15
    container_name: myapp-db
    restart: always
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=securepass
      - POSTGRES_DB=app
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d  # Run SQL on startup
    networks:
      - appnet
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    restart: always
    command: redis-server --appendonly yes
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - appnet

  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: myapp-nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - app
    networks:
      - appnet

volumes:
  db_data:
  redis_data:

networks:
  appnet:
    driver: bridge
```

#### **Commands You'll Use**

```bash
# Start all services (builds images if needed)
docker-compose up -d

# Follow logs from all services
docker-compose logs -f

# Follow logs from specific service
docker-compose logs -f app

# Stop all services (keeps volumes)
docker-compose down

# Stop and remove volumes
docker-compose down -v

# Rebuild images (if Dockerfile changed)
docker-compose up -d --build

# Execute command in container
docker-compose exec app npm run migrate

# Scale service (run multiple instances)
docker-compose up -d --scale app=3

# Check status
docker-compose ps
```

#### **Startup Flow**

When you run `docker-compose up -d`:

1. **Creates network `appnet`**
2. **Starts `db` container**
   - Waits for PostgreSQL to be ready (healthcheck)
   - Mounts SQL migrations
3. **Starts `redis` container**
   - Immediately available
4. **Starts `app` container**
   - Waits for db + redis (depends_on)
   - Connects via network
   - Runs `npm start`
5. **Starts `nginx` container**
   - Routes traffic to app
   - Waits for app to be healthy

**Result:** One command orchestrates 4 containers with proper ordering and networking.

#### **Development vs Production docker-compose**

**docker-compose.yml (Development)**
```yaml
services:
  app:
    build: .
    volumes:
      - ./src:/app/src          # Sync code
    environment:
      - NODE_ENV=development
```

**docker-compose.prod.yml (Production)**
```yaml
services:
  app:
    image: myrepo/myapp:1.0     # Pre-built image
    restart: always             # Auto-restart on crash
    # No volumes (no sync)
    environment:
      - NODE_ENV=production
```

**Usage:**
```bash
# Development
docker-compose up

# Production
docker-compose -f docker-compose.prod.yml up -d
```

#### **Interview Answer**

> "docker-compose orchestrates multi-service apps. One `docker-compose.yml` defines all services, volumes, networks, and dependencies. `docker-compose up` starts everything in the right order with proper networking. It's the standard way to develop and deploy backend systems."

---

### **7. ✅ Shell into a container and read its logs to debug it**

#### **Reading Logs**

**View all logs:**
```bash
docker logs container-id
```

**Follow logs (tail -f):**
```bash
docker logs -f container-id
```

**Last 50 lines:**
```bash
docker logs --tail 50 container-id
```

**With timestamps:**
```bash
docker logs -t container-id
```

**docker-compose logs:**
```bash
docker-compose logs                 # All services
docker-compose logs -f              # Follow all
docker-compose logs -f app          # Follow specific service
docker-compose logs --tail 100      # Last 100 lines
```

#### **Shell Access**

**Interactive bash:**
```bash
docker exec -it container-id bash
```

**Non-interactive command:**
```bash
docker exec container-id ls /app
```

**Within docker-compose:**
```bash
docker-compose exec app bash
docker-compose exec db psql -U user -d myapp
docker-compose exec redis redis-cli
```

#### **Real Debugging Scenario**

**Problem:** Next.js app won't start in Docker

**Step 1: Check logs**
```bash
docker logs myapp-app
# Output:
# npm ERR! code ENOENT
# npm ERR! errno -2
# npm ERR! syscall open
# npm ERR! enoent: no such file or directory, open '/app/package.json'
```

**Diagnosis:** package.json missing in container

**Step 2: Shell in**
```bash
docker exec -it myapp-app bash
cd /app
ls -la
```

**Output:** No package.json! Dockerfile COPY failed.

**Step 3: Fix Dockerfile**
```dockerfile
# Wrong
FROM node:18-alpine
WORKDIR /app
RUN npm install         # No package.json yet!

# Correct
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./   # Copy files FIRST
RUN npm install         # Then install
```

#### **Common Issues & Solutions**

**Issue 1: Port already in use**
```bash
docker logs myapp
# Error: listen EADDRINUSE: address already in use :::3000

# Solution:
docker-compose down
docker-compose up
```

**Issue 2: Database connection fails**
```bash
docker logs myapp
# Error: connect ECONNREFUSED 127.0.0.1:5432

# Problem: Using localhost, not container hostname
# Wrong: DATABASE_URL=postgresql://localhost/db
# Correct: DATABASE_URL=postgresql://db:5432/db

# Solution: Fix environment variable
docker-compose down
# Edit docker-compose.yml
docker-compose up
```

**Issue 3: Out of disk space**
```bash
docker system df
# Shows: 4.2 GB for images, 3.8 GB for containers

# Clean up:
docker system prune -a          # Remove all unused
docker image prune               # Remove unused images
docker volume prune              # Remove unused volumes
```

**Issue 4: Permission denied in container**
```bash
docker exec myapp touch /app/test.txt
# Permission denied

# Problem: Running as root, wrong permissions
# Solution: Add USER in Dockerfile
FROM node:18-alpine
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001
USER nodejs     # ← Run as non-root
```

#### **Debugging Checklist**

```bash
# 1. Check if container is running
docker ps
docker ps -a        # Even stopped ones

# 2. Read logs
docker logs container-id
docker logs -f container-id

# 3. Shell in
docker exec -it container-id bash

# 4. Check network
docker network ls
docker network inspect mynet

# 5. Check volume
docker volume ls
docker volume inspect mydata

# 6. Check port
netstat -tlnp | grep 3000
lsof -i :3000

# 7. Restart container
docker restart container-id

# 8. Full rebuild
docker-compose down
docker-compose up --build
```

#### **Interview Answer**

> "Debug Docker issues with three tools: logs show app errors, exec gives shell access, and docker-compose logs shows all services. I always start with logs to see the error, then shell in to inspect the filesystem and environment. This catches 95% of issues — usually wrong env variables, missing files, or network connectivity."

---

## Complete Mastery Checklist

### ✅ **All 7 Items Explained**

```
☑ 1. Explain image vs container vs VM
    → Images are blueprints, containers are running instances,
      VMs virtualize hardware (containers share kernel)

☑ 2. Write Dockerfile from scratch, CMD vs ENTRYPOINT
    → CMD is overridable default, ENTRYPOINT is fixed entry point
      Order: base image → dependencies → code → CMD/ENTRYPOINT

☑ 3. Order for layer caching
    → Put stable things first (base image, dependencies),
      changing things last (app code). Cache layers between builds.

☑ 4. Multi-stage builds for small images
    → Stage 1 builds app (keeps dev tools), Stage 2 copies only
      production files. 40-50% smaller final image.

☑ 5. Volumes for persistence, networks for communication
    → Named volumes persist data, bind mounts sync code.
      docker-compose networks auto-resolve service names to IPs.

☑ 6. docker-compose up for multi-service apps
    → One YAML file orchestrates app + db + cache + proxy.
      Proper startup order, networking, volumes.

☑ 7. Logs and exec for debugging
    → docker logs shows errors, docker exec gives shell access.
      Start with logs to isolate the problem.
```

---

## Production Best Practices

### ✅ **DO**
- Use specific base image versions (`node:18-alpine`, not `latest`)
- Multi-stage builds for production
- Named volumes for databases
- Run as non-root user
- Add HEALTHCHECK
- Use environment variables for config
- One process per container

### ❌ **DON'T**
- Use `latest` tag (use `1.0`, `1.2.0`, etc.)
- Run as root (security risk)
- Store secrets in images
- Layer bloat (combine RUN commands)
- Ignore logs
- Hardcode config (use env vars)

---

## Common Commands Reference

```bash
# Images
docker build -t myapp:1.0 .
docker images
docker tag myapp:1.0 user/myapp:1.0
docker push user/myapp:1.0
docker pull user/myapp:1.0

# Containers
docker run -d -p 3000:3000 myapp:1.0
docker ps
docker logs -f container-id
docker exec -it container-id bash
docker stop container-id
docker rm container-id

# Compose
docker-compose up -d
docker-compose logs -f
docker-compose exec service command
docker-compose down
docker-compose down -v

# Cleanup
docker system prune -a
docker image prune
docker volume prune
```

---

## Next Steps

You now understand:
✅ Images, containers, VMs
✅ Dockerfile optimization
✅ Multi-stage builds
✅ Volumes and networks
✅ docker-compose orchestration
✅ Debugging with logs and exec

**Next: Apply to your real project!**

With these 7 items mastered, you're ready to containerize production applications.
