# InvenTrack — Docker Deployment & Containerization Guide

This guide provides end-to-end instructions for building, running, and managing **InvenTrack** using Docker and Docker Compose.

---

## 1. Architecture & Container Topology

The deployment orchestrates three coordinated containers connected over an isolated Docker bridge network (`wms-network`):

```
                                  [ Client Browser ]
                                      │        │
                   Port 5173 (Web App)│        │Port 8080 (REST / H2)
                                      ▼        ▼
┌────────────────────────────────────────────────────────────────────────┐
│ Docker Network: wms-network                                            │
│                                                                        │
│   ┌────────────────────┐   API Calls    ┌────────────────────┐         │
│   │    wms-frontend    ├───────────────►│    wms-backend     │         │
│   │   (Node 20 / Vite) │                │  (Temurin JRE 17)  │         │
│   └────────────────────┘                └─────────┬──────────┘         │
│                                                   │                    │
│                                                   │ Cache reads/writes │
│                                                   ▼ (Port 6379)        │
│                                         ┌────────────────────┐         │
│                                         │     wms-redis      │         │
│                                         │  (Redis 7-Alpine)  │         │
│                                         └────────────────────┘         │
└────────────────────────────────────────────────────────────────────────┘
```

### Service Summary

| Service | Container Name | Image / Base | Internal Port | Host Port | Health Check |
|---|---|---|---|---|---|
| **Redis** | `wms-redis` | `redis:7-alpine` | `6379` | `6379` | `redis-cli ping` |
| **Backend** | `wms-backend` | `eclipse-temurin:17-jre-alpine` | `8080` | `8080` | `wget http://localhost:8080/h2-console` |
| **Frontend** | `wms-frontend` | `node:20-alpine` (served with `serve`) | `5173` | `5173` | `wget http://localhost:5173` |

---

## 2. Prerequisites

- **Docker Engine**: Version 20.10 or higher
- **Docker Compose**: Version 2.0 or higher (`docker compose` CLI)
- **Host Memory**: Minimum 2 GB free RAM (Spring Boot JVM + Node + Redis)

Check your installed versions:
```bash
docker --version
docker compose version
```

---

## 3. Quick Start

### 1. Build and Run All Services
From the repository root, run:
```bash
docker compose up --build
```
*To run in detached (background) mode, add `-d`:*
```bash
docker compose up --build -d
```

### 2. Verify Startup Sequence
The containers start deterministically thanks to healthcheck conditions:
1. `wms-redis` boots and responds `PONG` to `redis-cli ping`.
2. `wms-backend` initializes after Redis is healthy, runs migrations, seeds initial data, and passes HTTP health check on port 8080.
3. `wms-frontend` starts after the backend is fully healthy.

### 3. Access the Applications

