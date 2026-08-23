# RentNest (Rental Property Marketplace API)

A complete backend API for a rental property marketplace with three roles (Tenant, Landlord, Admin) built with Express, TypeScript, Prisma ORM, PostgreSQL, JWT authentication, and Stripe payments.

---

## Tech Stack

- **Runtime**: Node.js (ESM)
- **Framework**: Express.js
- **Language**: TypeScript (strict mode)
- **ORM**: Prisma
- **Database**: PostgreSQL
- **Auth**: JWT (access + refresh tokens in httpOnly cookies)
- **Payments**: Stripe Checkout Sessions
- **Validation**: Zod
- **Build**: tsup
- **Deployment**: Vercel

---

## Database Schema

[View ERD](./ERD.svg)

All models use UUID primary keys. Monetary values use `Decimal` for precision.

### User

| Field          | Type     | Constraints                     |
| -------------- | -------- | ------------------------------- |
| `id`           | UUID     | Primary key, `@default(uuid())` |
| `email`        | String   | `@unique`                       |
| `passwordHash` | String   |                                 |
| `role`         | UserRole | TENANT / LANDLORD / ADMIN       |
| `name`         | String   |                                 |
| `phone`        | String?  | Optional                        |
| `avatar`       | String?  | Optional                        |
| `isBanned`     | Boolean  | `@default(false)`               |
| `banReason`    | String?  | Optional                        |
| `createdAt`    | DateTime | `@default(now())`               |
| `updatedAt`    | DateTime | `@updatedAt`                    |

### Category

| Field         | Type     | Constraints                     |
| ------------- | -------- | ------------------------------- |
| `id`          | UUID     | Primary key, `@default(uuid())` |
| `name`        | String   | `@unique`                       |
| `description` | String?  | Optional                        |
| `createdAt`   | DateTime | `@default(now())`               |
| `updatedAt`   | DateTime | `@updatedAt`                    |

### Property

| Field             | Type           | Constraints                      |
| ----------------- | -------------- | -------------------------------- |
| `id`              | UUID           | Primary key, `@default(uuid())`  |
| `title`           | String         |                                  |
| `description`     | String         |                                  |
| `location`        | String         |                                  |
| `mapLocation`     | String?        | Optional                         |
| `status`          | PropertyStatus | AVAILABLE / RENTED / UNAVAILABLE |
| `images`          | String[]       | Array of image URLs              |
| `averageRating`   | Float?         | Optional                         |
| `securityDeposit` | Decimal        | `@db.Decimal(10, 2)`             |
| `monthlyRent`     | Decimal        | `@db.Decimal(10, 2)`             |
| `landlordId`      | UUID           | Foreign key to User              |
| `categoryId`      | UUID           | Foreign key to Category          |
| `createdAt`       | DateTime       | `@default(now())`                |
| `updatedAt`       | DateTime       | `@updatedAt`                     |

### Request

| Field               | Type          | Constraints                     |
| ------------------- | ------------- | ------------------------------- |
| `id`                | UUID          | Primary key, `@default(uuid())` |
| `tenantId`          | UUID          | Foreign key to User             |
| `propertyId`        | UUID          | Foreign key to Property         |
| `message`           | String?       | Optional                        |
| `status`            | RequestStatus | 8 states (see flow diagram)     |
| `securityDeposit`   | Decimal       | `@db.Decimal(10, 2)`            |
| `monthlyRent`       | Decimal       | `@db.Decimal(10, 2)`            |
| `moveInDate`        | DateTime      |                                 |
| `moveInApprovedAt`  | DateTime?     | Optional                        |
| `moveOutDate`       | DateTime?     | Optional                        |
| `damageAmount`      | Decimal?      | `@db.Decimal(10, 2)` Optional   |
| `moveOutApprovedAt` | DateTime?     | Optional                        |
| `completedAt`       | DateTime?     | Optional                        |
| `rejectedReason`    | String?       | Optional                        |
| `rejectedAt`        | DateTime?     | Optional                        |
| `createdAt`         | DateTime      | `@default(now())`               |
| `updatedAt`         | DateTime      | `@updatedAt`                    |

