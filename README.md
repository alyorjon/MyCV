# 🤖 Collector Bot

> **Field Agent Monitoring Platform** — Real-time GPS tracking system for debt collection agents, built with a Telegram bot interface and a FastAPI dashboard backend.

[![Python](https://img.shields.io/badge/Python-3.11-blue?style=flat-square&logo=python)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.135-009688?style=flat-square&logo=fastapi)](https://fastapi.tiangolo.com)
[![aiogram](https://img.shields.io/badge/aiogram-3.27-2CA5E0?style=flat-square&logo=telegram)](https://aiogram.dev)
[![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy-2.0-red?style=flat-square)](https://sqlalchemy.org)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat-square&logo=docker)](https://docker.com)
[![Redis](https://img.shields.io/badge/Redis-7-DC382D?style=flat-square&logo=redis)](https://redis.io)

---

## 📋 Overview | Umumiy ko'rinish

**EN:** Collector Bot is a dual-service production platform designed for a trading company's debt collection department. Field agents (collectors) use a Telegram bot to register, share their live GPS location during working hours, and access a rich in-app dashboard via Telegram WebApp. The FastAPI backend serves as the API layer connecting the bot, PostgreSQL, Redis, and multiple Firebird branch databases.

**UZ:** Collector Bot — savdo kompaniyasining qarzlarni undirish bo'limi uchun mo'ljallangan ikki xizmatli production platforma. Dala agentlari (inkassatorlar) Telegram bot orqali ro'yxatdan o'tadilar, ish vaqtida jonli GPS lokatsiyasini ulashadilar va Telegram WebApp orqali boy interfeysdan foydalanadilar. FastAPI backend bot, PostgreSQL, Redis va bir nechta Firebird filial ma'lumotlar bazalarini bog'laydigan API qatlami vazifasini bajaradi.

---

## 🏗 Architecture | Arxitektura

```
┌─────────────────────────────────────────────────────────────┐
│                        TELEGRAM                              │
│   Field Agent  ◄──────────────────►  Bot (aiogram 3.x)      │
└──────────────────────────────────────────────────────────────┘
                                │
                      Webhook / Polling
                                │
                   ┌────────────▼───────────┐
                   │      Bot Service        │
                   │       :8001             │
                   │                         │
                   │  handlers/              │
                   │  middlewares/ (FSM)     │
                   │  states/                │
                   └────────────┬───────────┘
                                │
            ┌───────────────────┼──────────────────┐
            ▼                   ▼                  ▼
      [PostgreSQL]           [Redis]         [FastAPI API]
       :5432                 :6379            :8000
       users                 JWT tokens            │
       locations             session cache    ┌────┴────┐
       branches              live GPS TTL     ▼         ▼
                                        [PostgreSQL] [Firebird]
                                        locations    auth
                                        write        contracts
                                                     payments
```

### Services | Xizmatlar

| Service | Tech | Port | Role |
|---------|------|------|------|
| **Bot** | aiogram 3.x + aiohttp | 8001 | Telegram interface, FSM, GPS tracking |
| **API** | FastAPI + asyncio | 8000 | REST backend, auth, business logic |
| **PostgreSQL** | postgres:16 | 5432 | Users, locations, branches |
| **Redis** | redis:7 | 6379 | JWT tokens, session cache, GPS TTL |
| **Firebird** | fdb / sqlalchemy-firebird | 3050 | Legacy ERP: contracts, payments (N branches) |

---

## ✨ Features | Xususiyatlar

### 🤖 Bot Service
- **FSM multi-step registration** — full name → phone (contact button) → branch selection (inline keyboard)
- **8-hour live GPS tracking** — agents share live location; bot middleware validates session from Redis and writes GPS coords to PostgreSQL at configurable intervals (default: every 0.5 min)
- **Branch switching** — agents can change their active branch at any time
- **Telegram WebApp** — rich in-app dashboard accessible from the main menu button
- **Working hours gate** — `LocationCheckMiddleware` restricts non-location actions outside working hours
- **DbSessionMiddleware** — automatically opens/commits/rolls back PostgreSQL session for every update

### 🌐 API Backend
- **JWT Auth** with refresh token + Redis token store — login, refresh, logout
- **Firebird multi-branch engine registry** — dynamically connects to N Firebird databases (one per company branch), configured via `db.json`
- **Contracts endpoint** — list agent's contracts with optional search (FIO, contract number, phone, passport, PNFL)
- **Contract detail** — full detail: client info, payments, schedule, comments, dialogs, all contracts, products, gifts
- **Payment insertion** — post payment (cash/card) to a contract with optional photo comment
- **User statistics** — agent's performance metrics from Firebird
- **PostgreSQL CRUD** — branches list, user branch assignment, live location storage
- **Swagger UI** protected via HTTP Basic Auth
- **Health check** endpoint — `/health`

### 🔐 Auth & Security
- JWT access + refresh tokens with configurable TTL
- Token blacklisting on logout via Redis
- User data cached in Redis (reduces Firebird hits)
- Pydantic v2 input validation with SQL-injection protection on username/password fields
- CORS middleware configured

---

## 📁 Project Structure | Loyiha tuzilmasi

```
collector_bot/
├── bot/                        # Telegram bot service
│   ├── main.py                 # Bot entry point — webhook/polling setup
│   ├── handlers/
│   │   ├── registry.py         # /start, registration FSM, branch selection
│   │   ├── location.py         # Live GPS handler, location updates
│   │   ├── branches.py         # Branch switching flow
│   │   └── echo.py             # Fallback handler
│   ├── middlewares/
│   │   ├── db_middlewares.py   # PostgreSQL session per update
│   │   └── working_times.py    # Location gate during working hours
│   ├── keyboards/
│   │   └── reply.py            # Main menu, phone, branch inline, WebApp
│   ├── states/
│   │   └── forms.py            # FSM states: Reg, FullName, MallState
│   └── utils/
│       ├── check_user.py
│       ├── checking_branches.py
│       ├── checking_time.py
│       └── working_times.py
│
├── api/                        # FastAPI dashboard backend
│   ├── main.py                 # (entry via api/routers/main.py)
│   └── routers/
│       ├── main.py             # App factory, lifespan, CORS, Swagger auth
│       ├── auth.py             # /login /refresh /logout /users/me
│       ├── firebird.py         # Contracts, payments, comments, stats
│       └── pg.py               # Branches, user branch, location
│
├── db/
│   ├── postgres/
│   │   ├── session.py          # Async engine + sessionmaker
│   │   ├── models/
│   │   │   ├── tg_user.py      # User model
│   │   │   ├── branches.py     # Branch model
│   │   │   └── locations.py    # UserLocation model
│   │   └── repositories/
│   │       ├── user.py         # UserRepository
│   │       └── locations.py    # LocationRepository
│   └── firebird/
│       ├── registry.py         # Dynamic engine registry, get_fb_session()
│       ├── models/             # Firebird ORM models (SAT_USERS, SHARTNOMA, TULOV…)
│       └── repositories/       # ShartnomaRepository, PaymentsRepository
│
├── core/
│   ├── config.py               # Pydantic BaseSettings, FIREBIRD_DATABASES loader
│   ├── security.py             # bcrypt password hashing
│   └── redis.py                # RedisClient wrapper (get/set/delete/ttl)
│
├── schemas/                    # Pydantic v2 schemas
│   ├── auth.py
│   ├── user.py
│   ├── location.py
│   └── firebird/
│       └── comments.py
│
├── services/
│   └── auth.py                 # AuthService (login_or_register, register)
│
├── alembic/                    # DB migrations
│   ├── env.py
│   └── versions/
│
├── docs/                       # Project documentation
│   ├── architecture.md
│   ├── api.md
│   ├── deployment.md
│   └── firebird.md
│
├── bot_main.py                 # Bot entry point (python bot_main.py)
├── web_api.py                  # API entry point
├── docker_compose.yml
├── alembic.ini
├── db.json                     # Firebird branch config (gitignored in prod)
├── req.txt                     # Python dependencies
└── .env.example                # Environment template
```

---

## ⚙️ Installation & Setup | O'rnatish

### Prerequisites | Talablar

- Python 3.11+
- Docker & Docker Compose
- PostgreSQL 16
- Redis 7
- Firebird 2.5+ (for legacy ERP access)

### 1. Clone & environment

```bash
git clone https://github.com/alyorjon/collector_bot.git
cd collector_bot

cp .env.example .env
# .env faylini to'ldiring | Fill in .env
```

### 2. Configure `.env`

```env
# App
APP_ENV=dev                          # dev | prod
SECRET_KEY=your-super-secret-key
DEBUG=false

# Telegram
BOT_TOKEN=123456789:AAFxxxxxxxxxxxx
WEBHOOK_URL=https://yourdomain.com/api/v1/bot/webhook
WEBHOOK_SECRET=your-webhook-secret
WEB_APP_URL=https://your-webapp.com

# PostgreSQL
POSTGRES_DSN=postgresql+asyncpg://user:password@postgres:5432/collector_bot_db

# Redis
REDIS_URL=redis://redis:6379/0
REDIS_PASSWORD=your-redis-password

# Firebird (connection string)
fdb_connection=firebird+fdb://SYSDBA:masterkey@

# API Swagger Basic Auth
api_docs_login=admin
api_docs_password=your-swagger-password

# Tokens TTL (minutes)
ACCESS_TOKEN_EXPIRE_MINUTES=10080    # 7 days
REFRESH_TOKEN_EXPIRE_MINUTES=43200   # 30 days
LIVE_LOCATION_UPDATE_INTERVAL=0.5   # minutes
```

### 3. Configure `db.json` (Firebird branches)

```json
{
  "q": {
    "path": "192.168.1.100:3050//opt/firebird/data/branch_q.fdb",
    "name": "Qo'qon filiali"
  },
  "t": {
    "path": "192.168.1.101:3050//opt/firebird/data/branch_t.fdb",
    "name": "Toshkent filiali"
  }
}
```

> Each key (`"q"`, `"t"`) is the `branch` code agents enter at login. | Har bir kalit agentlar login vaqtida kiritadigan `branch` kodidir.

### 4. Run with Docker Compose

```bash
# Build and start all services | Barcha xizmatlarni ishga tushirish
docker compose -f docker_compose.yml up -d --build

# Run DB migrations | Migratsiyalarni qo'llash
docker compose exec api alembic upgrade head

# Check logs | Loglarni ko'rish
docker compose logs -f bot
docker compose logs -f api
```

### 5. Local development (without Docker)

```bash
pip install -r req.txt

# Terminal 1 — Bot
python bot_main.py

# Terminal 2 — API
uvicorn api.routers.main:app --host 0.0.0.0 --port 8000 --reload
```

---

## 🗄 Database | Ma'lumotlar bazalari

### PostgreSQL tables (env-suffixed)

| Table | Model | Description |
|-------|-------|-------------|
| `collector_bot_users_{env}` | `User` | Telegram users — telegram_id, phone, full_name, branch_code |
| `malls_{env}` | `Branch` | Company branches — code, name (uz/ru), address, map |
| `clients_locations_{env}` | `UserLocation` | Live GPS history — lat/lon, timestamp, user info |

> Tables are environment-aware: `_dev` in development, `_prod` in production (controlled by `APP_ENV`).

### Firebird tables (read-only, legacy ERP)

| Table | Description |
|-------|-------------|
| `SAT_USERS` | Employee auth |
| `SHARTNOMA` | Contracts |
| `TULOV` | Payments |
| `XARIDOR` | Clients |
| `GRAFIK` | Payment schedule |
| `OPERATORIZOHI` | Operator comments |
| `MULOQOT` | Client dialogs |
| `CHIQIM` / `MAXSULOT` | Products on contract |
| `BONUS` | Gift items |

### Redis keys

| Key pattern | TTL | Purpose |
|-------------|-----|---------|
| `auth:token:{token}` | 7 days | JWT token validity store |
| `auth:user:{branch}:{username}` | 7 days | User session cache (reduces Firebird hits) |
| `tg_user:checking:{telegram_id}` | 1 hour | Telegram user quick-check cache |
| `tg_user:live:{telegram_id}` | 8 hours | Last GPS write timestamp (controls write interval) |

---

## 📡 API Reference | API Endpointlar

Base URL: `http://localhost:8000`  
Swagger UI: `http://localhost:8000/myapi/docs` *(requires HTTP Basic Auth)*

### Auth

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/api/auth/login` | Login with Firebird credentials → JWT tokens |
| `POST` | `/api/v1/api/auth/refresh` | Refresh access token |
| `POST` | `/api/v1/api/auth/logout` | Invalidate token (remove from Redis) |
| `GET` | `/api/v1/api/auth/users/me` | Current user info |

**Login request:**
```json
{
  "username": "4445",
  "password": "2002",
  "branch": "q"
}
```

**Login response:**
```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "bearer"
}
```

### Firebird (requires Bearer token)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/firebird/databases` | Ping all Firebird connections |
| `GET` | `/api/v1/firebird/get-contracts-by-user-id` | Agent's contracts (with optional search) |
| `GET` | `/api/v1/firebird/get-contract-detail-by-id/{contract_id}/{client_id}` | Full contract detail |
| `GET` | `/api/v1/firebird/payments_all` | Agent's payments by date range |
| `POST` | `/api/v1/firebird/insert-comment-to-contract` | Add comment + optional photo to contract |
| `POST` | `/api/v1/firebird/insert-payment-to-contract` | Insert payment (cash/card) |
| `GET` | `/api/v1/firebird/get-user-statistics` | Agent performance stats |
| `GET` | `/api/v1/firebird/izoh/turlar` | Comment type enum list |

### PostgreSQL (requires Bearer token)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/pg/branches` | List active branches |
| `POST` | `/api/v1/pg/user/branch` | Assign branch to user |
| `POST` | `/api/v1/pg/location` | Save GPS coordinates |
| `GET` | `/api/v1/pg/me` | Current user's PostgreSQL record |

---

## 🔄 Bot Flow | Bot oqimi

### Agent Registration (FSM)

```
/start
  │
  ├─ [Redis cache hit] ──► main_menu ✅
  │
  └─ [New user]
        │
        ▼
   Reg.full_name  ← "Ismingizni kiriting"
        │
        ▼
   Reg.phone      ← "Telefon raqamingizni kiriting" + contact button
        │
        ▼
   Reg.mall       ← "Filial tanlang" (inline keyboard from db.json)
        │
        ▼
   DB updated → Redis cached → main_menu ✅
```

### Live GPS Tracking

```
Agent shares 8-hour live location
        │
        ▼
handle_location()
  live_period == 28800 (8h)? ──► Redis: tg_user:live:{id} = now (TTL: 8h)
        │
        ▼
        ... Telegram sends location updates ...
        │
        ▼
handle_live_location_update()  [edited_message]
  now - last_saved > LIVE_LOCATION_UPDATE_INTERVAL?
        │
        ├─ NO  ──► skip
        │
        └─ YES ──► LocationRepository.create_location() → PostgreSQL
                   Redis: tg_user:live:{id} = now (reset TTL)
```

### Main Menu buttons

| Button | Action |
|--------|--------|
| 🌐 Web-Interface | Opens Telegram WebApp at `WEB_APP_URL/{tg_id}/{branch}` |
| 🏢 Filial | Branch switching flow (MallState FSM) |
| 📍 Lokatsiya | Instructions for sharing live location |
| 📅 Shaxsiy reja | Personal plan (planned) |
| 👤 Profilim | Profile info (planned) |

---

## 🚀 Deployment | Deploy

### Docker Compose (production)

```bash
# First deploy | Birinchi deploy
docker compose -f docker_compose.yml up -d --build
docker compose exec api alembic upgrade head

# After code update | Kod yangilangandan keyin
git pull
docker compose -f docker_compose.yml up -d --build
docker compose exec api alembic upgrade head
```

### Nginx (reverse proxy)

```nginx
server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    # Telegram Webhook
    location /webhook {
        proxy_pass http://127.0.0.1:8001/webhook;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Dashboard API
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        client_max_body_size 10M;
    }
}

server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$host$request_uri;
}
```

### GitHub Actions CI/CD

The pipeline (`.github/workflows/deploy.yml`) triggers on push to `main`:
1. Checkout code
2. Sync files to server via `rsync` over SSH (excludes `.github/`)
3. Run `docker compose up -d --build` on server
4. Run `alembic upgrade head`

**Required GitHub Secrets:**

| Secret | Value |
|--------|-------|
| `HOST` | Server IP |
| `PORT` | SSH port (22) |
| `USERNAME` | SSH user |
| `PASSWORD` | SSH password |

### Alembic migrations

```bash
# Create migration | Migratsiya yaratish
alembic revision --autogenerate -m "add location index"

# Apply | Qo'llash
alembic upgrade head

# Rollback | Orqaga qaytish
alembic downgrade -1

# Status | Holat
alembic current
alembic history --verbose
```

---

## 🔒 Security | Xavfsizlik

- **JWT tokens** stored in Redis with configurable TTL — logout instantly invalidates tokens
- **Firebird credentials** never stored server-side — only validated at login, then cached encrypted in Redis
- **Pydantic validators** block SQL injection patterns (`--`, `/*`, spaces) in username/password
- **Swagger UI** gated behind HTTP Basic Auth — not publicly accessible
- **Non-root** Docker containers recommended for production
- **db.json** must be excluded from version control in production (add to `.gitignore`)

---

## 🧰 Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Bot framework | aiogram | 3.27 |
| Web framework | FastAPI | 0.135 |
| ASGI server | uvicorn | 0.43 |
| Async ORM | SQLAlchemy | 2.0 |
| DB migrations | Alembic | 1.18 |
| Data validation | Pydantic | 2.12 |
| Auth | python-jose + bcrypt | — |
| Cache / sessions | Redis (async) | 7 |
| Primary DB | PostgreSQL | 16 |
| Legacy DB | Firebird + fdb | 2.5 |
| Containerization | Docker Compose | v3.9 |
| CI/CD | GitHub Actions | — |

---

## 📄 Documentation | Hujjatlar

| File | Content |
|------|---------|
| [`docs/architecture.md`](docs/architecture.md) | System architecture, component diagrams, FSM flows |
| [`docs/api.md`](docs/api.md) | Full API reference with request/response examples |
| [`docs/deployment.md`](docs/deployment.md) | Docker, Nginx, CI/CD, Alembic guides |
| [`docs/firebird.md`](docs/firebird.md) | Firebird db.json config, models, async thread usage |

---

## 👤 Author | Muallif

**Alyorjon Aminboyev**  
Python Backend Developer · Middle

[![GitHub](https://img.shields.io/badge/GitHub-alyorjon-181717?style=flat-square&logo=github)](https://github.com/alyorjon)
[![Telegram](https://img.shields.io/badge/Telegram-alyor__dev-2CA5E0?style=flat-square&logo=telegram)](https://t.me/alyor_dev)

---

*Production system — "Munis Savdo", Fargona, Uzbekistan*
