# BuildUp Marketplace

A multi-vendor e-commerce marketplace built with **NestJS, Next.js, PostgreSQL and Redis**. Customers shop across many sellers in a single cart, sellers manage their own shops, and admins oversee the platform.

The project focuses on the backend problems real marketplaces face: **role-based access control, safe stock handling under concurrent purchases, and reliable payment webhooks**.

> **Status:** 🚧 In active development (Sprint 1 started on 2026-11-01). Features marked ⏳ are planned, not yet implemented.

---

## Table of Contents

- [BuildUp Marketplace](#buildup-marketplace)
  - [Table of Contents](#table-of-contents)
  - [Highlights](#highlights)
  - [Features](#features)
  - [Tech Stack](#tech-stack)
  - [Architecture](#architecture)
  - [Roles and Permissions](#roles-and-permissions)
  - [Project Structure](#project-structure)
  - [Getting Started](#getting-started)
    - [Prerequisites](#prerequisites)
    - [1. Clone and configure](#1-clone-and-configure)
    - [2. Start the infrastructure](#2-start-the-infrastructure)
    - [3. Run the API](#3-run-the-api)
    - [4. Run the web app](#4-run-the-web-app)
    - [Useful commands](#useful-commands)
  - [API Documentation](#api-documentation)
  - [Roadmap](#roadmap)
  - [Documentation](#documentation)
  - [Author](#author)

---

## Highlights

| Challenge | Approach |
|---|---|
| **Authentication** | JWT access token plus rotating refresh token in an httpOnly cookie, refresh tokens stored hashed, revocation via Redis, rate limiting against brute force ⏳ |
| **Authorization** | RBAC with `CUSTOMER`, `SELLER`, `ADMIN` using NestJS Guards, plus resource ownership checks to prevent IDOR ⏳ |
| **Multi-vendor orders** | One cart can contain products from many sellers; checkout splits it into one **Order** and several **SubOrders**, one per seller ⏳ |
| **Overselling prevention** | Database transactions with row locking, temporary stock reservation in Redis, flash sales with limited quantity, verified by a concurrent load test ⏳ |
| **Payments** | VNPay sandbox integration with signature verification and **idempotent webhooks** that tolerate duplicate or late callbacks ⏳ |
| **One-command setup** | Docker Compose for PostgreSQL and Redis, environment-based configuration, CI with GitHub Actions ⏳ |

Measured results (for example, "500 concurrent purchase requests, zero negative stock") will be added here once the load tests are done.

---

## Features

**Customer**
- ⏳ Register, log in, log out, refresh session
- ⏳ Browse, search and filter products with pagination
- ⏳ Cart with items from multiple sellers
- ⏳ Checkout, online payment, order history and tracking

**Seller**
- ⏳ Apply to become a seller
- ⏳ Manage products, images and stock
- ⏳ Process the sub-orders that belong to their shop
- ⏳ View sales statistics

**Admin**
- ⏳ Approve or suspend sellers
- ⏳ Manage categories and users
- ⏳ Platform-wide revenue dashboard and audit log

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js (App Router), TypeScript, Tailwind CSS |
| Backend | NestJS, TypeScript |
| Database | PostgreSQL 16, Prisma ORM |
| Cache and tokens | Redis 7 |
| Infrastructure | Docker, Docker Compose, GitHub Actions |
| API docs | Swagger / OpenAPI |
| Integrations (planned) | VNPay sandbox, Cloudinary |

---

## Architecture

```
┌──────────────┐        HTTPS / JSON        ┌────────────────────────────┐
│   Next.js    │ ─────────────────────────► │         NestJS API         │
│  (apps/web)  │ ◄───────────────────────── │        (apps/api)          │
└──────────────┘      cookies + tokens      │                            │
                                            │  Controller → Service      │
                                            │        │          │        │
                                            │   Guards (RBAC)   │        │
                                            └────────┼──────────┼────────┘
                                                     │          │
                                           ┌─────────▼──┐  ┌────▼─────┐
                                           │ PostgreSQL │  │  Redis   │
                                           │  (Prisma)  │  │ tokens,  │
                                           │            │  │ cache,   │
                                           └────────────┘  │ stock    │
                                                           │ holds    │
                                                           └──────────┘
```

Backend modules follow a strict flow: **Controller → Service → Prisma**. Controllers only handle HTTP, services hold business logic, and every input is validated through DTOs.

The full entity-relationship diagram will be available in [`docs/erd.png`](docs/erd.png) and design decisions in [`docs/decisions.md`](docs/decisions.md).

---

## Roles and Permissions

| Action | Customer | Seller | Admin |
|---|:---:|:---:|:---:|
| Browse products | ✅ | ✅ | ✅ |
| Place and view own orders | ✅ | ✅ | ✅ |
| Create and edit own products | ❌ | ✅ | ✅ |
| Process sub-orders of own shop | ❌ | ✅ | ❌ |
| Approve or suspend sellers | ❌ | ❌ | ✅ |
| Manage categories and users | ❌ | ❌ | ✅ |
| View platform-wide statistics | ❌ | ❌ | ✅ |

Besides role checks, ownership checks ensure a seller can only modify their own resources.

---

## Project Structure

```
.
├── apps/
│   ├── api/                 # NestJS backend
│   │   ├── prisma/          # schema, migrations, seed
│   │   └── src/             # feature modules (auth, users, products, orders, ...)
│   └── web/                 # Next.js frontend
├── docs/                    # ERD, design decisions, screenshots, demo
├── docker-compose.yml       # PostgreSQL + Redis
├── .env.example
├── CLAUDE.md
└── README.md
```

---

## Getting Started

### Prerequisites

- Node.js 20 or newer
- Docker and Docker Compose
- Git

### 1. Clone and configure

```bash
git clone <your-repo-url>
cd <your-repo-folder>
cp .env.example .env
```

Edit `.env` if you want different ports or credentials. Never commit `.env`.

### 2. Start the infrastructure

```bash
docker compose up -d
```

This starts PostgreSQL and Redis. Data persists in a Docker volume between restarts.

### 3. Run the API

```bash
cd apps/api
npm install
npx prisma migrate dev
npx prisma db seed
npm run start:dev
```

The API runs at `http://localhost:3000`. Health check: `GET http://localhost:3000/health`.

### 4. Run the web app

```bash
cd apps/web
npm install
npx next dev -p 3001
```

The web app runs at `http://localhost:3001`.

### Useful commands

| Command | Description |
|---|---|
| `docker compose down` | Stop PostgreSQL and Redis (data is kept) |
| `npx prisma studio` | Browse database content (run in `apps/api`) |
| `npm run test` | Run unit tests (run in `apps/api`) |
| `npm run test:e2e` | Run end-to-end tests (run in `apps/api`) |

---

## API Documentation

Interactive Swagger documentation is available at `http://localhost:3000/docs` while the API is running.

---

## Roadmap

Development follows **Scrum** with one-week sprints.

| Sprint | Focus | Status |
|---|---|---|
| 1 | Foundation: repo, Docker Compose, ERD, Prisma, first API, validation, Swagger | 🚧 In progress |
| 2 | Authentication (register, login, JWT, refresh token rotation) | ⏳ Planned |
| 3 | Authorization (RBAC), seller onboarding | ⏳ Planned |
| 4 | Products: images, search, Redis caching | ⏳ Planned |
| 5 | Cart and multi-seller orders | ⏳ Planned |
| 6 | Inventory and flash sale (concurrency control) | ⏳ Planned |
| 7 | VNPay payment and idempotent webhooks | ⏳ Planned |
| 8 | Admin dashboard and start of frontend | ⏳ Planned |
| 9–10 | Frontend for all three roles | ⏳ Planned |
| 11 | Testing, deployment, documentation, demo | ⏳ Planned |

Dates and scope may be adjusted after each sprint review.

---

## Documentation

- Entity-relationship diagram: `docs/erd.png` (coming in Sprint 1)
- Design decisions: `docs/decisions.md` (coming in Sprint 1)
- Swagger screenshot and demo video: `docs/` (coming in Sprint 1)
- Live demo: _link will be added after deployment_

---

## Author

**Your Name**
- GitHub: [@your-username](https://github.com/your-username)
- LinkedIn: _add your link_
- Email: _add your email_

---

_This is a learning and portfolio project. Payment features run in sandbox mode only and do not process real money._