### Payment

| Field                   | Type            | Constraints                                       |
| ----------------------- | --------------- | ------------------------------------------------- |
| `id`                    | UUID            | Primary key, `@default(uuid())`                   |
| `transactionId`         | String          | `@unique`                                         |
| `requestId`             | UUID            | Foreign key to Request                            |
| `userId`                | UUID            | Foreign key to User                               |
| `amount`                | Decimal         | `@db.Decimal(10, 2)`                              |
| `currency`              | String          | Default: `BDT`                                    |
| `type`                  | PaymentType     | SECURITY_DEPOSIT / MONTHLY_RENT / MOVE_OUT_REFUND |
| `status`                | PaymentStatus   | PENDING / PAID / FAILED / REFUNDED                |
| `provider`              | PaymentProvider | STRIPE                                            |
| `stripePaymentIntentId` | String?         | Stripe Session ID, `@unique`                      |
| `paidAt`                | DateTime?       | Optional                                          |
| `periodStart`           | DateTime?       | Optional                                          |
| `periodEnd`             | DateTime?       | Optional                                          |
| `createdAt`             | DateTime        | `@default(now())`                                 |
| `updatedAt`             | DateTime        | `@updatedAt`                                      |

### Review

| Field        | Type     | Constraints                       |
| ------------ | -------- | --------------------------------- |
| `id`         | UUID     | Primary key, `@default(uuid())`   |
| `tenantId`   | UUID     | Foreign key to User               |
| `propertyId` | UUID     | Foreign key to Property           |
| `requestId`  | UUID     | `@unique` (one review per rental) |
| `rating`     | Int      | 1-5                               |
| `comment`    | String?  | Optional                          |
| `createdAt`  | DateTime | `@default(now())`                 |
| `updatedAt`  | DateTime | `@updatedAt`                      |

---

## Project Structure

```
src/
├── config/
│   └── index.ts           # Environment config (validated)
├── lib/
│   ├── prisma.ts          # Prisma client
│   └── stripe.ts          # Stripe client
├── middleware/
│   ├── auth.ts            # JWT auth + role authorization
│   ├── global-error.ts    # Global error handler
│   ├── not-found.ts       # 404 handler
│   └── validate.ts        # Zod validation wrapper
├── modules/
│   ├── admin/             # User mgmt + platform oversight
│   ├── auth/              # Register, login, refresh, logout, me
│   ├── category/          # Categories (admin create, public read)
│   ├── landlord/          # Landlord property management
│   ├── payment/           # Stripe checkout + webhook
│   ├── property/          # Public property browse + landlord CRUD
│   ├── request/           # Rental requests (tenant/landlord flow)
│   └── review/            # Property reviews (tenant, after move-out)
├── types/
│   └── index.ts           # Global types (IAuthRequest, etc.)
├── utils/
│   ├── app-error.ts       # AppError class
│   ├── catch-async.ts     # Async wrapper
│   ├── jwt.ts             # JWT helpers
│   ├── password.ts        # Password hashing
│   └── send-response.ts   # Standardized response format
├── app.ts                 # Express app setup
└── server.ts              # Entry point
```
---

## Prerequisites

- Node.js 18+
- PostgreSQL 14+
- Stripe account (test mode)

---

## Setup

```bash
# Install dependencies
npm install

# Configure environment
cp .env.example .env   # then edit values

# Database setup
npm run db:generate
npm run db:migrate     # or: npx prisma db push

# Seed database (creates test users, categories, properties, requests, reviews)
# Credentials loaded from config (ADMIN_EMAIL, ADMIN_PASSWORD)
npm run db:seed

# Development
npm run dev

# Build
npm run build

# Deploy
npm run deploy
```

---

## Available Scripts

| Command               | Description                                               |
| --------------------- | --------------------------------------------------------- |
| `npm run build`       | Type-check + build with tsup (includes `prisma generate`) |
| `npm run db:generate` | Generate Prisma Client                                    |
| `npm run db:migrate`  | Run Prisma migrations (dev)                               |
| `npm run db:seed`     | Seed database with test data                              |
| `npm run deploy`      | Deploy to Vercel                                          |
| `npm run dev`         | Development with tsx watch                                |

