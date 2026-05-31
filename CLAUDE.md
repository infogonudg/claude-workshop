# CLAUDE.md — claude-workshop

> **Project:** Amazon-style Landing Page (POC)
> **Budget:** Low-cost / proof-of-concept
> **Audience:** Office-going professionals (formal tone throughout)

---

## Table of Contents

1. [Project Purpose](#1-project-purpose)
2. [Architecture Overview](#2-architecture-overview)
3. [Technology Stack](#3-technology-stack)
4. [Folder Structure](#4-folder-structure)
5. [Services & Modules](#5-services--modules)
6. [MCP Servers](#6-mcp-servers)
7. [Skills](#7-skills)
8. [Style Rules](#8-style-rules)
9. [Customer Data Collection](#9-customer-data-collection)
10. [Key Commands — Run](#10-key-commands--run)
11. [Key Commands — Test](#11-key-commands--test)
12. [Load Balancing](#12-load-balancing)
13. [Unit Testing Standards](#13-unit-testing-standards)

---

## 1. Project Purpose

Build a production-ready, Amazon-style e-commerce landing page as a **Proof of Concept (POC)**.  
The page must:

- Display featured products, categories, and promotional banners.
- Support user authentication and session management.
- Capture and persist customer information securely.
- Be modular, independently deployable, and cost-efficient.

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────────┐
│                    Client (Browser)                  │
│              Next.js (TypeScript) Frontend           │
└───────────────────────┬──────────────────────────────┘
                        │ HTTPS / REST / WebSocket
┌───────────────────────▼──────────────────────────────┐
│                   API Gateway (Nginx)                │
│              Load Balancer + Rate Limiter            │
└──────┬────────────┬──────────────┬───────────────────┘
       │            │              │
┌──────▼───┐  ┌─────▼─────┐  ┌───▼──────────┐
│  Auth    │  │ Product   │  │  Customer    │
│ Service  │  │ Service   │  │  Service     │
│(FastAPI) │  │(FastAPI)  │  │  (FastAPI)   │
└──────┬───┘  └─────┬─────┘  └───┬──────────┘
       │            │             │
┌──────▼─────────────▼─────────────▼──────────┐
│              Supabase (PostgreSQL)           │
│   Auth · Products · Orders · Customers      │
└─────────────────────────────────────────────┘
       │
┌──────▼─────────────────┐
│   MCP Service Layer    │
│  (All MCPs in one pod) │
└────────────────────────┘
```

---

## 3. Technology Stack

| Layer | Technology | Rationale |
|---|---|---|
| Frontend | **Next.js 14 (TypeScript)** | SSR/SSG, SEO, fast page loads |
| Backend Services | **Python 3.11 + FastAPI** | Lightweight, async, low cost |
| Database | **Supabase (PostgreSQL)** | Free tier, built-in auth, real-time |
| Auth | **Supabase Auth** | Zero-cost JWT-based authentication |
| File Storage | **Supabase Storage** | Product images, customer uploads |
| MCP Layer | **Claude MCP SDK** | Modular AI-driven tool integrations |
| Reverse Proxy | **Nginx** | Load balancing, SSL termination |
| Containerisation | **Docker + Docker Compose** | Local dev & POC deployment |
| CI/CD | **GitHub Actions** | Free tier, automated testing |
| Testing (Python) | **Pytest + httpx** | Unit + integration tests |
| Testing (TS) | **Vitest + Testing Library** | Component & unit tests |
| Monitoring | **Prometheus + Grafana** | Free, self-hosted observability |
| Secret Management | **dotenv (.env files)** | Simple; upgrade to Vault for prod |

> **Additional stack note:** Nginx is introduced solely for load balancing across service replicas. Prometheus + Grafana are self-hosted on the same Docker network to keep costs at zero during POC.

---

## 4. Folder Structure

```
claude-workshop/
├── CLAUDE.md                  # This file
│
├── frontend/                  # Next.js (TypeScript) — Landing Page
│   ├── src/
│   │   ├── app/               # App Router pages
│   │   ├── components/        # Reusable UI components
│   │   ├── hooks/             # Custom React hooks
│   │   ├── lib/               # Supabase client, utilities
│   │   └── types/             # Shared TypeScript types
│   ├── public/
│   ├── vitest.config.ts
│   ├── package.json
│   └── tsconfig.json
│
├── services/
│   ├── auth-service/          # Authentication microservice (FastAPI)
│   │   ├── app/
│   │   │   ├── main.py
│   │   │   ├── routes/
│   │   │   ├── models/
│   │   │   └── schemas/
│   │   ├── tests/
│   │   ├── requirements.txt
│   │   └── Dockerfile
│   │
│   ├── product-service/       # Product catalogue microservice (FastAPI)
│   │   ├── app/
│   │   │   ├── main.py
│   │   │   ├── routes/
│   │   │   ├── models/
│   │   │   └── schemas/
│   │   ├── tests/
│   │   ├── requirements.txt
│   │   └── Dockerfile
│   │
│   └── customer-service/      # Customer data microservice (FastAPI)
│       ├── app/
│       │   ├── main.py
│       │   ├── routes/
│       │   ├── models/
│       │   └── schemas/
│       ├── tests/
│       ├── requirements.txt
│       └── Dockerfile
│
├── mcp/                       # ALL MCP servers — single unified service folder
│   ├── mcp-server/            # Main MCP server entry point
│   │   ├── server.py
│   │   ├── tools/             # MCP tool definitions
│   │   │   ├── product_tools.py
│   │   │   ├── customer_tools.py
│   │   │   └── auth_tools.py
│   │   ├── resources/         # MCP resource definitions
│   │   └── requirements.txt
│   ├── Dockerfile
│   └── README.md              # MCP service documentation
│
├── infra/
│   ├── nginx/
│   │   └── nginx.conf         # Reverse proxy + load balancer config
│   ├── monitoring/
│   │   ├── prometheus.yml
│   │   └── grafana/
│   └── docker-compose.yml     # Orchestrates all services
│
├── scripts/
│   ├── run_all.sh             # Start all services locally
│   ├── test_all.sh            # Run all test suites
│   └── seed_db.py             # Seed Supabase with sample data
│
├── docs/
│   ├── run.md                 # Running guide (auto-generated)
│   └── test.md                # Testing guide (auto-generated)
│
└── .env.example               # Environment variable template
```

---

## 5. Services & Modules

### 5.1 Auth Service (`services/auth-service/`)
- **Responsibility:** Register, login, logout, session refresh via Supabase Auth.
- **Endpoints:** `POST /auth/register`, `POST /auth/login`, `POST /auth/logout`, `GET /auth/me`
- **Port:** `8001`

### 5.2 Product Service (`services/product-service/`)
- **Responsibility:** CRUD for products, categories, banners, and search.
- **Endpoints:** `GET /products`, `GET /products/{id}`, `GET /categories`, `GET /banners`
- **Port:** `8002`

### 5.3 Customer Service (`services/customer-service/`)
- **Responsibility:** Capture and manage customer profiles, addresses, preferences, and purchase history.
- **Endpoints:** `POST /customers`, `GET /customers/{id}`, `PATCH /customers/{id}`, `GET /customers/{id}/orders`
- **Port:** `8003`
- **Note:** All customer PII is stored encrypted in Supabase. See [Section 9](#9-customer-data-collection).

---

## 6. MCP Servers

All MCP servers live under `mcp/mcp-server/` — **one service, one folder**.

| MCP Tool | Description |
|---|---|
| `product_tools.py` | Fetch, search, and recommend products via AI |
| `customer_tools.py` | Look up customer profiles and order history |
| `auth_tools.py` | Validate sessions and permissions |

**Running the MCP server:**

```bash
cd mcp/mcp-server
pip install -r requirements.txt
python server.py
```

**MCP server port:** `8004`

---

## 7. Skills

Skills are task-specific prompting patterns used with Claude inside this project.

| Skill | Trigger Condition |
|---|---|
| `product-recommend` | User views homepage; suggest personalised products |
| `customer-capture` | New visitor detected; collect name, email, preferences |
| `support-assist` | User raises a query; provide formal, helpful response |
| `order-summary` | Post-purchase; summarise order details for the customer |

Skills are configured in `.claude/settings.json` and invoked via the MCP layer.

---

## 8. Style Rules

- **Tone:** Formal and professional at all times. Suitable for office-going users.
- **Naming:** `snake_case` for Python; `camelCase` for TypeScript variables; `PascalCase` for components and classes.
- **Comments:** All public functions must include a docstring (Python) or JSDoc comment (TypeScript).
- **Commits:** Use Conventional Commits format — `feat:`, `fix:`, `test:`, `docs:`, `chore:`.
- **Error responses:** Always return structured JSON: `{ "error": "...", "code": "...", "timestamp": "..." }`.
- **Linting:** `ruff` for Python; `eslint` + `prettier` for TypeScript.

---

## 9. Customer Data Collection

> **Policy:** All customer data is collected with explicit consent and stored securely.

### Data Collected

| Field | Purpose | Storage |
|---|---|---|
| Full Name | Personalisation | Supabase (encrypted) |
| Email Address | Authentication, notifications | Supabase Auth |
| Phone Number | Order updates (optional) | Supabase (encrypted) |
| Shipping Address | Delivery | Supabase |
| Purchase History | Recommendations | Supabase |
| Browsing Preferences | Product ranking | Supabase |

### Rules

1. No customer data is logged to stdout in production.
2. PII fields use Supabase Row-Level Security (RLS) policies — users may only access their own records.
3. A `customers` table migration must include `created_at`, `updated_at`, and `consent_given` columns.
4. Data deletion requests must be honoured within 30 days (GDPR/CCPA readiness).

---

## 10. Key Commands — Run

> Full documentation: `docs/run.md`

### Local Development (Recommended for POC)

```bash
# 1. Copy environment variables
cp .env.example .env
# Fill in your Supabase URL, anon key, and service role key.

# 2. Start all services via Docker Compose
docker compose -f infra/docker-compose.yml up --build

# 3. Or start services individually:
# Frontend
cd frontend && npm install && npm run dev          # http://localhost:3000

# Auth Service
cd services/auth-service && pip install -r requirements.txt
uvicorn app.main:app --reload --port 8001

# Product Service
cd services/product-service && pip install -r requirements.txt
uvicorn app.main:app --reload --port 8002

# Customer Service
cd services/customer-service && pip install -r requirements.txt
uvicorn app.main:app --reload --port 8003

# MCP Server
cd mcp/mcp-server && pip install -r requirements.txt
python server.py                                   # http://localhost:8004
```

### Using the Convenience Script

```bash
chmod +x scripts/run_all.sh
./scripts/run_all.sh
```

---

## 11. Key Commands — Test

> Full documentation: `docs/test.md`

### Python Services (Pytest)

```bash
# Run all Python unit tests
pytest services/ -v --cov=app --cov-report=term-missing

# Run tests for a specific service
pytest services/auth-service/tests/ -v
pytest services/product-service/tests/ -v
pytest services/customer-service/tests/ -v
```

### TypeScript Frontend (Vitest)

```bash
cd frontend
npm run test          # Run unit tests
npm run test:coverage # With coverage report
```

### All Tests at Once

```bash
chmod +x scripts/test_all.sh
./scripts/test_all.sh
```

### API Integration Tests (httpx)

```bash
pytest services/ -m integration -v
```

---

## 12. Load Balancing

Nginx distributes traffic across multiple replicas of each FastAPI service.

**`infra/nginx/nginx.conf` — upstream example:**

```nginx
upstream auth_service {
    least_conn;
    server auth-service-1:8001;
    server auth-service-2:8001;
}

upstream product_service {
    least_conn;
    server product-service-1:8002;
    server product-service-2:8002;
}

upstream customer_service {
    least_conn;
    server customer-service-1:8003;
    server customer-service-2:8003;
}
```

Scale replicas in Docker Compose:

```bash
docker compose up --scale auth-service=2 --scale product-service=2 --scale customer-service=2
```

> For POC, two replicas per service is sufficient. Scale further when traffic demands it.

---

## 13. Unit Testing Standards

Every module **must** have unit tests. Follow these standards:

| Requirement | Standard |
|---|---|
| Minimum coverage | **80%** per service |
| Test file naming | `test_<module_name>.py` / `<module>.test.ts` |
| Mocking | Use `pytest-mock` (Python) / `vi.mock` (Vitest) |
| Fixtures | Define reusable fixtures in `conftest.py` |
| Test categories | Mark with `@pytest.mark.unit` or `@pytest.mark.integration` |
| CI enforcement | GitHub Actions runs `test_all.sh` on every pull request |

### Example Python Unit Test Structure

```python
# services/auth-service/tests/test_auth_routes.py
import pytest
from httpx import AsyncClient
from app.main import app

@pytest.mark.unit
@pytest.mark.asyncio
async def test_register_user_success():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post("/auth/register", json={
            "email": "user@example.com",
            "password": "SecurePass123!"
        })
    assert response.status_code == 201
    assert "user" in response.json()
```

### Example TypeScript Unit Test Structure

```typescript
// frontend/src/components/__tests__/ProductCard.test.tsx
import { render, screen } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import ProductCard from '../ProductCard';

describe('ProductCard', () => {
  it('renders product name and price', () => {
    render(<ProductCard name="Laptop Pro" price={999.99} />);
    expect(screen.getByText('Laptop Pro')).toBeInTheDocument();
    expect(screen.getByText('$999.99')).toBeInTheDocument();
  });
});
```

---

*Last updated: 2026-05-31 | Maintained by: claude-workshop team*
