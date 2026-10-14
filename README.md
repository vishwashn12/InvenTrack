# InvenTrack — Warehouse Inventory & Shipment Management Platform

[![Java](https://img.shields.io/badge/Java-17-orange.svg?logo=openjdk)](https://adoptium.net/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2.5-brightgreen.svg?logo=springboot)](https://spring.io/projects/spring-boot)
[![React](https://img.shields.io/badge/React-18.3-blue.svg?logo=react)](https://react.dev/)
[![Vite](https://img.shields.io/badge/Vite-5.4-purple.svg?logo=vite)](https://vitejs.dev/)
[![Redis](https://img.shields.io/badge/Redis-7.x-red.svg?logo=redis)](https://redis.io/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED.svg?logo=docker)](https://www.docker.com/)

**InvenTrack** is an enterprise-grade Warehouse Management System (WMS) engineered using **Object-Oriented Analysis and Design (OOAD)** principles and **GoF Design Patterns**. It provides end-to-end warehouse workflows: inventory tracking, bin/rack storage management, automated replenishment triggers, purchase orders, customer orders with dynamic pricing, shipment dispatching, multi-role dashboards, and Redis-backed caching for sub-millisecond response times.

---

## 📑 Quick Documentation Links

- **[Docker & Deployment Guide (`DEPLOYMENT.md`)](./DEPLOYMENT.md)** — Multi-stage builds, container topology, networking, standalone execution, and production best practices.
- **[Redis Caching & Performance Guide (`REDIS.md`)](./REDIS.md)** — Cache namespaces, invalidation lifecycle, Lettuce configuration, and automated PowerShell benchmarking.

---

## 🌟 Key Features

- **📦 Real-Time Inventory & Storage Layout**: Granular tracking across Zones, Aisles, Racks, and Bins with low-stock warnings and location re-assignment.
- **🏷️ Product Catalog**: SKU tracking, categorizations, pricing, and soft deactivation.
- **🛒 Order Processing Pipeline**: Multi-item customer orders, automated stock reservation, order parking for out-of-stock items (`PENDING_STOCK`), and status transitions.
- **🏭 Purchase Order Replenishment**: Restocking workflows connecting Managers, Suppliers, and Warehouse Staff.
- **🚚 Shipment Lifecycle Management**: Automatic shipment creation upon order fulfillment with tracking numbers, address validation, and delivery confirmation.
- **👥 Role-Based Access Control (RBAC)**: Fine-grained permissions and dedicated dashboards for **Managers**, **Warehouse Staff**, **Customers**, and **Suppliers**.
- **⚡ Redis Performance Caching**: Multi-tier cache layer for product catalogs and intensive aggregation reports, with sub-5ms cached latency.
- **🔔 Event-Driven Notification Center**: System-wide notifications powered by the Observer pattern across all user roles.

---

## 📐 Architecture & OOAD Design Patterns

The backend follows SOLID design principles, GRASP design patterns (Controller, Information Expert, Low Coupling, High Cohesion), and classic GoF patterns:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                  PRESENTATION LAYER                                    │
│       React 18 SPA (Vite)  ◄─── Axios HTTP Interceptors (Injects X-User-Role)          │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │ REST Requests
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                CONTROLLER & SECURITY LAYER                             │
│   AuthController • ProductController • InventoryController • OrderController           │
│   PurchaseOrderController • ShipmentController • StorageLocationController • Reports  │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              APPLICATION & PATTERN LAYER                               │
│                                                                                        │
│   ┌────────────────────────────────┐        ┌──────────────────────────────────────┐   │
│   │   Facade Pattern (OrderFacade) │        │  Strategy Pattern (PricingStrategy)  │   │
│   │   - Unifies order placement    │        │  - StandardPricing                   │   │
│   │   - Coordinates inventory sync │        │  - DiscountPricing                   │   │
│   │   - Dispatches notifications   │        └──────────────────────────────────────┘   │
│   └───────────────┬────────────────┘                                                   │
│                   │                                                                    │
│   ┌───────────────▼────────────────┐        ┌──────────────────────────────────────┐   │
│   │    Factory & Builder Patterns  │        │   Observer Pattern (Notification)    │   │
│   │    - OrderBuilder              │        │   - WmsObserver / NotificationService│   │
│   │    - OrderFactory              │───────►│   - CustomerObserver, StaffObserver  │   │
│   │    - PurchaseOrderFactory      │        │   - ManagerObserver, SupplierObserver│   │
│   │    - ShipmentFactory           │        └──────────────────────────────────────┘   │
│   └────────────────────────────────┘                                                   │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                   PERSISTENCE LAYER                                    │
│   Spring Data JPA Repositories • Hibernate ORM • Redis Cache (Lettuce) • H2 Database   │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

### GoF Patterns in the Codebase

| Design Pattern | Implementation Class | Role & Architectural Purpose |
|---|---|---|
| **Facade** | [`OrderFacade`](./backend/src/main/java/com/wms/facade/OrderFacade.java) | Simplifies order execution by coordinating stock availability, dynamic pricing, entity persistence, and notification dispatch behind a clean single-method interface. |
| **Builder** | [`OrderBuilder`](./backend/src/main/java/com/wms/builder/OrderBuilder.java) | Assembles complex `Order` aggregates step-by-step with customer info, items, pricing rules, and destination addresses. |
| **Factory** | [`OrderFactory`](./backend/src/main/java/com/wms/factory/OrderFactory.java)<br>[`PurchaseOrderFactory`](./backend/src/main/java/com/wms/factory/PurchaseOrderFactory.java)<br>[`ShipmentFactory`](./backend/src/main/java/com/wms/factory/ShipmentFactory.java) | Decouples caller code from concrete instantiations of domain entities and handles code generation (e.g. `PO-xxx`, `SHP-xxx`). |
| **Observer** | [`WmsObserver`](./backend/src/main/java/com/wms/observer/WmsObserver.java)<br>[`NotificationService`](./backend/src/main/java/com/wms/observer/NotificationService.java) | Event-driven notification system. Subscribes role listeners (`CustomerObserver`, `ManagerObserver`, `StaffObserver`, `SupplierObserver`) to real-time warehouse events. |
| **Strategy** | [`PricingStrategy`](./backend/src/main/java/com/wms/strategy/PricingStrategy.java)<br>[`StandardPricing`](./backend/src/main/java/com/wms/strategy/StandardPricing.java)<br>[`DiscountPricing`](./backend/src/main/java/com/wms/strategy/DiscountPricing.java) | Encapsulates swappable pricing algorithms to calculate net order totals (e.g. standard rates vs bulk promotional discounts). |

---

## 👥 Role-Based Access Control (RBAC) & Demo Accounts

Authentication is managed via `/auth/login`. On successful login, the frontend stores the user profile in `localStorage` and automatically attaches role headers (`X-User-Role`, `X-User-Id`, `X-User-Name`) via an Axios request interceptor.

The database seeds four demo personas upon launch:

| Role | Email | Password | Persona & Scope |
|---|---|---|---|
| **Manager** | `manager@wms.com` | `manager123` | **Yashas** (`EMP-MGR-001`) — Full administrative access: create products, manage storage locations, issue POs, review out-of-stock orders, and inspect system reports. |
| **Staff** | `staff@wms.com` | `staff123` | **Vinod** (`EMP-STF-001`, Shift: `MORNING`) — Warehouse operations: assign and relocate inventory items, receive supplier goods, process orders, and dispatch shipments. |
| **Customer** | `customer@wms.com` | `customer123` | **Vikas** (`CUST-001`) — Catalog browsing, placing new orders, and tracking order delivery status. |
| **Supplier** | `supplier@wms.com` | `supplier123` | **Vishwas** (`WMS Supplies Inc`, `SUP-001`) — Vendor operations: receive POs sent by managers and mark supplies as delivered. |

---

## 🚀 Getting Started

### Option 1: Docker Compose (Fastest & Recommended)

Ensure Docker Desktop or Docker Engine is running, then execute:

```bash
docker compose up --build
```

This starts:
- **Frontend SPA**: [http://localhost:5173](http://localhost:5173)
- **Backend REST API**: [http://localhost:8080](http://localhost:8080)
- **H2 Database Console**: [http://localhost:8080/h2-console](http://localhost:8080/h2-console)
- **Redis Cache**: `localhost:6379`

To run in the background:
```bash
docker compose up --build -d
```
To stop all containers and networks:
```bash
docker compose down -v
```

*(See [`DEPLOYMENT.md`](./DEPLOYMENT.md) for individual container builds and deployment options).*

---

### Option 2: Manual Local Development

#### Prerequisites
- **Java 17 JDK** (e.g. Eclipse Temurin 17)
- **Maven 3.8+**
- **Node.js 18+** & **npm**
- **Redis** (optional; defaults to `localhost:6379`, falls back to in-memory if stopped)

#### 1. Start Redis (Optional but Recommended)
```bash
docker run -d --name wms-redis -p 6379:6379 redis:7-alpine
```

#### 2. Run Backend
```bash
cd backend
mvn spring-boot:run
```
- API Base: `http://localhost:8080`
- H2 Console: `http://localhost:8080/h2-console`
  - **JDBC URL**: `jdbc:h2:mem:wmsdb`
  - **Username**: `sa`
  - **Password**: *(leave blank)*

#### 3. Run Frontend
```bash
cd frontend
npm install
npm run dev
```
- Application: `http://localhost:5173`

---

## 📡 REST API Reference

All requests accept and return `application/json`. Authenticated routes require role verification passed via `X-User-Role`.

### Authentication
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `POST` | `/auth/login` | Public | Authenticates credentials and returns user token and profile |
| `GET` | `/auth/me` | Public | Returns current session details from headers |

### Product Catalog
| Method | Endpoint | Access | Cache Behavior | Description |
|---|---|---|---|---|
| `GET` | `/products` | All Roles | `@Cacheable("products-list")` (1 hr) | Retrieves active catalog items |
| `GET` | `/products/{id}` | All Roles | `@Cacheable("products-by-id")` (1 hr) | Retrieves a single active product |
| `POST` | `/products` | `MANAGER` | Evicts `products-list` | Creates a new product SKU |
| `PUT` | `/products/{id}` | `MANAGER` | Evicts `products-list`, `products-by-id` | Updates product metadata and pricing |
| `PATCH` | `/products/{id}/deactivate` | `MANAGER` | Evicts `products-list`, `products-by-id` | Soft-deactivates a product |

### Inventory & Storage Locations
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/inventory` | All Roles | Lists all inventory stock items with quantities and locations |
| `GET` | `/inventory/check` | All Roles | Verifies stock availability for a product ID and quantity |
| `POST` | `/inventory/assign` | `STAFF` | Assigns an unassigned inventory item to a storage location |
| `POST` | `/inventory/move` | `STAFF` | Relocates inventory between storage locations |
| `GET` | `/storage-locations` | All Roles | Lists warehouse storage bins (Zone, Aisle, Rack, Bin) |
| `POST` | `/storage-locations` | `MANAGER` | Creates a new warehouse storage bin |
| `PUT` | `/storage-locations/{id}` | `MANAGER` | Modifies an existing storage location |

### Orders & Fulfillment
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/orders` | All Roles | Lists customer orders |
| `GET` | `/orders/pending-stock` | `MANAGER` | Lists orders waiting on inventory replenishment |
| `POST` | `/orders` | `CUSTOMER` | Places an order (via `OrderFacade` & `PricingStrategy`) |
| `POST` | `/orders/process/{id}` | `STAFF` | Allocates stock, completes order, and creates shipment |
| `DELETE` | `/orders/{id}` | `CUSTOMER` | Cancels a pending order |

### Purchase Orders & Suppliers
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/purchase-orders` | All Roles | Lists all purchase orders |
| `POST` | `/purchase-orders` | `MANAGER` | Creates a new PO to restock inventory from a supplier |
| `POST` | `/purchase-orders/send/{id}` | `MANAGER` | Transitions PO status to `SENT_TO_SUPPLIER` |
| `PATCH` | `/purchase-orders/{id}/deliver`| `SUPPLIER` | Marks supplier delivery (`DELIVERED_BY_SUPPLIER`) |
| `POST` | `/purchase-orders/{id}/receive`| `STAFF` | Verifies physical receipt, stocks inventory, marks `RECEIVED` |
| `GET` | `/suppliers` | All Roles | Returns registered vendor suppliers |

### Shipments & Tracking
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/shipments` | All Roles | Lists all warehouse shipments |
| `GET` | `/shipments/{id}` | All Roles | Tracks a specific shipment by ID |
| `PATCH` | `/shipments/{id}/ship` | `STAFF` | Dispatches shipment (`CREATED` → `SHIPPED`) |
| `PATCH` | `/shipments/{id}/deliver` | `STAFF` | Confirms final delivery (`SHIPPED` → `DELIVERED`) |

### Reports & Notifications
| Method | Endpoint | Access | Cache Behavior | Description |
|---|---|---|---|---|
| `GET` | `/reports/inventory` | `MANAGER` | `@Cacheable("inventory-report")` (1 hr) | Stock summary with low-stock flags |
| `GET` | `/reports/orders` | `MANAGER` | `@Cacheable("order-report")` (1 hr) | Aggregated order distribution by status |
| `GET` | `/reports/shipments` | `MANAGER` | `@Cacheable("shipment-report")` (1 hr) | Shipment transit times and statuses |
| `GET` | `/notifications` | All Roles | Real-time | Retrieves the last 50 system observer events |

---

## ⚡ Performance & Cache Benchmarking

InvenTrack includes an automated PowerShell benchmarking suite: [`benchmark-cache.ps1`](./benchmark-cache.ps1). It executes identical request workloads against the application in two states:
1. **`NO_CACHE`**: Launches the backend with [`docker-compose.benchmark.override.yml`](./docker-compose.benchmark.override.yml) (`SPRING_CACHE_TYPE=none`).
2. **`REDIS_CACHE`**: Launches with Redis caching enabled and primes cache keys.

### Running the Benchmark
```powershell
.\benchmark-cache.ps1 -ProductsRequests 300 -ReportsRequests 200 -WarmupRequests 20
```

The script benchmarks response latencies (Average, P95), throughput (req/sec), error rates, and queries Redis hit-ratio stats directly from `redis-cli INFO stats`.

*(See [`REDIS.md`](./REDIS.md) for full benchmark documentation and sample output metrics).*

---

## 📂 Repository Structure

```text
InvenTrack-Inventory-Shipment-Management-Platform/
├── backend/                                # Spring Boot 3.2.5 REST Application
│   ├── Dockerfile                          # Multi-stage Maven/Temurin JRE build
│   ├── pom.xml                             # Dependencies: JPA, Redis, Jedis, H2, Lombok
│   └── src/main/java/com/wms/
│       ├── builder/                        # Builder Pattern (OrderBuilder)
│       ├── config/                         # AppConfig, DataInitializer, RedisConfig, WebConfig
│       ├── controller/                     # REST API Controllers (RBAC guarded)
│       ├── dto/                            # Data Transfer Objects
│       ├── facade/                         # Facade Pattern (OrderFacade)
│       ├── factory/                        # Factory Pattern (Order, PO, Shipment Factories)
│       ├── model/                          # Domain Entities (core & user models)
│       ├── observer/                       # Observer Pattern (NotificationService & Listeners)
│       ├── repository/                     # Spring Data JPA Repositories
│       ├── service/                        # Business Logic Services
│       └── strategy/                       # Strategy Pattern (Standard & Discount Pricing)
├── frontend/                               # React 18 + Vite 5 Application
│   ├── Dockerfile                          # Multi-stage Node 20 build with 'serve'
│   ├── package.json                        # Vite, React, Axios
│   └── src/
│       ├── api/                            # Axios API client & request interceptors
│       ├── components/                     # Reusable UI components & layouts
│       ├── context/                        # Global state / Authentication context
│       ├── pages/                          # Role-specific dashboards (Manager, Staff, etc.)
│       └── styles/                         # Vanilla CSS design system
├── DEPLOYMENT.md                           # Comprehensive Docker & Deployment guide
├── REDIS.md                                # Redis caching & benchmarking documentation
├── benchmark-cache.ps1                     # Automated PowerShell benchmark script
├── docker-compose.yml                      # Production Docker Compose orchestration
├── docker-compose.benchmark.override.yml   # Compose override to disable caching
├── .gitignore
└── README.md                               # Main project documentation
```

---

## 🛠️ Technology Stack

| Layer | Technology |
|---|---|
| **Backend Framework** | Spring Boot 3.2.5 (Java 17) |
| **Data Persistence** | Spring Data JPA, Hibernate, In-memory H2 Database |
| **Caching Layer** | Redis 7 (Alpine), Spring Data Redis, Lettuce Client |
| **Frontend Framework** | React 18.3, Vite 5.4, Axios 1.9, Vanilla CSS |
| **Architecture** | Object-Oriented Analysis & Design (OOAD), GoF Design Patterns, RESTful APIs |
| **Containerization** | Docker, Docker Compose (Multi-stage builds) |
| **Benchmarking** | PowerShell automated testing suite |

---

## 📄 License & Academic Note

This project was developed as an advanced **Object-Oriented Analysis and Design (OOAD)** Warehouse Management System platform demonstrating enterprise design patterns, clean architecture, role-based workflows, and distributed caching. Available for academic, demonstration, and portfolio review.
