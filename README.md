# Task Tracker

A production-grade microservice system demonstrating event-driven architecture, real-time WebSocket notifications, and distributed infrastructure — built as a portfolio project with a focus on engineering decisions over CRUD.

---

## Overview

Task Tracker allows users to:
- Register and authenticate via JWT
- Create, update, filter, and delete tasks
- Receive real‑time notifications about task changes
- Get email reports based on scheduled checks and Kafka events

The project is organized as a Maven multi‑module monorepo and is suitable both as a learning project and as a portfolio‑grade showcase.

---

## Features

- **JWT authentication & authorization**
  - `/auth/login`, `/auth/refresh` for issuing and refreshing tokens
  - `/user` endpoints for registration, profile retrieval, update, and deletion
- **Task management**
  - `/tasks` REST API for CRUD, filtering, and pagination (via `TaskFilter`)
- **Real‑time updates**
  - WebSocket + STOMP over `/ws`, with user‑specific destinations via `/user/{username}/topic/...`
- **Scheduling & notifications**
  - Dedicated `task-tracker-scheduler` service that periodically scans tasks and publishes Kafka events
  - `task-tracker-email-sender` service consumes Kafka topics and sends email reports
- **Rate limiting**
  - Custom `@RateLimit` annotation, Redis + Bucket4j‑based rate limiting on sensitive endpoints (e.g. login, refresh)
- **Security**
  - Spring Security, BCrypt password hashing, CORS configuration, per‑user authorization checks in controllers
- **Infrastructure**
  - MySQL, Redis, Kafka, Zookeeper, Vault, all wired through Docker Compose
  - Flyway for DB migrations, centralized configuration via Vault

---

## Modules

- `common` – shared DTOs and classes used across backend, scheduler, and email‑sender.
- `task-tracker-backend` – main REST API:
  - authentication, users, tasks
  - WebSocket configuration (`WebSocketConfig`) and interceptors
  - JWT, rate limiting, caching (`@EnableCaching`)
- `task-tracker-scheduler` – Spring Boot service with scheduled jobs:
  - reads tasks from MySQL
  - publishes reports to Kafka topics (e.g. `all-tasks-topic`, `unfinished-tasks-topic`, `finished-tasks-topic`)
- `task-tracker-email-sender` – Kafka consumer + SMTP email sender:
  - `EmailKafkaListener` consumes Kafka topics and maps messages to email DTOs
- `task-tracker-frontend` – Next.js (React, TypeScript) SPA:
  - talks to backend via `/api` proxy and `axios` client with JWT interceptors
  - integrates with WebSocket/STOMP for live updates

---

## Technology Stack

- **Backend & services**
  - Java 17+ (Spring Boot 3.x, Spring Security, Spring Data JPA, Spring Kafka, Spring WebSocket, Spring Scheduling)
  - Hibernate, Flyway, Redis (caching & rate limiting), Redisson, Bucket4j
  - Vault for configuration and secrets management
- **Frontend**
  - Next.js (React), TypeScript, axios, STOMP.js / SockJS
- **Data & messaging**
  - MySQL
  - Kafka + Zookeeper
- **Infrastructure / tooling**
  - Docker, Docker Compose
  - Maven (multi‑module)
  - Makefile for build & run shortcuts

---

## Project Structure

```text
.
├── common/                      # Shared DTOs and utilities
├── task-tracker-backend/        # Core REST API + WebSocket + security
├── task-tracker-scheduler/      # Scheduler microservice (Kafka producer)
├── task-tracker-email-sender/   # Email microservice (Kafka consumer + SMTP)
├── task-tracker-frontend/       # Next.js frontend
├── Docker-compose.yml           # All services + infra
├── Makefile                     # Helper targets for build/run
└── .env.dev                     # Environment variables for Docker compose
```

---

## Architecture Decisions

- **Why microservices?** Scheduler and email-sender are isolated 
  to allow independent scaling and deployment