---

## Environment Variables

| Variable                        | Required | Description                                        |
| ------------------------------- | -------- | -------------------------------------------------- |
| `NODE_ENV`                      | Yes      | `development` or `production`                      |
| `PORT`                          | Yes      | Server port (default: 5000)                        |
| `FRONTEND_URL`                  | Yes      | Frontend URL for Stripe redirects                  |
| `DATABASE_URL`                  | Yes      | PostgreSQL connection string                       |
| `BCRYPT_SALT_ROUNDS`            | Yes      | Password hash rounds (10/12/14)                    |
| `JWT_ACCESS_SECRET`             | Yes      | 64+ char random string                             |
| `JWT_ACCESS_SECRET_EXPIRES_IN`  | Yes      | Access token TTL (15m/1h/1d)                       |
| `JWT_REFRESH_SECRET`            | Yes      | 64+ char random string (different from access)     |
| `JWT_REFRESH_SECRET_EXPIRES_IN` | Yes      | Refresh token TTL (7d/30d)                         |
| `STRIPE_SECRET_KEY`             | Yes      | Stripe secret key (`sk_test_...` or `sk_live_...`) |
| `STRIPE_WEBHOOK_SECRET`         | Yes      | Stripe webhook signing secret (`whsec_...`)        |
| `ADMIN_EMAIL`                   | No       | Seeded admin email (used by seed script)           |
| `ADMIN_PASSWORD`                | No       | Seeded admin password (used by seed script)        |
| `CRON_SECRET`                   | No       | Secret for cron endpoint auth (Vercel Cron)        |

---

## API Endpoints

All responses follow:
```json
{
  "success": true,
  "statusCode": 200,
  "message": "string",
  "data": {},
  "meta": {}
}
```

### Authentication (`/api/auth`)

| Method | Endpoint    | Role          | Description                             |
| ------ | ----------- | ------------- | --------------------------------------- |
| POST   | `/register` | Public        | Register new user (TENANT, LANDLORD)    |
| POST   | `/login`    | Public        | Login, sets httpOnly cookies            |
| POST   | `/refresh`  | Authenticated | Refresh access token via refresh cookie |
| GET    | `/me`       | Authenticated | Get current user profile                |
| POST   | `/logout`   | Authenticated | Clear cookies                           |

### Categories (`/api/categories`)

| Method | Endpoint | Role   | Description                     |
| ------ | -------- | ------ | ------------------------------- |
| GET    | `/`      | Public | List all categories (paginated) |
| GET    | `/:id`   | Public | Get category by ID              |
| POST   | `/`      | ADMIN  | Create category                 |
| PUT    | `/:id`   | ADMIN  | Update category                 |
| DELETE | `/:id`   | ADMIN  | Delete category                 |

### Properties (`/api/properties`)

| Method | Endpoint       | Role   | Description                                                                                     |
| ------ | -------------- | ------ | ----------------------------------------------------------------------------------------------- |
| GET    | `/`            | Public | Browse properties (filter: search, categoryId, minPrice, maxPrice, status; paginated, sortable) |
| GET    | `/:id`         | Public | Get property details with landlord + category                                                   |
| GET    | `/:id/reviews` | Public | Get property reviews (paginated, includes `averageRating` in meta)                              |

### Landlord Properties (`/api/landlord/properties`)

| Method | Endpoint | Role     | Description                                                |
| ------ | -------- | -------- | ---------------------------------------------------------- |
| POST   | `/`      | LANDLORD | Create property listing                                    |
| GET    | `/`      | LANDLORD | List own properties (paginated, sortable)                  |
| GET    | `/:id`   | LANDLORD | Get own property details                                   |
| PUT    | `/:id`   | LANDLORD | Update own property (all fields optional, includes status) |
| DELETE | `/:id`   | LANDLORD | Delete own property (if no active requests)                |

### Requests (`/api/requests`)

