# turborepo monorepo with Next.js 15 + NestJS 11 + shadcn


## 🏗️ Architecture Overview (Lender-Marketplace)

```
superepo/
├─ apps/
│  ├─ web/     ← Next.js 15 dashboard (Broker • Lender • Admin)
│  └─ api/     ← NestJS 11 HTTP + WebSocket API
├─ packages/
│  ├─ ui/      ← shared shadcn-based components
│  └─ shared/  ← TypeScript DTOs, validators, utils
└─ docker-compose.yml  ← Postgres • Redis • (future) minio
```

### 1. Frontend – **Next.js 15**

* **App Router** with role-based route groups (`/broker`, `/lender`, `/admin`)
* TailwindCSS + shadcn/ui; RSC + Server Actions call the API directly
* Auth handled by **NextAuth** (credentials provider) – stores JWT returned by the API
* Socket.IO client for real-time deal & chat updates

### 2. Backend – **NestJS 11**

| Layer             | Details                                                               |
| ----------------- | --------------------------------------------------------------------- |
| **HTTP & WS**     | REST controllers (`/deals`, `/quotes`), WebSocket gateway (`/chat`)   |
| **Prisma ORM**    | PostgreSQL models: `User`, `Deal`, `Quote`, `Message`, `Wallet`       |
| **Auth module**   | JWT (access + refresh) with optional TOTP (speakeasy)                 |
| **Queues**        | **BullMQ** on Redis – tasks: lender-matching, PDF export, email/Slack |
| **External APIs** | iwoca OpenLending, future Plaid/TrueLayer for open-banking            |
| **Docs**          | Swagger at `/api/docs` (auto-generated from decorators)               |

### 3. Infrastructure

* **Turborepo** – fast incremental builds; single `pnpm dev` spins up web + api
* **Docker Compose** – local Postgres 16 (`5432`) & Redis 7 (`6379`)
* **ENV management** – per-app `.env` files; secrets never committed
* **CI/CD** – GitHub Actions → Turbo cache → Vercel (web) & Fly.io / Render (api)

### 4. High-level Data Flow

```mermaid
sequenceDiagram
    Broker UI->>API: POST /deals
    API->>Prisma/Postgres: Persist Deal
    API--)BullMQ(matcher): enqueue({dealId})
    BullMQ-->Matcher Worker: Job picked
    Matcher Worker-->>Lender APIs: GET /quote
    Lender APIs-->>Matcher Worker: APR, term, fees
    Matcher Worker->>Postgres: save Quote(s)
    Matcher Worker-->>Socket.IO: emit "quote-ready"
    Socket.IO-->>Broker UI: real-time notification
```

### 5. Why This Design?

* **Single-repo DX:** shared types prevent contract drift; atomic PRs cover UI + API.
* **Prisma + Postgres:** strong relations, migrations, type-safe client.
* **BullMQ:** isolates CPU- or I/O-heavy tasks; keeps API responsive.
* **Socket.IO:** low-latency chat & status updates without polling.
* **Turborepo:** minimal config yet fast builds; Nx can be layered later if needed.


