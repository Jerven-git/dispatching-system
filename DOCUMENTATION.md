# Dispatching System - Technical Documentation

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Tech Stack](#2-tech-stack)
3. [Architecture](#3-architecture)
4. [Folder & File Structure](#4-folder--file-structure)
5. [Backend Documentation](#5-backend-documentation)
6. [Frontend Documentation](#6-frontend-documentation)
7. [Setup Guide](#7-setup-guide)
8. [Usage Guide](#8-usage-guide)
9. [Code Review](#9-code-review)
10. [Security Review](#10-security-review)
11. [Scalability](#11-scalability)

---

## 1. Project Overview

### Purpose

The Dispatching System is a **B2B field service management platform** designed to help service companies manage job dispatching, technician assignments, customer relations, invoicing, and field operations from a single unified interface.

### Problem It Solves

Service companies (e.g., HVAC, plumbing, electrical) face challenges coordinating technicians, tracking job status, managing inventory, billing customers, and providing visibility to stakeholders. This system centralizes those operations into a web application with role-based dashboards.

### Target Users

| Role | Description |
|------|-------------|
| **Admin** | Company owner/manager with full system control |
| **Dispatcher** | Operations staff who create and assign jobs |
| **Technician** | Field workers who execute and update job status |
| **Customer** | End customers who access a self-service portal |

### Core Features

- **Job Management** - Create, assign, track, and complete service jobs with full status workflow
- **Technician Dispatching** - Assign technicians, view workloads, optimize routes
- **Customer Portal** - Self-service for customers to request services, view jobs, and pay invoices
- **Invoicing & Parts** - Generate invoices, track inventory, manage parts used on jobs
- **GPS & ETA Tracking** - Real-time technician location and estimated arrival times
- **Analytics & Reporting** - Revenue trends, technician performance, service profitability, scheduled reports
- **Multi-Tenancy** - Isolated tenant data with plan-based feature gating
- **Role-Based Access Control** - System roles + custom roles with 40+ granular permissions
- **Recurring Jobs** - Automated job generation on daily/weekly/biweekly/monthly schedules
- **Offline Sync** - Mobile technician support with offline data synchronization
- **Audit Trail** - Complete change tracking across all entities
- **Digital Signatures** - Capture customer signatures on completed jobs

---

## 2. Tech Stack

### Backend

| Component | Technology |
|-----------|------------|
| Language | PHP 8.4 |
| Framework | Laravel 12.x |
| Auth | Laravel Sanctum (token-based API auth) |
| ORM | Eloquent |
| Task Scheduling | Laravel Scheduler (via `schedule:work`) |
| Notifications | Laravel Notifications (database + mail) |
| PDF Generation | Supported via GD library |

### Frontend

| Component | Technology |
|-----------|------------|
| Framework | Next.js 16.2.1 |
| Language | TypeScript 5.9.3 |
| UI Library | React 19.2.4 |
| Styling | Tailwind CSS 4.2.2 |
| Component Variants | class-variance-authority (CVA) |
| Icons | Lucide React |
| UI Primitives | Radix UI (dialog, dropdown, select) |
| State Management | React Context API (AuthContext) |
| Runtime (dev) | Bun 1.3.6 |

### Database

| Component | Technology |
|-----------|------------|
| DBMS | MySQL (latest) |
| Admin Tool | phpMyAdmin 5 (development only) |

### Infrastructure

| Component | Technology |
|-----------|------------|
| Containerization | Docker + Docker Compose |
| Web Server | Nginx (Alpine) |
| PHP Runtime | PHP-FPM 8.4 |
| SSL/TLS | Configurable (production config included) |
| Environments | Development, Staging, Production |

---

## 3. Architecture

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────┐
│                       Client Browser                      │
└──────────────┬────────────────────────────┬───────────────┘
               │                            │
               │  Port 3000 (dev)           │  Port 8000 (dev)
               │  Port 443 (prod)           │  Port 443 (prod)
               ▼                            ▼
┌──────────────────────┐    ┌──────────────────────────────┐
│   Next.js Frontend   │    │         Nginx Reverse        │
│   (React/TypeScript) │    │         Proxy Server         │
│                      │    │                              │
│  - Admin Dashboard   │    │  /api/* → Laravel (PHP-FPM)  │
│  - Dispatcher View   │    │  /storage/* → Static Files   │
│  - Technician App    │    │  /* → Next.js Frontend       │
│  - Customer Portal   │    │                              │
└──────────┬───────────┘    └──────────┬───────────────────┘
           │                           │
           │  HTTP/JSON API            │  FastCGI :9000
           │                           ▼
           │               ┌───────────────────────┐
           └──────────────►│   Laravel Backend     │
                           │   (PHP 8.4 + FPM)     │
                           │                       │
                           │  - REST API           │
                           │  - Auth (Sanctum)     │
                           │  - Business Logic     │
                           │  - Notifications      │
                           │  - Scheduled Tasks    │
                           └───────────┬───────────┘
                                       │
                                       │  MySQL Protocol
                                       ▼
                           ┌───────────────────────┐
                           │     MySQL Database     │
                           │                       │
                           │  21 tables            │
                           │  Multi-tenant scoped  │
                           └───────────────────────┘
```

### Request Flow

1. **Browser** sends request to Nginx
2. **Nginx** routes `/api/*` requests to Laravel via FastCGI, all other paths to Next.js
3. **Laravel** authenticates via Sanctum, resolves tenant, checks role/permissions
4. **Controller** delegates to **Service** classes for business logic
5. **Service** interacts with **Eloquent Models** (auto-scoped by tenant)
6. **API Resource** transforms the response to JSON
7. Response flows back through Nginx to the browser

### Key Design Patterns

| Pattern | Usage |
|---------|-------|
| **Repository-like Service Layer** | Business logic isolated in 11 service classes (not raw Eloquent in controllers) |
| **Global Scopes (Multi-Tenancy)** | `TenantScope` auto-filters all queries by `tenant_id` |
| **Trait Composition** | `BelongsToTenant` and `Auditable` traits mixed into models |
| **Event-Driven** | `JobStatusChanged` and `TechnicianAssigned` events trigger notifications |
| **Form Request Validation** | 13 dedicated request classes handle input validation |
| **API Resources** | 8 resource classes standardize JSON output |
| **Status Machine** | `StatusTransitionService` enforces valid job status transitions |
| **Singleton API Client** | Frontend `ApiClient` with caching and request deduplication |
| **Context-Based Auth** | React `AuthContext` provides global auth state |
| **Role Guard Pattern** | `RoleGuard` component enforces role-based UI rendering |

---

## 4. Folder & File Structure

### Root Level

```
dispatching-system/
├── .env                        # Docker Compose env vars (DB credentials)
├── .env.example                # Template for .env
├── .gitmodules                 # Git submodule definitions
├── docker-compose.yml          # Development environment (6 services)
├── docker-compose-prod.yml     # Production environment (3 services)
├── nginx/
│   ├── conf/
│   │   └── nginx.conf          # Global Nginx config (gzip, workers)
│   └── conf.d/
│       ├── default.conf        # Dev: HTTP proxy rules
│       ├── production.conf     # Prod: HTTPS with SSL/TLS
│       └── staging.conf        # Staging: HTTP without SSL
├── storage/
│   └── db-data/                # MySQL data volume (gitignored)
├── dispatching-backend/        # Git submodule → Laravel API
└── dispatching-frontend/       # Git submodule → Next.js UI
```

### Backend Structure

```
dispatching-backend/
├── app/
│   ├── Console/Commands/           # 2 scheduled commands
│   │   ├── GenerateRecurringJobs.php
│   │   └── SendScheduledReports.php
│   ├── Events/                     # 2 domain events
│   │   ├── JobStatusChanged.php
│   │   └── TechnicianAssigned.php
│   ├── Http/
│   │   ├── Controllers/Api/        # 19 API controllers
│   │   │   ├── AuthController.php
│   │   │   ├── DashboardController.php
│   │   │   ├── ServiceJobController.php
│   │   │   ├── CustomerController.php
│   │   │   ├── UserController.php
│   │   │   ├── InvoiceController.php
│   │   │   ├── ServiceController.php
│   │   │   ├── PartController.php
│   │   │   ├── JobPartController.php
│   │   │   ├── JobEnhancementController.php
│   │   │   ├── ServiceRequestController.php
│   │   │   ├── TechnicianLocationController.php
│   │   │   ├── RouteController.php
│   │   │   ├── SyncController.php
│   │   │   ├── ReportController.php
│   │   │   ├── AnalyticsController.php
│   │   │   ├── ExportController.php
│   │   │   ├── ScheduledReportController.php
│   │   │   ├── TenantController.php
│   │   │   ├── RoleController.php
│   │   │   ├── AuditLogController.php
│   │   │   ├── NotificationController.php
│   │   │   └── Portal/
│   │   │       ├── CustomerAuthController.php
│   │   │       └── CustomerPortalController.php
│   │   ├── Middleware/             # 4 middleware classes
│   │   │   ├── EnsureRole.php
│   │   │   ├── CheckPermission.php
│   │   │   ├── ResolveTenant.php
│   │   │   └── TenantRateLimiter.php
│   │   ├── Requests/              # 13 form request classes
│   │   └── Resources/             # 8 API resource classes
│   ├── Models/                    # 20 Eloquent models
│   │   ├── Scopes/
│   │   │   └── TenantScope.php
│   │   └── Traits/
│   │       ├── BelongsToTenant.php
│   │       └── Auditable.php
│   ├── Notifications/             # 4 notification classes
│   ├── Providers/                 # Service providers
│   └── Services/                  # 11 service classes
├── config/                        # Laravel configuration
├── database/
│   ├── factories/                 # Model factories
│   ├── migrations/                # 21 migration files
│   └── seeders/                   # DatabaseSeeder, PermissionSeeder
├── routes/
│   ├── api.php                    # All API route definitions
│   └── console.php                # Scheduled task registration
├── Dockerfile                     # PHP 8.4-FPM image
├── composer.json                  # PHP dependencies
└── .env.example                   # Backend env template
```

### Frontend Structure

```
dispatching-frontend/
├── src/
│   ├── app/
│   │   ├── (protected)/           # Auth-required routes
│   │   │   ├── admin/             # 13 admin page groups
│   │   │   │   ├── page.tsx                    # Dashboard
│   │   │   │   ├── users/                      # User CRUD
│   │   │   │   ├── customers/                  # Customer CRUD
│   │   │   │   ├── jobs/                       # Job CRUD
│   │   │   │   ├── services/                   # Service CRUD
│   │   │   │   ├── parts/                      # Parts CRUD
│   │   │   │   ├── invoices/                   # Invoice management
│   │   │   │   ├── tenants/                    # Tenant management
│   │   │   │   ├── roles/                      # Role & permissions
│   │   │   │   ├── analytics/                  # Business analytics
│   │   │   │   ├── reports/                    # Reports
│   │   │   │   ├── calendar/                   # Calendar view
│   │   │   │   ├── audit-logs/                 # Audit trail
│   │   │   │   └── layout.tsx                  # Admin layout
│   │   │   ├── dispatcher/        # 4 dispatcher page groups
│   │   │   │   ├── page.tsx                    # Dashboard
│   │   │   │   ├── jobs/                       # Job dispatch
│   │   │   │   ├── customers/                  # Customer view
│   │   │   │   ├── calendar/                   # Calendar
│   │   │   │   ├── reports/                    # Reports
│   │   │   │   └── layout.tsx                  # Dispatcher layout
│   │   │   ├── technician/        # 2 technician page groups
│   │   │   │   ├── page.tsx                    # Dashboard
│   │   │   │   ├── my-jobs/                    # My assigned jobs
│   │   │   │   └── layout.tsx                  # Technician layout
│   │   │   └── layout.tsx         # Protected route wrapper
│   │   ├── login/page.tsx         # Login page
│   │   ├── layout.tsx             # Root layout (AuthProvider)
│   │   ├── page.tsx               # Home (role-based redirect)
│   │   └── globals.css            # Global styles & CSS vars
│   ├── components/
│   │   ├── ui/                    # 13 reusable UI components
│   │   ├── layout/                # Sidebar, TopBar, NotificationBell
│   │   ├── auth/                  # RoleGuard
│   │   ├── dashboard/             # StatsGrid
│   │   ├── jobs/                  # 10 job-related components
│   │   ├── customers/             # 3 customer components
│   │   ├── services/              # 3 service components
│   │   ├── users/                 # 3 user components
│   │   ├── parts/                 # 4 parts/inventory components
│   │   ├── invoices/              # 2 invoice components
│   │   ├── calendar/              # CalendarView
│   │   ├── reports/               # 5 report components
│   │   └── ErrorBoundary.tsx      # Error boundary
│   ├── contexts/
│   │   └── AuthContext.tsx        # Authentication state
│   ├── hooks/
│   │   └── useFormSubmit.ts       # Form submission hook
│   ├── lib/
│   │   ├── api.ts                 # API client (caching, CSRF)
│   │   ├── roles.ts              # Role utilities
│   │   ├── job-constants.ts      # Status/priority constants
│   │   └── utils.ts              # Tailwind class merger
│   └── types/
│       └── index.ts               # TypeScript type definitions
├── package.json                   # Node dependencies
├── tailwind.config.ts             # Tailwind theme config
├── tsconfig.json                  # TypeScript config
├── next.config.ts                 # Next.js config
├── Dockerfile                     # Bun runtime image
└── .env.example                   # Frontend env template
```

---

## 5. Backend Documentation

### 5.1 Models & Relationships

#### Entity Relationship Overview

```
Tenant ──┬── Users ──── TechnicianLocations
         ├── Customers ──┬── ServiceJobs ──┬── JobAttachments
         ├── Roles       │                 ├── JobChecklistEntries
         ├── AuditLogs   │                 ├── JobComments
         │               │                 ├── JobStatusLogs
         │               │                 ├── JobParts ── Parts
         │               │                 ├── JobReviews
         │               │                 └── Invoices
         │               └── ServiceRequests
         └── Services ── ChecklistItems
```

#### Core Models

**User** (`users`)
- Fields: `name`, `email`, `phone`, `password`, `role` (admin/dispatcher/technician), `is_active`, `tenant_id`, `custom_role_id`
- Relations: `assignedJobs()`, `createdJobs()`, `tenant()`, `customRole()`, `locations()`, `latestLocation()`
- Methods: `hasPermission()`, `isAdmin()`, `isDispatcher()`, `isTechnician()`

**Customer** (`customers`)
- Fields: `name`, `email`, `phone`, `address`, `city`, `state`, `zip_code`, `notes`, `password`, `portal_access`
- Relations: `serviceJobs()`, `reviews()`, `serviceRequests()`, `invoices()`

**Service** (`services`)
- Fields: `name`, `description`, `base_price`, `estimated_duration_minutes`, `is_active`
- Relations: `serviceJobs()`, `checklistItems()`

**ServiceJob** (`service_jobs`)
- Fields: `reference_number` (auto: JOB-YYYY-NNNNN), `customer_id`, `service_id`, `technician_id`, `created_by`, `status`, `priority`, `description`, `address`, `scheduled_date`, `scheduled_time`, `started_at`, `completed_at`, `cancelled_at`, `technician_notes`, `total_cost`, `recurring_frequency`, `recurring_end_date`, `parent_job_id`, `signature_path`, `signed_by_name`, `signed_at`, `latitude`, `longitude`
- Relations: `customer()`, `service()`, `technician()`, `creator()`, `statusLogs()`, `attachments()`, `checklistEntries()`, `comments()`, `invoices()`, `parts()`, `parentJob()`, `childJobs()`

**Invoice** (`invoices`)
- Fields: `invoice_number` (auto: INV-YYYY-NNNNN), `service_job_id`, `customer_id`, `created_by`, `subtotal`, `tax_rate`, `tax_amount`, `total`, `status` (draft/sent/paid/overdue/cancelled), `notes`, `issued_date`, `due_date`, `paid_at`

**Part** (`parts`)
- Fields: `name`, `description`, `sku`, `unit_price`, `stock_quantity`, `minimum_stock`, `unit`, `is_active`
- Method: `isLowStock()` - returns `true` if `stock_quantity <= minimum_stock`

**Tenant** (`tenants`)
- Fields: `name`, `slug`, `domain`, `plan` (free/basic/pro/enterprise), `max_users`, `settings` (JSON), `is_active`
- Method: `canAddUsers()` - checks against `max_users` limit

#### Supporting Models

| Model | Purpose |
|-------|---------|
| `JobAttachment` | File uploads (before/after photos, documents) |
| `ChecklistItem` | Service-specific checklist templates |
| `JobChecklistEntry` | Checklist completion per job |
| `JobComment` | Internal/external discussion threads |
| `JobStatusLog` | Status change audit trail |
| `JobReview` | Customer ratings (1-5 stars) |
| `JobPart` | Parts consumed on a job (snapshot pricing) |
| `ServiceRequest` | Customer portal service requests |
| `TechnicianLocation` | GPS tracking points |
| `ScheduledReport` | Automated report scheduling |
| `Role` | Custom roles per tenant |
| `Permission` | Granular permission definitions |
| `AuditLog` | Polymorphic change tracking |

#### Model Traits

- **BelongsToTenant** - Auto-applies `TenantScope` (filters by `tenant_id`), auto-sets `tenant_id` on creation
- **Auditable** - Listens to `created`, `updated`, `deleted` events and creates `AuditLog` entries with old/new values

### 5.2 Services (Business Logic)

| Service | Responsibilities |
|---------|-----------------|
| **JobAssignmentService** | Assign technicians to jobs, get technician workload stats |
| **StatusTransitionService** | Define & validate allowed status transitions |
| **StatusLogService** | Log status changes, update job timestamps |
| **LocationTrackingService** | Record GPS, get latest/all locations, Haversine distance, purge old data |
| **ETAService** | Calculate ETA using Haversine + road factor (1.4x) at 40 km/h average |
| **InventoryService** | Add/remove parts to jobs, adjust stock, recalculate job totals, low stock alerts |
| **RouteOptimizationService** | Nearest-neighbor algorithm for technician route ordering |
| **AnalyticsService** | Revenue/job trends, service popularity, customer LTV, profitability |
| **ReportService** | Summary stats, jobs by status/date, technician performance |
| **ExportService** | CSV exports for jobs, invoices, customers, technician performance |

### 5.3 Job Status Workflow

```
              ┌──────────┐
              │ pending   │
              └─────┬─────┘
           ┌────────┼────────┐
           ▼        │        ▼
      ┌─────────┐   │   ┌──────────┐
      │cancelled│   │   │ assigned │
      └─────────┘   │   └────┬─────┘
                    │   ┌────┼────────┐
                    │   ▼    │        ▼
                    │ ┌──────────┐  ┌──────────┐
                    │ │on_the_way│  │ pending  │
                    │ └────┬─────┘  └──────────┘
                    │      ▼
                    │ ┌───────────┐
                    └►│in_progress│
                      └─────┬─────┘
                     ┌──────┼──────┐
                     ▼      │      ▼
               ┌──────────┐ │ ┌──────────┐
               │cancelled │ │ │completed │
               └──────────┘ │ └──────────┘
                            ▼
                       ┌─────────┐
                       │cancelled│
                       └─────────┘
```

### 5.4 API Endpoints

#### Authentication

| Method | Route | Description |
|--------|-------|-------------|
| POST | `/api/login` | User login (throttled: 5/min) |
| POST | `/api/logout` | User logout |
| GET | `/api/me` | Get current user |
| POST | `/api/portal/login` | Customer portal login |

#### Service Jobs

| Method | Route | Description | Roles |
|--------|-------|-------------|-------|
| GET | `/api/service-jobs` | List jobs (paginated, filterable) | Admin, Dispatcher |
| POST | `/api/service-jobs` | Create job | Admin, Dispatcher |
| GET | `/api/service-jobs/{id}` | Get job details | Admin, Dispatcher |
| PUT | `/api/service-jobs/{id}` | Update job | Admin, Dispatcher |
| DELETE | `/api/service-jobs/{id}` | Delete job | Admin only |
| PATCH | `/api/service-jobs/{id}/status` | Update status | Admin, Dispatcher |
| PATCH | `/api/service-jobs/{id}/assign` | Assign technician | Admin, Dispatcher |
| POST | `/api/service-jobs/{id}/clone` | Clone job | Admin, Dispatcher |
| GET | `/api/service-jobs/calendar` | Calendar view data | Admin, Dispatcher |
| GET | `/api/my-jobs` | Technician's jobs | Technician |
| GET | `/api/my-jobs/{id}` | Technician job detail | Technician |
| PATCH | `/api/my-jobs/{id}/status` | Technician status update | Technician |

#### Job Enhancements (All authenticated roles)

| Method | Route | Description |
|--------|-------|-------------|
| GET/POST | `/api/service-jobs/{id}/attachments` | List/upload attachments |
| DELETE | `/api/service-jobs/{id}/attachments/{aid}` | Delete attachment |
| PATCH | `/api/service-jobs/{id}/checklist/{item}/toggle` | Toggle checklist item |
| POST | `/api/service-jobs/{id}/signature` | Capture digital signature |
| GET/POST | `/api/service-jobs/{id}/comments` | List/add comments |
| DELETE | `/api/service-jobs/{id}/comments/{cid}` | Delete comment |

#### Job Parts (All authenticated roles)

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/service-jobs/{id}/parts` | List parts on job |
| POST | `/api/service-jobs/{id}/parts` | Add part to job |
| PUT | `/api/service-jobs/{id}/parts/{pid}` | Update part quantity |
| DELETE | `/api/service-jobs/{id}/parts/{pid}` | Remove part from job |

#### Customers

| Method | Route | Description | Roles |
|--------|-------|-------------|-------|
| GET | `/api/customers` | List customers | Admin, Dispatcher |
| POST | `/api/customers` | Create customer | Admin, Dispatcher |
| GET | `/api/customers/{id}` | Get customer | Admin, Dispatcher |
| PUT | `/api/customers/{id}` | Update customer | Admin, Dispatcher |
| DELETE | `/api/customers/{id}` | Delete customer | Admin, Dispatcher |

#### Services

| Method | Route | Description | Roles |
|--------|-------|-------------|-------|
| GET | `/api/services` | List services | Admin, Dispatcher |
| POST | `/api/services` | Create service | Admin |
| GET | `/api/services/{id}` | Get service | Admin, Dispatcher |
| PUT | `/api/services/{id}` | Update service | Admin |
| DELETE | `/api/services/{id}` | Delete service | Admin |

#### Parts / Inventory

| Method | Route | Description | Roles |
|--------|-------|-------------|-------|
| GET | `/api/parts` | List parts | Admin, Dispatcher |
| POST | `/api/parts` | Create part | Admin |
| GET | `/api/parts/{id}` | Get part | Admin, Dispatcher |
| PUT | `/api/parts/{id}` | Update part | Admin |
| DELETE | `/api/parts/{id}` | Delete part | Admin |
| GET | `/api/parts/low-stock` | Low stock alerts | Admin |
| POST | `/api/parts/{id}/adjust-stock` | Manual stock adjustment | Admin |

#### Invoices

| Method | Route | Description | Roles |
|--------|-------|-------------|-------|
| GET | `/api/invoices` | List invoices | Admin, Dispatcher |
| POST | `/api/invoices` | Create invoice | Admin, Dispatcher |
| GET | `/api/invoices/{id}` | Get invoice | Admin, Dispatcher |
| PUT | `/api/invoices/{id}` | Update invoice | Admin, Dispatcher |
| GET | `/api/invoices/{id}/pdf` | Download PDF | Admin, Dispatcher |

#### Users

| Method | Route | Description | Roles |
|--------|-------|-------------|-------|
| GET | `/api/users` | List users | Admin |
| POST | `/api/users` | Create user | Admin |
| GET | `/api/users/{id}` | Get user | Admin |
| PUT | `/api/users/{id}` | Update user | Admin |
| DELETE | `/api/users/{id}` | Delete user | Admin |

#### GPS & Routing

| Method | Route | Description | Roles |
|--------|-------|-------------|-------|
| POST | `/api/my-location` | Record location | Technician |
| GET | `/api/my-route` | Get optimized route | Technician |
| GET | `/api/technicians/locations` | All tech locations | Admin, Dispatcher |
| GET | `/api/technicians/{id}/location` | Specific tech location | Admin, Dispatcher |
| GET | `/api/technicians/{id}/route` | Tech route | Admin, Dispatcher |
| GET | `/api/service-jobs/{id}/eta` | Job ETA | Admin, Dispatcher |
| GET | `/api/my-jobs/{id}/eta` | My job ETA | Technician |

#### Reporting & Analytics (Admin, Dispatcher)

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/reports/summary` | Summary statistics |
| GET | `/api/reports/jobs-by-status` | Jobs grouped by status |
| GET | `/api/reports/jobs-by-date` | Jobs grouped by date |
| GET | `/api/reports/technician-performance` | Technician metrics |
| GET | `/api/analytics/revenue-trend` | Monthly revenue trend |
| GET | `/api/analytics/job-trend` | Weekly job trend |
| GET | `/api/analytics/service-popularity` | Service popularity |
| GET | `/api/analytics/customer-lifetime-value` | Customer LTV |
| GET | `/api/analytics/job-profitability` | Job profit analysis |

#### Exports (Admin, Dispatcher)

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/export/jobs` | Export jobs to CSV |
| GET | `/api/export/invoices` | Export invoices to CSV |
| GET | `/api/export/customers` | Export customers to CSV |
| GET | `/api/export/technician-performance` | Export tech performance to CSV |

#### Admin-Only Resources

| Method | Route | Description |
|--------|-------|-------------|
| CRUD | `/api/tenants` | Tenant management |
| CRUD | `/api/roles` | Role management |
| GET/POST | `/api/permissions` | Permission management |
| GET | `/api/audit-logs` | Audit trail |
| CRUD | `/api/scheduled-reports` | Scheduled report management |
| CRUD | `/api/services/{id}/checklist` | Service checklist management |

#### Sync (Technician)

| Method | Route | Description |
|--------|-------|-------------|
| POST | `/api/sync` | Offline data sync (status updates, locations, checklists, comments) |

#### Customer Portal

| Method | Route | Description |
|--------|-------|-------------|
| POST | `/api/portal/login` | Customer login |
| GET | `/api/portal/me` | Customer profile |
| POST | `/api/portal/logout` | Customer logout |
| GET | `/api/portal/jobs` | My jobs |
| GET | `/api/portal/jobs/{id}` | Job detail |
| GET | `/api/portal/jobs/{id}/eta` | Job ETA |
| GET | `/api/portal/invoices` | My invoices |
| GET | `/api/portal/invoices/{id}` | Invoice detail |
| GET | `/api/portal/invoices/{id}/pdf` | Download invoice PDF |
| GET | `/api/portal/services` | Available services |
| POST | `/api/portal/service-requests` | Submit service request |
| GET | `/api/portal/service-requests` | My service requests |
| POST | `/api/portal/jobs/{id}/review` | Submit review |
| GET | `/api/portal/jobs/{id}/review` | Get review |
| GET | `/api/portal/reviews` | All my reviews |

### 5.5 Middleware Pipeline

```
Request → TenantRateLimiter → auth:sanctum → ResolveTenant → EnsureRole → CheckPermission → Controller
```

| Middleware | Purpose |
|-----------|---------|
| **TenantRateLimiter** | Plan-based rate limiting (free: 100/min, basic: 300, pro: 1000, enterprise: 5000) |
| **auth:sanctum** | Validates API token/session |
| **ResolveTenant** | Binds `current_tenant_id` to container; activates `TenantScope` |
| **EnsureRole** | Verifies user has one of the required roles |
| **CheckPermission** | Checks fine-grained permissions (admins bypass) |

### 5.6 Scheduled Tasks

| Schedule | Command | Description |
|----------|---------|-------------|
| Daily 06:00 | `jobs:generate-recurring` | Creates upcoming jobs from recurring templates (7-day lookahead) |
| Daily 07:00 | `reports:send-scheduled` | Sends scheduled reports to configured email recipients |

### 5.7 Notifications

| Notification | Channels | Trigger |
|-------------|----------|---------|
| `JobStatusChanged` | Database, Mail | Job status transition |
| `JobAssigned` | Database, Mail | Technician assigned to job |
| `CustomerJobUpdate` | Mail only | Status updates for customer (on_the_way, completed) |
| `ScheduledReportMail` | Mail only | Automated scheduled report delivery |

---

## 6. Frontend Documentation

### 6.1 Pages & Routes

#### Public Routes

| Route | Page | Description |
|-------|------|-------------|
| `/` | Home | Redirects to role-specific dashboard |
| `/login` | Login | Email/password authentication |

#### Admin Routes (`/admin/*`)

| Route | Description |
|-------|-------------|
| `/admin` | Dashboard with system-wide statistics |
| `/admin/users` | User list, create, view, edit |
| `/admin/customers` | Customer list, create, view, edit |
| `/admin/jobs` | Job list, create, view, edit, delete |
| `/admin/services` | Service catalog management |
| `/admin/parts` | Parts/inventory management |
| `/admin/invoices` | Invoice list, detail view |
| `/admin/tenants` | Tenant list with create dialog, detail/edit |
| `/admin/roles` | Role & permission management |
| `/admin/analytics` | Business analytics dashboards |
| `/admin/reports` | Job reports with date filtering |
| `/admin/calendar` | Calendar view of scheduled jobs |
| `/admin/audit-logs` | System audit trail |

#### Dispatcher Routes (`/dispatcher/*`)

| Route | Description |
|-------|-------------|
| `/dispatcher` | Dispatcher dashboard |
| `/dispatcher/jobs` | Job dispatch management (CRUD) |
| `/dispatcher/customers` | Customer management (CRUD) |
| `/dispatcher/calendar` | Job scheduling calendar |
| `/dispatcher/reports` | Dispatch reports |

#### Technician Routes (`/technician/*`)

| Route | Description |
|-------|-------------|
| `/technician` | Dashboard with today's jobs |
| `/technician/my-jobs` | My assigned jobs with status filters |
| `/technician/my-jobs/{id}` | Job detail with status progression |

### 6.2 Component Architecture

#### UI Components (Reusable Design System)

All UI components use CVA (class-variance-authority) for variant-based styling:

| Component | Variants | Props |
|-----------|----------|-------|
| **Button** | primary, secondary, destructive, ghost, outline | size (sm/md/lg), loading, disabled |
| **Card** | default, interactive, elevated | padding (sm/md/lg) |
| **Input** | - | label, error, helperText, fullWidth |
| **Select** | - | options, label, error |
| **Dialog** | - | open, onClose, title, size (sm/md/lg) |
| **Alert** | error, success, warning, info | dismissible |
| **Badge** | success, warning, danger, info, neutral, primary, accent | - |
| **Table** | - | Compound: Table, TableHeader, TableBody, TableHead, TableRow, TableCell |
| **Pagination** | - | currentPage, totalPages, onPageChange |
| **PageHeader** | - | title, subtitle, backLink, actions |
| **FormField** | - | label, error, children |
| **Textarea** | - | label, error, rows |

#### Layout Components

- **Sidebar** - Responsive navigation (mobile drawer, desktop fixed). Role-based menu items.
- **TopBar** - Desktop header with search, notifications, user profile
- **NotificationBell** - Unread notification indicator with dropdown

#### Feature Components

| Domain | Components | Key Features |
|--------|-----------|--------------|
| **Jobs** | JobList, JobForm, JobDetail, JobTabs, JobStatusHistory, JobComments, JobAttachments, JobChecklist, TechnicianAssignPanel, TechnicianJobCard | Status filtering, technician assignment, file uploads, checklist toggles |
| **Customers** | CustomerList, CustomerForm, CustomerDetail | Search, pagination, associated jobs |
| **Services** | ServiceList, ServiceForm, ServiceDetail | CRUD with pricing |
| **Users** | UserList, UserForm, UserDetail | Role management, active/inactive toggle |
| **Parts** | PartList, PartForm, PartDetail, JobPartsPanel | Stock levels, low stock alerts, job parts management |
| **Invoices** | InvoiceList, InvoiceDetail | Status tracking, PDF download |
| **Calendar** | CalendarView | Visual job scheduling |
| **Reports** | ReportsPage, DateRangeFilter, StatusBreakdown, TechnicianPerformanceTable, DailyJobsTable | Date-range filtering, tabular data |

### 6.3 State Management

**AuthContext** is the sole global state provider:

```typescript
interface AuthContextType {
  user: User | null;
  loading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
}
```

- Wraps the entire app via `AuthProvider` in root layout
- On mount: fetches `/api/me` to restore session
- On login: POST `/api/login`, then fetch user
- On logout: POST `/api/logout`, clear cache, set user to null

All other state is component-local (`useState`, `useEffect`).

### 6.4 API Client

The singleton `ApiClient` class provides:

- **CSRF handling** - Automatically reads `XSRF-TOKEN` cookie and sends as `X-XSRF-TOKEN` header
- **Caching** - In-memory cache with configurable TTL (default 30s)
- **Request deduplication** - Concurrent identical GET requests return same promise
- **Error handling** - Throws `ApiError` with `status` and `errors` (validation) fields
- **Methods** - `get<T>()`, `post<T>()`, `put<T>()`, `patch<T>()`, `delete<T>()`

```typescript
// Usage examples
const jobs = await api.get<PaginatedResponse<ServiceJob>>('/service-jobs', { page: '1', status: 'pending' });
await api.patch(`/service-jobs/${id}/status`, { status: 'completed' });
```

### 6.5 TypeScript Types

Key type definitions in `src/types/index.ts`:

```typescript
type Role = 'admin' | 'dispatcher' | 'technician';
type JobStatus = 'pending' | 'assigned' | 'on_the_way' | 'in_progress' | 'completed' | 'cancelled';
type JobPriority = 'low' | 'medium' | 'high' | 'urgent';
type InvoiceStatus = 'draft' | 'sent' | 'paid' | 'cancelled';
type TenantPlan = 'free' | 'basic' | 'pro' | 'enterprise';

// Paginated API response
interface PaginatedResponse<T> {
  data: T[];
  meta: { current_page: number; last_page: number; per_page: number; total: number };
  links: { first: string; last: string; prev: string | null; next: string | null };
}
```

### 6.6 Styling & Theming

- **CSS Framework**: Tailwind CSS 4.2 with custom design tokens
- **Color Palette**: Primary (indigo `#6366f1`), Accent (teal `#0d9488`), Destructive (red), Alert (orange)
- **Component Styling**: CVA for variants, `twMerge` for class conflict resolution
- **Responsive**: Mobile-first with `md:` and `lg:` breakpoints
- **Design System**: CSS custom properties for colors, shadows, border-radius

---

## 7. Setup Guide

### 7.1 Prerequisites

- Docker & Docker Compose
- Git (with SSH key for submodule access)

### 7.2 Installation

```bash
# 1. Clone the repository with submodules
git clone --recurse-submodules git@github.com:Jerven-git/dispatching-system.git
cd dispatching-system

# 2. Configure environment variables
cp .env.example .env
# Edit .env with your database credentials:
#   DB_DATABASE=dispatching
#   DB_USERNAME=dispatching_user
#   DB_PASSWORD=<secure-password>
#   DB_ROOT_PASSWORD=<secure-root-password>

# 3. Configure backend environment
cp dispatching-backend/.env.example dispatching-backend/.env
# Edit dispatching-backend/.env:
#   DB_CONNECTION=mysql
#   DB_HOST=db
#   DB_PORT=3306
#   DB_DATABASE=dispatching
#   DB_USERNAME=dispatching_user
#   DB_PASSWORD=<same-as-above>

# 4. Configure frontend environment
cp dispatching-frontend/.env.example dispatching-frontend/.env
# Edit dispatching-frontend/.env:
#   NEXT_PUBLIC_API_BASE_URL=http://localhost:8000

# 5. Build and start containers
docker compose build
docker compose up -d

# 6. Install backend dependencies & set up database
docker compose exec backend composer install
docker compose exec backend php artisan key:generate
docker compose exec backend php artisan migrate
docker compose exec backend php artisan db:seed

# 7. Install frontend dependencies (if not done by Dockerfile)
docker compose exec frontend bun install
```

### 7.3 Environment Variables

#### Root `.env` (Docker Compose)

| Variable | Description | Example |
|----------|-------------|---------|
| `DB_DATABASE` | MySQL database name | `dispatching` |
| `DB_USERNAME` | MySQL user | `dispatching_user` |
| `DB_PASSWORD` | MySQL password | `secure_password` |
| `DB_ROOT_PASSWORD` | MySQL root password | `root_secure_password` |

#### Backend `.env` (Laravel)

| Variable | Description | Default |
|----------|-------------|---------|
| `APP_NAME` | Application name | `Laravel` |
| `APP_ENV` | Environment | `local` |
| `APP_KEY` | Encryption key | Generated via `artisan key:generate` |
| `APP_DEBUG` | Debug mode | `true` |
| `APP_URL` | Backend URL | `http://localhost:8000` |
| `DB_CONNECTION` | Database driver | `mysql` |
| `DB_HOST` | Database host | `db` (Docker service name) |
| `DB_PORT` | Database port | `3306` |
| `QUEUE_CONNECTION` | Queue driver | `database` |
| `MAIL_MAILER` | Mail driver | `log` (development) |

#### Frontend `.env`

| Variable | Description | Default |
|----------|-------------|---------|
| `NEXT_PUBLIC_API_BASE_URL` | Backend API base URL | `http://localhost:8000` |
| `APP_NAME` | App display name | `Dispatching Frontend` |
| `ENVIRONMENT` | Environment name | `development` |

### 7.4 Accessing the Application

| Service | URL | Purpose |
|---------|-----|---------|
| Frontend | `http://localhost:3000` | Next.js development server |
| API (via Nginx) | `http://localhost:8000` | Laravel API |
| phpMyAdmin | `http://localhost:8001` | Database management |

### 7.5 Default Seed Data

After running `php artisan db:seed`:

| User | Email | Role |
|------|-------|------|
| Admin | (see DatabaseSeeder) | admin |
| Dispatcher | (see DatabaseSeeder) | dispatcher |
| Technician 1 | (see DatabaseSeeder) | technician |
| Technician 2 | (see DatabaseSeeder) | technician |

Plus: 5 services, 2 customers, 2 sample jobs, 40+ permissions, and system roles.

### 7.6 Docker Services

| Service | Image | Ports | Purpose |
|---------|-------|-------|---------|
| `frontend` | dispatching-frontend:dev | 3000, 24678 | Next.js dev server + HMR |
| `backend` | dispatching-backend:dev | (internal 9000) | PHP-FPM application |
| `scheduler` | dispatching-backend:dev | - | Laravel scheduler (`schedule:work`) |
| `nginx` | nginx:alpine | 8000 | Reverse proxy |
| `db` | mysql:latest | (internal 3306) | MySQL database |
| `phpmyadmin` | phpmyadmin:5 | 8001 | Database admin UI |

---

## 8. Usage Guide

### 8.1 Admin Workflow

1. **Login** at `/login` with admin credentials
2. **Dashboard** shows system-wide stats (pending, assigned, in-progress, completed jobs)
3. **Manage Users** - Create dispatcher/technician accounts under Admin > Users
4. **Configure Services** - Define service catalog with pricing under Admin > Services
5. **Manage Parts** - Add inventory items with SKU, pricing, and minimum stock levels
6. **Create Jobs** - Admin > Jobs > Create Job, fill customer/service/technician/schedule
7. **Assign Technicians** - From job detail, click "Assign" and select available technician
8. **Track Progress** - View status changes, comments, and attachments on job detail
9. **Generate Invoices** - Create invoices from completed jobs, download PDFs
10. **View Reports** - Admin > Reports/Analytics for performance insights
11. **Export Data** - Download CSV exports of jobs, invoices, customers

### 8.2 Dispatcher Workflow

1. **Login** → Dispatcher Dashboard with today's job overview
2. **Create Jobs** - Enter customer, service, schedule, priority
3. **Assign Jobs** - Select technician based on workload view
4. **Monitor Jobs** - Track status via job list and calendar view
5. **Manage Customers** - Add/edit customer information

### 8.3 Technician Workflow

1. **Login** → Technician Dashboard showing today's assigned jobs
2. **View My Jobs** - Filter by status (assigned, on_the_way, in_progress)
3. **Update Status** - Progress through: Assigned → On The Way → In Progress → Completed
4. **Add Details** - Upload before/after photos, complete checklists, add notes
5. **Capture Signature** - Get customer signature on completion
6. **Record Parts** - Log parts used on the job
7. **GPS Tracking** - Location automatically recorded for route optimization

### 8.4 Customer Portal Workflow

1. **Login** at `/api/portal/login` with customer credentials
2. **View Jobs** - See all service jobs and their current status
3. **Track ETA** - View technician ETA when status is "on the way"
4. **Request Service** - Submit new service requests
5. **View Invoices** - Access and download invoice PDFs
6. **Leave Reviews** - Rate completed jobs (1-5 stars) with comments

---

## 9. Code Review

### 9.1 Strengths

- **Clean Architecture** - Clear separation: Controllers → Services → Models. Business logic is not in controllers.
- **Comprehensive Feature Set** - Covers the full field service lifecycle: scheduling, dispatch, execution, invoicing, reporting.
- **Multi-Tenancy** - Well-implemented via global scopes and traits; data isolation is automatic.
- **Type Safety** - Frontend uses TypeScript with comprehensive type definitions matching backend responses.
- **API Resource Layer** - Consistent JSON output formatting across all endpoints.
- **Form Request Validation** - Input validation separated from controller logic with 13 dedicated request classes.
- **Status Machine** - Explicit transition rules prevent invalid job status changes.
- **Reusable UI Components** - CVA-based design system with consistent styling patterns.
- **API Client Design** - Request caching and deduplication reduce unnecessary network calls.
- **Audit Trail** - Complete change tracking via the Auditable trait across models.
- **Role-Based + Permission-Based Access** - Dual-layer access control (role guards + granular permissions).

### 9.2 Weaknesses

- **No Automated Test Coverage** - Test directories exist but no meaningful test suites were found. This is a significant gap for a system of this complexity.
- **SQLite Default in Backend** - The `.env.example` defaults to SQLite, but production uses MySQL. This mismatch could cause migration/query differences.
- **No Queue Worker Container** - Events and notifications reference `ShouldQueue` but no dedicated queue worker container is defined in Docker Compose.
- **Client-Side Only Caching** - The API client caches in memory (lost on page reload). No server-side caching (Redis) is configured.
- **No WebSocket/Real-Time** - GPS tracking and job status updates require polling; no real-time push mechanism.
- **Route Optimization is Naive** - Nearest-neighbor algorithm is O(n^2) and does not guarantee optimal routes.
- **ETA Calculation is Approximate** - Uses Haversine distance with a fixed road factor (1.4x) instead of real routing APIs.
- **No File Upload Validation** - Attachment uploads should validate file types and sizes server-side more rigorously.
- **Production Config Placeholder** - `production.conf` has `domain_name_here.com.au` as a placeholder; needs templating.
- **Large Upload Limit in Dev** - The dev Nginx config allows 10GB uploads (`client_max_body_size 10000M`).

### 9.3 Technical Debt

| Area | Issue | Priority |
|------|-------|----------|
| Testing | No unit, integration, or E2E tests | High |
| Error Handling | Some controllers return raw exceptions instead of structured error responses | Medium |
| Database | SQLite default mismatches MySQL production; potential query incompatibilities | Medium |
| Caching | No server-side cache (Redis/Memcached) for frequently accessed data | Medium |
| Real-Time | No WebSocket support for live updates (GPS, job status) | Medium |
| Customer Portal Frontend | Portal is API-only; no dedicated frontend pages for customers | Medium |
| Documentation | No custom README (uses default Laravel README) | Low |
| Dead Code | `app/Listeners/` directory appears empty | Low |

### 9.4 Suggested Improvements

1. **Add Test Suites** - Write PHPUnit tests for services (especially `StatusTransitionService`, `InventoryService`) and feature tests for API endpoints.
2. **Add Redis** - Use Redis for caching, sessions, and queues instead of database driver.
3. **Add Queue Worker** - Add a dedicated `worker` container running `php artisan queue:work` to process notifications asynchronously.
4. **Real-Time Updates** - Implement Laravel Broadcasting with Pusher/Soketi for live job status and GPS updates.
5. **Integrate Routing API** - Replace Haversine ETA with Google Maps/Mapbox Directions API for accurate ETAs.
6. **Customer Portal Frontend** - Build Next.js pages for the customer portal (currently API-only).
7. **E2E Testing** - Add Cypress/Playwright tests for critical user flows (login, create job, assign, complete).
8. **API Documentation** - Generate OpenAPI/Swagger spec from routes and form requests.

---

## 10. Security Review

### 10.1 Authentication

| Mechanism | Implementation | Assessment |
|-----------|---------------|------------|
| **User Auth** | Laravel Sanctum (cookie-based SPA auth) | Good - industry standard |
| **Customer Auth** | Separate guard with session-based auth | Good - isolated from user auth |
| **CSRF Protection** | XSRF-TOKEN cookie + X-XSRF-TOKEN header | Good - automatic via Sanctum |
| **Password Hashing** | bcrypt via Laravel `Hash` facade | Good - secure default |
| **Rate Limiting** | Login: 5/min; API: plan-based (100-5000/min) | Good |

### 10.2 Authorization

| Layer | Implementation | Assessment |
|-------|---------------|------------|
| **Role-Based** | `EnsureRole` middleware on route groups | Good |
| **Permission-Based** | `CheckPermission` middleware + `hasPermission()` | Good |
| **Tenant Isolation** | Global scope auto-filters all queries | Good - prevents cross-tenant access |
| **Form Requests** | Authorization checks in `authorize()` methods | Good |

### 10.3 Input Validation

| Area | Implementation | Assessment |
|------|---------------|------------|
| **Form Requests** | 13 dedicated validation classes | Good |
| **SQL Injection** | Eloquent ORM parameterized queries | Good |
| **XSS** | React auto-escapes output by default | Good |
| **Mass Assignment** | `$fillable` defined on all models | Good |

### 10.4 Potential Vulnerabilities

| Risk | Description | Severity | Mitigation |
|------|-------------|----------|------------|
| **File Upload** | Attachments may not fully validate file types server-side | Medium | Add MIME type validation and virus scanning |
| **Signature Storage** | Digital signatures stored as file paths; ensure path traversal prevention | Low | Validate file paths, use storage disk abstraction |
| **API Error Exposure** | `APP_DEBUG=true` may expose stack traces in production | Medium | Ensure `APP_DEBUG=false` in production `.env` |
| **CORS Configuration** | CORS config should be restrictive in production | Medium | Verify `config/cors.php` allows only trusted origins |
| **Sensitive Data in Logs** | `MAIL_MAILER=log` logs email content including reports | Low | Use proper mail driver in production |
| **phpMyAdmin Exposed** | phpMyAdmin on port 8001 in dev; must not be in production compose | Medium | Production compose correctly excludes it |
| **Large Upload Limit** | Dev Nginx allows 10GB uploads | Low | Production limits to 100MB (correct) |

### 10.5 Recommendations

1. **Enforce HTTPS** in production (already configured in `production.conf`)
2. **Remove `APP_DEBUG=true`** in production environment
3. **Add Content Security Policy** headers in Nginx
4. **Validate uploaded file MIME types** beyond just extension checking
5. **Add API rate limiting per endpoint** for sensitive operations (password reset, etc.)
6. **Rotate Sanctum tokens** periodically or implement token expiry

---

## 11. Scalability

### 11.1 Current Architecture Limitations

| Area | Limitation |
|------|-----------|
| **Database** | Single MySQL instance; no read replicas or sharding |
| **Session/Cache** | Database-backed; adds load to primary DB |
| **Queue** | Database queue driver; competes with application queries |
| **File Storage** | Local disk; not scalable across multiple app servers |
| **Search** | SQL `LIKE` queries; no full-text search engine |
| **Real-Time** | No WebSocket; clients must poll for updates |
| **Multi-Tenancy** | Shared database; all tenants in same tables |
| **Horizontal Scaling** | Sticky sessions required (no external session store) |

### 11.2 Scaling Recommendations

#### Short-Term (Quick Wins)

1. **Add Redis** - Replace database driver for cache, sessions, and queues
2. **Add Queue Worker** - Dedicated container for async notification/report processing
3. **S3 File Storage** - Move file uploads to S3/MinIO for scalable object storage
4. **Database Indexing** - Add composite indexes on frequently queried columns (`tenant_id` + `status`, `scheduled_date`)

#### Medium-Term

5. **Read Replicas** - MySQL read replica for reporting/analytics queries
6. **WebSocket Server** - Laravel Reverb or Soketi for real-time updates
7. **CDN** - Serve frontend static assets via CloudFront/Cloudflare
8. **Elasticsearch** - Full-text search for jobs, customers, audit logs
9. **Horizontal App Scaling** - Multiple backend containers behind load balancer (requires Redis sessions)

#### Long-Term

10. **Database-per-Tenant** - For enterprise customers with strict data isolation requirements
11. **Microservices Extraction** - Split notification, reporting, and GPS tracking into independent services
12. **Event Sourcing** - For audit-heavy modules (jobs, invoices) where complete history is critical
13. **Geographic Distribution** - Multi-region deployment for Australian/global customers

### 11.3 Performance Benchmarks to Establish

- API response times (p50, p95, p99) under load
- Database query performance for large tenant datasets (10K+ jobs)
- Concurrent user capacity per plan tier
- File upload throughput and storage growth rate
- Report generation time for large date ranges

---

*Document generated: 2026-04-04*
*System version: Phases 1-5 complete*
