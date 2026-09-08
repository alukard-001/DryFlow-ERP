<div align="center">

<img src="django_erp.png" width="22%" alt="Project Logo"/>

# DryFlow ERP

**Multi-tenant ERP & real-time logistics platform for building-materials distributors**

<em>Inventory, purchasing, delivery tracking, fleet and workforce management — in one API-first system.</em>

<p>
<img src="https://img.shields.io/badge/license-Apache%202.0-blue.svg" alt="license">
<img src="https://img.shields.io/badge/python-3.9%2B-3776AB.svg?logo=python&logoColor=white" alt="python">
<img src="https://img.shields.io/badge/Django-5.2-092E20.svg?logo=django&logoColor=white" alt="django">
<img src="https://img.shields.io/badge/DRF-3.16-A30000.svg?logo=django&logoColor=white" alt="drf">
<img src="https://img.shields.io/badge/Next.js-15-000000.svg?logo=next.js&logoColor=white" alt="nextjs">
<img src="https://img.shields.io/badge/Docker-ready-2496ED.svg?logo=docker&logoColor=white" alt="docker">
</p>

</div>

---

## Contents

- [About the project](#about-the-project)
- [Key features](#key-features)
- [Tech stack](#tech-stack)
- [Architecture](#architecture)
- [Project structure](#project-structure)
- [Getting started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Option A — Docker (recommended)](#option-a--docker-recommended)
  - [Option B — Manual setup](#option-b--manual-setup)
  - [Environment variables](#environment-variables)
- [API documentation](#api-documentation)
- [User roles & permissions](#user-roles--permissions)
- [Real-time features (WebSocket)](#real-time-features-websocket)
- [Background jobs](#background-jobs)
- [Testing](#testing)
- [CI/CD](#cicd)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## About the project

**DryFlow ERP** is a modular Enterprise Resource Planning system built for companies that sell, stock, and deliver building materials — originally designed around a **drywall (gypsum board) distribution business**, but generic enough to fit any warehouse-and-delivery operation.

It replaces spreadsheets and disconnected tools with a single source of truth for:

- what's in stock and where,
- what's on order from suppliers,
- what's on its way to a customer right now, and
- who did what, when.

The backend is a **Django REST Framework** API (JWT auth, WebSockets, Celery workers) and the frontend is a **Next.js / TypeScript** SPA consuming that API. Both live in this monorepo (`BackEnd/` and `FrontEnd/`) and can be run together via Docker Compose or independently.

## Key features

- 🏢 **Multi-tenant by design** — every core model is scoped to a `Companie` (organization) automatically, so one deployment can safely serve several independent businesses.
- 📦 **Full inventory lifecycle** — products, brands, categories, suppliers and warehouses, with **inflows**, **outflows**, **transfers**, **purchase orders** and **load orders**, each with an approval workflow (`pending → approved → completed / rejected / cancelled`) and automatic stock movement history.
- 🚚 **Real-time delivery tracking** — deliveries move through `pending → pickup_in_progress → in_transit → delivered` (or `returned` / `failed`), with GPS checkpoints pushed live to customers and managers over WebSockets.
- 🚐 **Fleet & driver management** — vehicles, assignments and driver-scoped delivery access.
- 👥 **HR basics** — employees, positions, and attendance tracking with access-code check-in.
- 🗓️ **Scheduling module** for planning installation/service jobs.
- 🔔 **Notification center** — in-app + WebSocket notifications, extensible per module.
- 🔐 **JWT authentication & RBAC** — 11 built-in roles (CEO, Owner, Admin, Manager, Employee, Installer, Stocker, Salesman, Driver, Customer, Supplier), enforced by permission groups on both API and UI.
- 📚 **Self-documenting API** — OpenAPI 3 schema via `drf-spectacular`, with Swagger and ReDoc UIs out of the box.
- ⚙️ **Event-driven core** — Django signals + service-layer validators keep business rules (stock availability, warehouse capacity, status transitions) consistent regardless of which endpoint triggered the change.

## Tech stack

| Layer | Technology |
|---|---|
| Backend framework | Django 5.2, Django REST Framework |
| Auth | `djangorestframework-simplejwt` (JWT), `django-axes` (brute-force protection) |
| Real-time | Django Channels (ASGI), Daphne, Redis (channel layer) |
| Background jobs | Celery + Celery Beat, Redis as broker/result backend |
| Database | PostgreSQL 15+ |
| API docs | drf-spectacular (OpenAPI 3, Swagger UI, ReDoc) |
| Frontend framework | Next.js 15 (App Router), React 19, TypeScript |
| Frontend styling | Tailwind CSS |
| Frontend forms | React Hook Form + Zod |
| HTTP client | Axios |
| Infra | Docker, Docker Compose, Nginx, Whitenoise |
| CI/CD | GitHub Actions, semantic-release |

## Architecture

```
                         ┌──────────────────────┐
                         │   FrontEnd (Next.js)  │
                         │  React 19 · TS · RHF  │
                         └───────────┬───────────┘
                                     │ REST + WebSocket
                                     ▼
        ┌───────────────────────────────────────────────────┐
        │                BackEnd (Django + DRF)               │
        │  ┌───────────┐ ┌────────────┐ ┌───────────────────┐│
        │  │ accounts  │ │ companies  │ │     inventory      ││
        │  │  (JWT,    │ │(customers, │ │ (product, brand,   ││
        │  │  profiles)│ │ employeers,│ │  supplier, wh.,    ││
        │  │           │ │ attendance)│ │  in/out/transfer,  ││
        │  └───────────┘ └────────────┘ │  purchase/load ord)││
        │  ┌───────────┐ ┌────────────┐ └───────────────────┘│
        │  │ delivery  │ │  vehicle   │ ┌───────────────────┐│
        │  │(tracking, │ │  (fleet)   │ │   scheduller /     ││
        │  │ checkpts) │ │            │ │   notifications    ││
        │  └───────────┘ └────────────┘ └───────────────────┘│
        │                    basemodels (multi-tenant base)   │
        └───────────────────┬───────────────────┬─────────────┘
                             │                   │
                     ┌───────▼───────┐   ┌───────▼───────┐
                     │  PostgreSQL   │   │     Redis     │
                     │   (data)      │   │ (channels,    │
                     │               │   │  celery, cache│
                     └───────────────┘   └───────┬───────┘
                                                  │
                                          ┌───────▼───────┐
                                          │ Celery worker │
                                          │  + Beat       │
                                          └───────────────┘
```

Every business model inherits from a shared `BaseModel` (`basemodels` app) that automatically stamps `id (UUID)`, `companie`, `created_by`/`updated_by` and timestamps — this is what makes multi-tenancy and audit history consistent across all modules.

## Project structure

```
DryFlow-ERP/
├── BackEnd/
│   ├── core/               # settings, ASGI/WSGI, Celery app, URL root
│   ├── api/                # top-level API router, JWT & schema endpoints
│   ├── basemodels/         # shared multi-tenant BaseModel
│   └── apps/
│       ├── accounts/       # auth, users, profiles
│       ├── companies/      # companies, customers, employeers, attendance
│       ├── inventory/      # product, brand, category, supplier, warehouse,
│       │                   # inflows, outflows, transfer, purchase_order, load_order, movements
│       ├── delivery/       # real-time delivery tracking (REST + WebSocket)
│       ├── vehicle/        # fleet management
│       ├── scheduller/     # job/visit scheduling
│       └── notifications/  # in-app + WebSocket notifications
├── FrontEnd/
│   └── src/
│       ├── app/            # Next.js App Router pages (auth, dashboard, products, suppliers...)
│       ├── components/     # UI, layout, auth, modals
│       ├── contexts/       # AuthContext
│       ├── services/       # API clients (auth, product, supplier, brand, user)
│       ├── hooks/, lib/, types/
├── Plans/                  # module enhancement & roadmap docs
├── SysRequirements/        # original system requirements & architecture spec
└── docker-compose.yml
```

## Getting started

### Prerequisites

**Backend**
- Python 3.9+
- PostgreSQL 15+
- Redis 7+
- Docker & Docker Compose (recommended)

**Frontend**
- Node.js 18+
- npm or yarn

### Option A — Docker (recommended)

This spins up the API (Daphne/ASGI), PostgreSQL, Redis, Celery worker, Celery Beat and Nginx.

```bash
git clone https://github.com/<your-org>/DryFlow-ERP.git
cd DryFlow-ERP/BackEnd

cp .env.example .env
# edit .env with your own secret key, DB credentials, etc.

docker compose up --build
```

The stack applies migrations, collects static files and sets up permission groups automatically on startup. The API becomes available behind Nginx at `http://localhost/` (raw app container on port `8000`).

### Option B — Manual setup

**Backend:**

```bash
cd BackEnd
python -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate

pip install -r requirements.txt
cp .env.example .env          # point DATABASE_HOST/REDIS_URL at localhost, etc.

python manage.py migrate
python manage.py setup_permission_groups
python manage.py createsuperuser
python manage.py runserver
```

WebSocket features (delivery tracking, notifications) require an ASGI server — for local development you can instead run:

```bash
daphne -b 0.0.0.0 -p 8000 core.asgi:application
```

Celery (optional locally, required for async emails/reports/notifications):

```bash
celery -A core worker -l INFO
celery -A core beat -l INFO
```

**Frontend:**

```bash
cd FrontEnd
npm install
npm run dev
```

The frontend runs at `http://localhost:3000` and the backend at `http://localhost:8000` (or `http://localhost` behind Nginx).

### Environment variables

Configured via `BackEnd/.env` (see `.env.example` for the full list):

| Variable | Purpose |
|---|---|
| `DJANGO_SECRET_KEY`, `DJANGO_DEBUG`, `DJANGO_ALLOWED_HOSTS` | Core Django config |
| `DATABASE_NAME/USER/PASSWORD/HOST/PORT` | PostgreSQL connection |
| `REDIS_URL`, `CELERY_BROKER_URL`, `CELERY_RESULT_BACKEND` | Redis / Celery |
| `CORS_ALLOWED_ORIGINS`, `CSRF_TRUSTED_ORIGINS` | Cross-origin & CSRF for the frontend domain |
| `EMAIL_*` | Optional SMTP settings for transactional email |
| `AWS_*` | Optional S3 storage for media |
| `STRIPE_KEY` | Reserved for future billing integration (not active yet) |

## API documentation

Once the backend is running, the full OpenAPI schema and interactive docs are available at:

- Swagger UI — `/api/v1/swagger/`
- ReDoc — `/api/v1/redoc/`
- Raw schema — `/api/v1/schema/`
- Django admin — `/admin/`

Main resource groups under `/api/v1/`: `user/`, `profiles/`, `companies/`, `customers/`, `employeers/`, `attendance/`, `delivery/`, `vehicle/`, `suppliers/`, `brands/`, `products/`, `warehouse/`, `transfers/`, `inflows/`, `outflows/`, `movements/`, `load-orders/`, `purchase-orders/`, `scheduller/`, `notifications/`.

## User roles & permissions

Built-in `user_type` roles, enforced through permission groups and used to shape UI access on the frontend:

`CEO` · `Owner` · `Admin` · `Manager` · `Employee` · `Installer` · `Stocker` · `Salesman` · `Driver` · `Customer` · `Supplier`

Examples: drivers can only see and update their own deliveries; customers only see deliveries addressed to them; managers get full company-wide visibility.

## Real-time features (WebSocket)

Built with Django Channels on top of Redis:

- **Delivery tracking** — live GPS/status updates pushed to a `delivery_id`-scoped channel group as a shipment moves through its lifecycle.
- **Notifications** — instant in-app alerts (low stock, order approvals, delivery status, etc.).

## Background jobs

Celery + Celery Beat handle asynchronous and scheduled work such as notification dispatch, periodic reports and stock/threshold checks, using Redis as the broker and result backend.

## Testing

```bash
cd BackEnd
python manage.py test
```

The CI pipeline runs the same suite against real PostgreSQL and Redis services on every push/PR to `main` and `Dev`.

## CI/CD

- **GitHub Actions** (`django.yml`) — runs the Django test suite against PostgreSQL and Redis service containers on each push/PR.
- **semantic-release** (`release.yml`, `release.config.js`) — automates versioning and changelog generation from conventional commits (see `CHANGELOG.md`).

## Roadmap

Ongoing and planned work is tracked in [`Plans/`](./Plans):

- [`EXECUTIVE_SUMMARY_ANALYSIS.md`](./Plans/EXECUTIVE_SUMMARY_ANALYSIS.md) — current state assessment
- [`EXISTING_MODULES_ENHANCEMENT_PLAN.md`](./Plans/EXISTING_MODULES_ENHANCEMENT_PLAN.md) — improvements to shipped modules
- [`PARTIAL_MODULES_ENHANCEMENT_PLAN.md`](./Plans/PARTIAL_MODULES_ENHANCEMENT_PLAN.md) — modules that are partially implemented
- [`MISSING_MODULES_PLAN.md`](./Plans/MISSING_MODULES_PLAN.md) — modules not yet started (e.g. billing/Stripe, finance/accounting)

Original domain requirements and DB/architecture spec live in [`SysRequirements/`](./SysRequirements).

## Contributing

1. Fork the repo and create a feature branch: `git checkout -b feature/my-feature`
2. Follow the existing service-layer pattern (validators + handlers) when touching business logic.
3. Add/update tests for anything you change.
4. Use [Conventional Commits](https://www.conventionalcommits.org/) — the changelog and versioning are generated from them.
5. Open a pull request against `Dev`.

## License

Distributed under the **Apache License 2.0** — see [`LICENSE`](./LICENSE) for the full text.
