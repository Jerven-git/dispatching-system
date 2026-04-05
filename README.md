# Dispatching System

A field service management platform for small service businesses (plumbing, electrical, HVAC, cleaning, pest control, etc.). Manage jobs, dispatch technicians, track field operations, invoice customers, and monitor business performance from one system.

## Features

- **Job Management** - Create, assign, track, and complete service jobs with full status workflow
- **Technician Dispatching** - Assign jobs, view workloads, GPS tracking, route optimization
- **Customer Portal** - Customers can request services, view jobs, track ETA, and access invoices
- **Invoicing** - Generate invoices from completed jobs, track payments, download PDFs
- **Inventory & Parts** - Parts catalog, stock tracking, low stock alerts, usage per job
- **Analytics & Reporting** - Revenue trends, technician performance, service profitability, CSV exports
- **Role-Based Access** - Admin, Dispatcher, and Technician dashboards with granular permissions
- **Recurring Jobs** - Automated job generation on daily/weekly/biweekly/monthly schedules
- **Digital Signatures** - Capture customer signatures on job completion
- **Audit Trail** - Complete change tracking across all entities

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | Laravel 12 (PHP 8.4) |
| Frontend | Next.js 16 (React 19, TypeScript) |
| Database | MySQL |
| Cache/Queue/Session | Redis |
| Web Server | Nginx |
| Containerization | Docker & Docker Compose |

## Quick Start

### Prerequisites

- Docker & Docker Compose
- Git (with SSH access to submodule repos)

### Setup

```bash
# Clone with submodules
git clone --recurse-submodules <repo-url>
cd dispatching-system

# Configure environment
cp .env.example .env
cp dispatching-backend/.env.example dispatching-backend/.env
cp dispatching-frontend/.env.example dispatching-frontend/.env

# Edit .env files with your credentials (see Environment Variables below)

# Build and start
docker compose build
docker compose up -d

# Set up backend
docker compose exec backend composer install
docker compose exec backend php artisan key:generate
docker compose exec backend php artisan migrate
docker compose exec backend php artisan db:seed
```

### Access

| Service | URL |
|---------|-----|
| Application | http://localhost:8000 |
| Frontend Dev Server | http://localhost:3000 |
| phpMyAdmin | http://localhost:8001 |
| Health Check | http://localhost:8000/api/health |

## Environment Variables

### Root `.env` (Docker Compose)

```env
DB_DATABASE=dispatching
DB_USERNAME=dispatching_user
DB_PASSWORD=your_secure_password
DB_ROOT_PASSWORD=your_secure_root_password
REDIS_PASSWORD=your_secure_redis_password
```

### Backend `.env` (key variables)

```env
APP_ENV=local
APP_DEBUG=true
DB_CONNECTION=mysql
DB_HOST=db
REDIS_HOST=redis
QUEUE_CONNECTION=redis
CACHE_STORE=redis
SESSION_DRIVER=redis
```

### Frontend `.env`

```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000
```

## Docker Services

| Service | Description |
|---------|-------------|
| `frontend` | Next.js dev server (port 3000) |
| `backend` | Laravel PHP-FPM (port 9000 internal) |
| `scheduler` | Laravel task scheduler |
| `queue` | Laravel queue worker (Redis-backed) |
| `nginx` | Reverse proxy (port 8000) |
| `redis` | Cache, sessions, and queues |
| `db` | MySQL database |
| `phpmyadmin` | DB admin UI (dev only, port 8001) |

## Project Structure

```
dispatching-system/
├── dispatching-backend/    # Laravel API (git submodule)
├── dispatching-frontend/   # Next.js UI (git submodule)
├── nginx/                  # Nginx configs (dev/staging/prod)
├── storage/                # Persistent data (gitignored)
├── docker-compose.yml      # Development environment
└── docker-compose-prod.yml # Production environment
```

## Common Commands

```bash
# Start/stop
docker compose up -d
docker compose down

# Backend
docker compose exec backend php artisan migrate
docker compose exec backend php artisan db:seed
docker compose exec backend php artisan tinker

# Logs
docker compose logs -f backend
docker compose logs -f queue
docker compose logs -f scheduler

# Rebuild after Dockerfile changes
docker compose build backend
docker compose up -d
```

## Default Users (after seeding)

Run `php artisan db:seed` to create sample data including admin, dispatcher, technician accounts, services, customers, and sample jobs. Check `database/seeders/DatabaseSeeder.php` for credentials.

## Production Deployment

```bash
# Use the production compose file
docker compose -f docker-compose-prod.yml build
docker compose -f docker-compose-prod.yml up -d

# Run migrations
docker compose -f docker-compose-prod.yml exec backend php artisan migrate --force
```

Key production checklist:
- Set `APP_ENV=production` and `APP_DEBUG=false` in backend `.env`
- Configure real mail driver (Mailgun, SES, etc.)
- Set up SSL certificates in `nginx/ssl/`
- Set strong passwords for DB and Redis
- Configure `SANCTUM_STATEFUL_DOMAINS` for your domain

## License

Proprietary. All rights reserved.
