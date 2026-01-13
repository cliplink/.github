# Scalable URL Shortener System

A production-ready, microservices-based URL shortening platform designed for high concurrency and scalability.

This project is organized as a set of **independent repositories** within a GitHub Organization. This **Polyrepo** architecture ensures strict separation of concerns, while maintaining strict type safety across service boundaries via shared NPM packages.

---

## 🛠 Tech Stack

### Core
![TypeScript](https://img.shields.io/badge/TypeScript-007ACC?style=for-the-badge&logo=typescript&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-43853D?style=for-the-badge&logo=node.js&logoColor=white)
![NestJS](https://img.shields.io/badge/NestJS-E0234E?style=for-the-badge&logo=nestjs&logoColor=white)
![Nuxt](https://img.shields.io/badge/Nuxt-002E3B?style=for-the-badge&logo=nuxt.js&logoColor=white)

### Data & Messaging
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)
![NATS](https://img.shields.io/badge/NATS-27AAE1?style=for-the-badge&logo=nats&logoColor=white)
![BullMQ](https://img.shields.io/badge/BullMQ-Status-orange?style=for-the-badge)

### Infrastructure & CI/CD
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=for-the-badge&logo=github-actions&logoColor=white)

---

## 🏗 Architecture Highlights

### 1. Contract-First Development (Shared NPM Packages)
Despite being physically separated repositories, the system acts as a cohesive unit thanks to **automated contract publishing**:
- **Backend** and **Click-Worker** repositories contain a `src/_contracts` directory.
- On merge to `main`, GitHub Actions automatically build and publish these contracts as versioned **NPM packages** to the GitHub Registry.
- **Data Flow Safety:**
  - `click-worker` exports events (e.g., `ClickCreatedEvent`). -> **Backend** imports this package to publish strictly typed events to NATS.
  - `backend` exports API DTOs. -> **Frontend** imports this package to make strictly typed HTTP requests.

### 2. Event-Driven Analytics (NATS JetStream)
Click tracking is fully decoupled using **Event-Driven Architecture**:
- **Gateway:** The Backend publishes a lightweight event to **NATS JetStream** immediately after redirecting the user.
- **Consumer:** A dedicated `click-worker` microservice subscribes to the stream.
- **Reliability:** JetStream ensures at-least-once delivery and durable message persistence.

### 3. Database Partitioning & Archiving
- **PostgreSQL Native Partitioning** is used for the `links` table.
- **Automation:** A **Redis-backed BullMQ repeatable job** creates monthly partitions in advance and archives old ones to a separate schema, keeping the active dataset lightweight.

### 4. High-Performance Writes (Batching)
- The `click-worker` buffers incoming NATS events and flushes them to PostgreSQL in **bulk batches**. This drastically reduces database IOPS.

---

## 🧩 Repositories & Services

### [Backend (NestJS)](backend)
The core API service.
- **Features:** Link management, Auth (Passport: Local/JWT), Partition Scheduling.
- **CI/CD:**
  - 📦 **NPM:** Publishes API contracts for Frontend consumption.
  - 📦 **Docker:** Builds production images to GHCR.
  - **Dependency:** Consumes `click-worker` contracts for NATS events.

### [Click Worker (NestJS)](click-worker)
Background processor.
- **Features:** NATS Consumer, Batch Database Writer.
- **CI/CD:**
  - 📦 **NPM:** Publishes NATS Event contracts (e.g. `ClickCreatedEvent`) for Backend consumption.
  - 📦 **Docker:** Builds independent deployment artifact.

### [Frontend (Nuxt.js)](frontend)
Modern SSR User Interface.
- **Features:** User dashboard, authentication flows.
- **Dependency:** Consumes `backend` contracts for 100% type-safe API integration.
- **CI/CD:** Builds SSR-ready Docker image.

### [Utils](utils)
Shared library.
- **Purpose:** Common utilities and helpers shared between Backend and Worker.
- **CI/CD:** Automatically versioned and published as a private NPM package.

---

## 🚀 DevOps & Development

The project includes uniform tooling across all repositories:

1.  **Quality Gate:** Strict ESLint and Prettier configurations.
2.  **Artifact Versioning:** Automated Semantic Versioning for all NPM packages.
3.  **Deployment:** Docker Compose configurations are provided for:
    - `dev`: Hot-reload enabled for all services.
    - `prod`: Restart policies and optimized images.
    - `test`: Tmpfs volumes for fast, stateless testing.

```bash
# Example: Start the full stack in production mode
npm run docker:prod
```