| Method | Endpoint | Role            | Description                                             |
| ------ | -------- | --------------- | ------------------------------------------------------- |
| POST   | `/`      | TENANT          | Submit move-in request                                  |
| GET    | `/`      | TENANT          | List own requests (paginated, filter by status)         |
| GET    | `/:id`   | TENANT          | Get request details (owner only)                        |
| PATCH  | `/:id`   | TENANT/LANDLORD | Unified state transitions (role-based field validation) |

**State transitions by role:**
- TENANT: `MOVE_OUT_REQUESTED` (from `MOVED_IN`, requires `moveOutDate`)
- LANDLORD: `MOVE_IN_APPROVED`/`MOVE_IN_REJECTED` (from `MOVE_IN_REQUESTED`), `MOVE_OUT_APPROVED`/`MOVE_OUT_REJECTED` (from `MOVE_OUT_REQUESTED`, `damageAmount` only on approve, `rejectedReason` required on reject)

### Landlord Requests (`/api/landlord/requests`)

| Method | Endpoint | Role     | Description                                                    |
| ------ | -------- | -------- | -------------------------------------------------------------- |
| GET    | `/`      | LANDLORD | List requests for own properties (paginated, filter by status) |

### Payments (`/api/payments`)

| Method | Endpoint   | Role            | Description                                                                     |
| ------ | ---------- | --------------- | ------------------------------------------------------------------------------- |
| POST   | `/`        | TENANT/LANDLORD | Create Checkout Session (type: SECURITY_DEPOSIT, MONTHLY_RENT, MOVE_OUT_REFUND) |
| GET    | `/`        | Authenticated   | List own payments (paginated, filter: status, type, id)                         |
| GET    | `/:id`     | Owner           | Get payment details (includes `periodStart`/`periodEnd` for monthly rent)       |
| POST   | `/webhook` | Stripe          | Webhook handler (raw body, signature verified)                                  |

**Payment types & authorization:**
- `SECURITY_DEPOSIT`: TENANT, request status `MOVE_IN_APPROVED`, auto-amount
- `MONTHLY_RENT`: TENANT, request status `MOVED_IN`/`MOVE_OUT_REQUESTED`/`MOVE_OUT_APPROVED`, auto-amount, auto period (1st-last day of month), duplicate blocked
- `MOVE_OUT_REFUND`: LANDLORD, request status `MOVE_OUT_APPROVED`, auto-amount = securityDeposit - damageAmount

**Payment status values:** `PENDING`, `PAID`, `FAILED`, `REFUNDED`

### Reviews (`/api/reviews`)

| Method | Endpoint                  | Role   | Description                                                        |
| ------ | ------------------------- | ------ | ------------------------------------------------------------------ |
| POST   | `/`                       | TENANT | Create review (only after request status = `MOVED_OUT`)            |
| GET    | `/properties/:id/reviews` | Public | Get property reviews (paginated, includes `averageRating` in meta) |

### Admin (`/api/admin`)

| Method | Endpoint      | Role  | Description                                                                   |
| ------ | ------------- | ----- | ----------------------------------------------------------------------------- |
| GET    | `/stats`      | ADMIN | Platform statistics (users, properties, requests, payments, totalTransaction) |
| GET    | `/users`      | ADMIN | List users (paginated, filter: role, isBanned, search)                        |
| PATCH  | `/users/:id`  | ADMIN | Ban/unban user (`{ isBanned: boolean, banReason?: string }`)                  |
| GET    | `/properties` | ADMIN | All properties (filter: status, landlordId)                                   |
| GET    | `/requests`   | ADMIN | All requests (filter: status)                                                 |
| GET    | `/payments`   | ADMIN | All payments (filter: status, type)                                           |

### Cron (`/api/cron`)

| Method | Endpoint                     | Auth         | Description                                          |
| ------ | ---------------------------- | ------------ | ---------------------------------------------------- |
| GET    | `/cleanup-rejected-requests` | CRON_SECRET* | Delete `MOVE_IN_REJECTED` requests older than 7 days |

*CRON_SECRET optional in local dev (works without auth if unset). Required in production.
## Request Status Flow (8 States)