- **Why Kafka?** Decouples task events from notification delivery — 
  email-sender can fail without affecting core API
- **Why Vault?** Secrets management from day one, 
  not as an afterthought
- **Why Redis + Bucket4j?** Rate limiting at application level 
  without nginx dependency

---

## Running with Docker (recommended)

### Prerequisites

- JDK 17+ (for building Java modules)
- Docker & Docker Compose
- Maven 3.9+

### 1. Configure environment

The file `.env.dev` contains the environment variables used by Docker Compose, for example:

- `DB_HOST`, `DB_PORT`, `DB_NAME`
- `MYSQL_ROOT_PASSWORD`
- `REDIS_HOST`, `REDIS_PORT`
- `VAULT_URI`, `VAULT_TOKEN`
- `MAIL_HOST`, `MAIL_PORT`
- `KAFKA_BROKER`, `TRUSTED_KAFKA_PACKAGES`

Adjust these values if necessary for your environment.

### 2. Build and start all services

From the repository root:

```sh
make build
# or, to only start containers (if artifacts are already built)
make run
```

This will:
- Build backend, scheduler, and email‑sender JARs with Maven
- Start Docker Compose with:
  - MySQL (`3306`)
  - Redis (`6379`)
  - Kafka (`9092`) + Zookeeper (`2181`)
  - Vault (`8200`)
  - Backend (`8080`)
  - Frontend (`80`)
  - Scheduler and email‑sender services

### 3. Access the application

- **Frontend**: `http://localhost` (Next.js app behind the `frontend` container)
- **Backend API**: `http://localhost:8080`
- **Vault UI**: `http://localhost:8200` (dev mode, token from `.env.dev`)

To stop all services:

```sh
make stop
```

---

## Local Development without Docker (optional)

You can also run services directly from your IDE:

1. Start infrastructure manually (or via Docker):
   - MySQL
   - Redis
   - Kafka + Zookeeper
   - Vault (optional but recommended)
2. Configure `application.yml` / environment variables for each service to point to your local infrastructure.
3. Run Spring Boot applications:
   - `task-tracker-backend` – main API
   - `task-tracker-scheduler`
   - `task-tracker-email-sender`
4. Run the frontend:

   ```sh
   cd task-tracker-frontend
   npm install
   npm run dev
   ```

   By default it runs on `http://localhost:3000` and proxies API requests to the backend (`/api`).

---

## API Overview

- **Authentication:**
  - `POST /auth/login` – login with credentials, returns JWT tokens and user data
  - `POST /auth/refresh` – refresh access token using refresh token
- **User:**
  - `POST /user` – registration (returns tokens on success)
  - `GET /user` – get current authenticated user
  - `PATCH /user/{id}` – update current user
  - `DELETE /user/{id}` – delete current user and their tasks
- **Tasks:**
  - `GET /tasks` – list tasks for current user, with filtering via `TaskFilter`
  - `POST /tasks` – create task
  - `PATCH /tasks/{id}` – partial update
  - `DELETE /tasks/{id}` – delete task

All protected endpoints expect an `Authorization: Bearer <accessToken>` header.

---

## WebSocket Usage

- STOMP endpoint: `/ws`
- Allowed origins are configured via the `cors.allowed-origins` property.
- Typical flow:
  1. Connect to `/ws` with the user’s JWT for authentication (handled by `WebSocketAuthChannelInterceptor`).
  2. Subscribe to user‑specific destinations such as:
     - `/user/{username}/topic/notifications`
  3. Receive real‑time updates about task changes.

---

## Notes on Quality and Design

This project demonstrates:

- Clean separation between controllers, services, DTOs, and persistence
- Event‑driven architecture with Kafka
- Real‑time user experience via WebSocket/STOMP
- Proper use of Redis, Vault, Flyway, and Docker for a production‑like setup

It can be used both as a learning playground for modern Spring/Next.js practices and as a portfolio project.

---

## Contact

- Telegram: [@metara5h](https://t.me/metara5h)