| Interface | URL | Credentials / Notes |
|---|---|---|
| **Frontend Application** | [http://localhost:5173](http://localhost:5173) | See demo accounts below |
| **Backend REST API** | [http://localhost:8080](http://localhost:8080) | Health & REST endpoints |
| **H2 Database Console** | [http://localhost:8080/h2-console](http://localhost:8080/h2-console) | JDBC URL: `jdbc:h2:mem:wmsdb`<br>User: `sa`<br>Password: *(blank)* |

---

## 4. Demo Login Credentials

The backend automatically seeds 4 role-based demo accounts upon startup:

| Role | Email | Password | Assigned Persona & Privileges |
|---|---|---|---|
| **Manager** | `manager@wms.com` | `manager123` | **Yashas** — Full admin, product catalog management, reports, storage locations |
| **Staff** | `staff@wms.com` | `staff123` | **Vinod** — Inventory adjustments, order fulfillment, shipment dispatch |
| **Customer** | `customer@wms.com` | `customer123` | **Vikas** — Product browsing, placing orders, order status tracking |
| **Supplier** | `supplier@wms.com` | `supplier123` | **Vishwas** (`WMS Supplies Inc`) — Purchase order fulfillment, inventory replenishment |

---

## 5. Multi-Stage Dockerfile Highlights

Both services use multi-stage builds to keep production images secure, lightweight, and free from compiler toolchains:

### Backend Multi-Stage Build ([`backend/Dockerfile`](./backend/Dockerfile))
- **Stage 1 (Builder)**: `maven:3.9.6-eclipse-temurin-17` downloads dependencies (`mvn dependency:go-offline`) and packages the JAR without running integration tests.
- **Stage 2 (Runtime)**: `eclipse-temurin:17-jre-alpine` copies only the generated `.jar` file, stripping Maven, JDK tooling, and source code. Final size is reduced by >70%.

### Frontend Multi-Stage Build ([`frontend/Dockerfile`](./frontend/Dockerfile))
- **Stage 1 (Builder)**: `node:20-alpine` runs `npm install` and compiles the React application via Vite (`npm run build`).
- **Stage 2 (Runtime)**: `node:20-alpine` copies only the compiled `/dist` directory and serves it using the lightweight, production-grade `serve` package on port 5173.

---

## 6. Standalone Container Execution (Without Compose)

If you need to build and run the containers individually:

```bash
# 1. Create a shared Docker network
docker network create wms-network

# 2. Run Redis
docker run -d \
  --name wms-redis \
  --network wms-network \
  -p 6379:6379 \
  redis:7-alpine redis-server --appendonly yes

# 3. Build and Run Backend
docker build -t inventrack-backend ./backend

docker run -d \
  --name wms-backend \
  --network wms-network \
  -p 8080:8080 \
  -e SPRING_DATA_REDIS_HOST=wms-redis \
  -e SPRING_DATA_REDIS_PORT=6379 \
  -e SPRING_DATASOURCE_URL="jdbc:h2:mem:wmsdb;DB_CLOSE_ON_EXIT=FALSE;DB_CLOSE_DELAY=-1" \
  inventrack-backend

# 4. Build and Run Frontend
docker build -t inventrack-frontend ./frontend

docker run -d \
  --name wms-frontend \
  --network wms-network \
  -p 5173:5173 \
  -e VITE_API_BASE_URL=http://localhost:8080 \
  inventrack-frontend
```

---

## 7. Configuration & Environment Variables

### Backend Configuration
| Variable | Compose Value | Purpose |
|---|---|---|
| `JAVA_OPTS` | `-Xmx512m -Xms256m` | JVM memory heap tuning |
| `SERVER_PORT` | `8080` | Spring Boot HTTP port |
| `SPRING_DATASOURCE_URL` | `jdbc:h2:mem:wmsdb;...` | H2 JDBC in-memory connection string |
| `SPRING_JPA_HIBERNATE_DDL_AUTO` | `update` | Schema generation strategy |
| `SPRING_DATA_REDIS_HOST` | `redis` | Points to the `wms-redis` service name |
| `SPRING_DATA_REDIS_PORT` | `6379` | Redis port |
| `SPRING_CACHE_TYPE` | `redis` (default) | Set to `none` to disable cache layer |

### Frontend Configuration
| Variable | Compose Value | Purpose |
|---|---|---|
| `VITE_API_BASE_URL` | `http://localhost:8080` | URL consumed by Axios client in the browser |

---

## 8. Managing Containers & Logs

```bash
# View aggregated real-time logs
docker compose logs -f

# View logs for a specific service
docker compose logs -f backend
docker compose logs -f frontend
docker compose logs -f redis

# Check container health and status
docker compose ps

# Restart an individual service
docker compose restart backend

# Stop all containers
docker compose stop

# Stop and tear down containers, networks, and ephemeral volumes
docker compose down -v
```

---

## 9. Running Performance Benchmarks via Compose

To benchmark the application with cache disabled versus enabled:
```bash
# Run with cache completely disabled (override file sets SPRING_CACHE_TYPE=none)
docker compose -f docker-compose.yml -f docker-compose.benchmark.override.yml up --build -d

# Or run the fully automated benchmark script (handles both modes automatically)
.\benchmark-cache.ps1
```
*(See [`REDIS.md`](./REDIS.md) for full benchmarking details).*

---

## 10. Troubleshooting

### Port Conflicts (8080, 5173, or 6379)
If a port is already taken on your host machine, modify the host port binding in `docker-compose.yml`:
```yaml
ports:
  - "8081:8080"  # Expose backend on host 8081
  - "5174:5173"  # Expose frontend on host 5174
  - "6380:6379"  # Expose redis on host 6380
```
*Note: If changing the backend host port, also update `VITE_API_BASE_URL` on the frontend accordingly.*

### Backend Shows "Failed to connect to redis"
- Verify Redis health: `docker compose ps` (should state `wms-redis (healthy)`).
- Check Redis logs: `docker compose logs redis`.
- Ensure backend container has `SPRING_DATA_REDIS_HOST=redis` set.

### H2 Database Resets on Container Restart
- In-memory H2 database (`jdbc:h2:mem:wmsdb`) re-initializes on each restart. Initial seed users, categories, products, and storage locations are recreated automatically by `DataInitializer.java`.
- For persistent production deployments, swap the datasource URL with PostgreSQL or MySQL and mount a Docker volume.