```
MOVE_IN_REQUESTED → MOVE_IN_APPROVED → MOVED_IN → MOVE_OUT_REQUESTED → MOVE_OUT_APPROVED → MOVED_OUT
↓                                                 ↓
↓ (reject)                                        ↓ (reject)
↓                                                 ↓
MOVE_IN_REJECTED                                  MOVE_OUT_REJECTED
```

- **MOVED_IN / MOVED_OUT**: Internal only, triggered by payment webhooks
- **Move-out window**: Request 1st-10th, move-out date 11th-last day of current month

---

## Payment Flow (Stripe Checkout)

1. Tenant creates request → `MOVE_IN_REQUESTED`
2. Landlord approves → `MOVE_IN_APPROVED`
3. Tenant calls `POST /payments` with `{ requestId, type: "SECURITY_DEPOSIT" }`
4. Returns `checkoutUrl` → tenant pays on Stripe hosted page
5. **Stripe webhook** (`checkout.session.completed`) fires:
   - Payment → `PAID`
   - Request → `MOVED_IN`
   - Property → `RENTED`
6. **Move-out**: Landlord approves → `MOVE_OUT_APPROVED`
7. Landlord pays `MOVE_OUT_REFUND` → webhook:
   - Payment → `REFUNDED`
   - Request → `MOVED_OUT`
   - Property → `AVAILABLE`

**Test Card**: `4242 4242 4242 4242` (any future date, any CVC)

---

## Local Webhook Testing

```bash
stripe login
stripe listen --forward-to localhost:5000/api/payments/webhook
# Copy whsec_... to .env as STRIPE_WEBHOOK_SECRET
```

---

## Test Credentials (Seeded)

Admin credentials loaded from config (`ADMIN_EMAIL`, `ADMIN_PASSWORD`). Landlord and tenant credentials are hardcoded in seed.

| Role     | Email                   | Password                   |
| -------- | ----------------------- | -------------------------- |
| Admin    | `ADMIN_EMAIL` from .env | `ADMIN_PASSWORD` from .env |
| Landlord | `landlord01@email.com`  | `1a2s3d4f`                 |
| Tenant   | `tenant01@email.com`    | `1a2s3d4f`                 |
| Tenant   | `tenant02@email.com`    | `1a2s3d4f`                 |
| Tenant   | `tenant03@email.com`    | `1a2s3d4f`                 |
| Tenant   | `tenant04@email.com`    | `1a2s3d4f`                 |
| Tenant   | `tenant05@email.com`    | `1a2s3d4f`                 |

---

## Cron Cleanup (Auto-delete rejected move-in requests)

A scheduled job deletes `MOVE_IN_REJECTED` requests older than 7 days.

**Local development:** Runs via `node-cron` daily at 2:00 AM (started in `server.ts`). 
Endpoint works without auth if `CRON_SECRET` is not set in `.env`. If set, requires `Authorization: Bearer <CRON_SECRET>`.

**Vercel production:** Runs via Vercel Cron daily at 2:00 AM, hitting `GET /api/cron/cleanup-rejected-requests`.
Requires `CRON_SECRET` set in Vercel environment variables; Vercel Cron automatically sends the auth header.

**Manual trigger (for testing):**
```bash
# Without CRON_SECRET (local only)
curl http://localhost:5000/api/cron/cleanup-rejected-requests

# With CRON_SECRET (local or production)
curl -H "Authorization: Bearer YOUR_CRON_SECRET" \
  https://your-domain.vercel.app/api/cron/cleanup-rejected-requests
```

## API Documentation

See **[POSTMAN_GUIDE.md](./postman/POSTMAN_GUIDE.md)** for:
- Importing the Postman collection (`ph-se-026.postman_collection.json`)
- Setting up environments
- Authentication flow
- Testing all endpoints
- Payment flow walkthrough

---

## Deployment (Vercel CLI)

```bash
# Install Vercel CLI
npm i -g vercel

# Login
vc login

# Deploy from project root
vc --prod
```

**Vercel config** (`vercel.json`) included for serverless function handling. \
**Environment variables** must be added in Vercel Dashboard → Project → Settings → Environment Variables.