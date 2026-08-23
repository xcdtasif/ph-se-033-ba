# Postman Collection Guide

## Collection File
`ph-se-026.postman_collection.json` — Import via File → Import.

## Base URLs

| Environment         | Base URL                         |
| ------------------- | -------------------------------- |
| Local Development   | http://localhost:5000/api        |
| Production (Vercel) | https://ph-se-026.vercel.app/api |

Set `baseUrl` as a **collection variable** in Postman.

---

## Authentication Flow

All protected endpoints require JWT access token in httpOnly cookie.

### 1. Register / Login
- POST /auth/register — Create account (TENANT, LANDLORD only; ADMIN forbidden)
- POST /auth/login — Returns access + refresh tokens in cookies

**Test credentials (seeded users):**

| Role     | Email                | Password |
| -------- | -------------------- | -------- |
| ADMIN    | ***                  | ***      |
| LANDLORD | landlord01@email.com | 1a2s3d4f |
| TENANT   | tenant01@email.com   | 1a2s3d4f |
| TENANT   | tenant02@email.com   | 1a2s3d4f |
| TENANT   | tenant03@email.com   | 1a2s3d4f |
| TENANT   | tenant04@email.com   | 1a2s3d4f |
| TENANT   | tenant05@email.com   | 1a2s3d4f |

*Admin credentials will be provided in assignment submission.*

### 2. Cookie Handling
Postman auto-stores cookies per domain. After login, subsequent requests send cookies automatically.

- Enable Cookies: Settings → General → Cookie jar
- Or manually copy accessToken from login → set Cookie header: accessToken=<value>; refreshToken=<value>

### 3. Refresh Token
- POST /auth/refresh — Uses refreshToken cookie to issue new access token

---

## Auth Module

### Register User [PUBLIC]
POST {{baseUrl}}/auth/register
```json
{
  "email": "user@test.com",
  "password": "1a2s3d4f",
  "role": "TENANT|LANDLORD|ADMIN",
  "name": "Test User",
  "phone": "01700000000"
}
```
- TENANT → 201 Created
- LANDLORD → 201 Created
- ADMIN → 403 Forbidden

---

### Login User [PUBLIC]
POST {{baseUrl}}/auth/login
```json
{
  "email": "user@email.com",
  "password": "1a2s3d4f"
}
```
- Valid → 200 OK (sets cookies)
- Invalid → 401 Unauthorized

---

### Refresh Tokens [TENANT | LANDLORD | ADMIN]
POST {{baseUrl}}/auth/refresh
- Valid refreshToken → 200 OK (new accessToken)
- Missing/expired → 401 Unauthorized

---

### Get My Profile [TENANT | LANDLORD | ADMIN]
GET {{baseUrl}}/auth/me
- Valid cookie → 200 OK (returns user)
- Missing/expired → 401 Unauthorized

---

### Logout User [TENANT | LANDLORD | ADMIN]
POST {{baseUrl}}/auth/logout
- Always → 200 OK (clears cookies)

---

## Category Module

### Get Categories [PUBLIC]
GET {{baseUrl}}/categories?page=1&limit=10&search=
- Public → 200 OK (paginated with meta)
- Query: page, limit (max 100), search

---

### Get Category [PUBLIC]
GET {{baseUrl}}/categories/:id
- Public → 200 OK or 404 Not Found

---

### Create Category [ADMIN]
POST {{baseUrl}}/categories
```json
{
  "name": "Mansion",
  "description": "Large luxury residential estates"
}
```
- Admin → 201 Created
- Non-admin → 403 Forbidden
- No auth → 401 Unauthorized

---

### Update Category [ADMIN]
PUT {{baseUrl}}/categories/:id
```json
{
  "name": "Luxury Mansion",
  "description": "Premium luxury estates"
}
```
- Admin → 200 OK
- Non-admin → 403 Forbidden
- No auth → 401 Unauthorized

---

### Delete Category [ADMIN]
DELETE {{baseUrl}}/categories/:id
- Admin + no properties → 200 OK
- Admin + has properties → 400 Bad Request
- Non-admin → 403 Forbidden
- No auth → 401 Unauthorized

---

## Property Module

### Get Properties [PUBLIC]
GET {{baseUrl}}/properties?page=1&limit=10&search=&categoryId=&minPrice=&maxPrice=&status=AVAILABLE&sortBy=createdAt&sortOrder=desc
- Public → 200 OK (paginated with meta)
- Query: page, limit (max 100), search, categoryId, minPrice, maxPrice, status (AVAILABLE|RENTED|UNAVAILABLE), sortBy (monthlyRent|createdAt), sortOrder (asc|desc)

---

### Get Property [PUBLIC]
GET {{baseUrl}}/properties/:id
- Public → 200 OK or 404 Not Found

---

### Get Property Reviews [PUBLIC]
GET {{baseUrl}}/properties/:id/reviews?page=1&limit=10
- Public → 200 OK (paginated with averageRating in meta)
- Query: page, limit (max 100)

---

## Request Module

### Create MoveIn Request [TENANT]
POST {{baseUrl}}/requests
```json
{
  "propertyId": "0a3456aa-27a8-422f-8030-c8f6362b4341",
  "moveInDate": "2026-08-15",
  "message": "Interested in this property"
}
```
- TENANT + valid AVAILABLE property → 201 Created
- Property not AVAILABLE → 400 Bad Request
- Duplicate request → 400 Bad Request

---

### Get Tenant Requests [TENANT]
GET {{baseUrl}}/requests?page=1&limit=10&status=MOVE_IN_REQUESTED
- TENANT → 200 OK (paginated with meta)
- Query: page, limit (max 100), status

---

### Get Tenant Request [TENANT]
GET {{baseUrl}}/requests/:id
- TENANT (owner) → 200 OK
- Not owner → 403 Forbidden

---

### Approve MoveIn Request [LANDLORD]
PATCH {{baseUrl}}/requests/:id
```json
{
  "status": "MOVE_IN_APPROVED"
}
```

---

### Reject MoveIn Request [LANDLORD]
PATCH {{baseUrl}}/requests/:id
```json
{
  "status": "MOVE_IN_REJECTED",
  "rejectedReason": "Not suitable"
}
```

---

### Request MoveOut [TENANT]
PATCH {{baseUrl}}/requests/:id
```json
{
  "status": "MOVE_OUT_REQUESTED",
  "moveOutDate": "2026-08-30"
}
```

---

### Approve MoveOut Request [LANDLORD]
PATCH {{baseUrl}}/requests/:id
```json
{
  "status": "MOVE_OUT_APPROVED",
  "damageAmount": 5000
}
```

---

### Reject MoveOut Request [LANDLORD]
PATCH {{baseUrl}}/requests/:id
```json
{
  "status": "MOVE_OUT_REJECTED",
  "rejectedReason": "Damage exceeds deposit"
}
```

---

**State transitions by role:**
- TENANT: MOVE_OUT_REQUESTED (from MOVED_IN, requires moveOutDate)
- LANDLORD: MOVE_IN_APPROVED/MOVE_IN_REJECTED (from MOVE_IN_REQUESTED), MOVE_OUT_APPROVED/MOVE_OUT_REJECTED (from MOVE_OUT_REQUESTED, damageAmount only on approve, rejectedReason required on reject)

**Field validation per role/transition:**
- Tenant: moveOutDate required for MOVE_OUT_REQUESTED; cannot send damageAmount or rejectedReason
- Landlord: rejectedReason required for rejections; damageAmount only for MOVE_OUT_APPROVED; cannot send moveOutDate for move-in decisions

---

## Landlord Module

### Create Property [LANDLORD]
POST {{baseUrl}}/landlord/properties
```json
{
  "title": "Test Apartment",
  "description": "Nice place in Dhaka",
  "location": "Dhaka",
  "mapLocation": "https://maps.google.com/?q=...",
  "monthlyRent": 25000,
  "securityDeposit": 50000,
  "images": ["https://example.com/img1.jpg"],
  "categoryId": "<category-id-from-GET-/categories>"
}
```
- LANDLORD → 201 Created (status: AVAILABLE)
- Invalid category → 400 Bad Request

---

### Get My Properties [LANDLORD]
GET {{baseUrl}}/landlord/properties?page=1&limit=10&sortBy=createdAt&sortOrder=desc
- LANDLORD → 200 OK (paginated with meta)
- Query: page, limit (max 100), sortBy (monthlyRent|createdAt), sortOrder (asc|desc)

---

### Get My Property [LANDLORD]
GET {{baseUrl}}/landlord/properties/:id
- LANDLORD (owner) → 200 OK
- Not owner → 403 Forbidden

---

### Update Property [LANDLORD]
PUT {{baseUrl}}/landlord/properties/:id
```json
{
  "title": "Updated Apartment",
  "monthlyRent": 27000,
  "securityDeposit": 60000,
  "status": "UNAVAILABLE"
}
```
- LANDLORD (owner) → 200 OK
- Valid fields: title, description, location, mapLocation, monthlyRent, securityDeposit, images, categoryId, status (all optional)
- Cannot update if active requests exist (MOVE_IN_REQUESTED|MOVE_IN_APPROVED|MOVED_IN) → 400 Bad Request
- Not owner → 403 Forbidden

---

### Delete Property [LANDLORD]
DELETE {{baseUrl}}/landlord/properties/:id
- LANDLORD (owner) + no active requests → 200 OK
- Active requests exist → 400 Bad Request
- Not owner → 403 Forbidden

---

### Get Landlord Requests [LANDLORD]
GET {{baseUrl}}/landlord/requests?page=1&limit=10
- LANDLORD → 200 OK (paginated with meta)
- Query: page, limit (max 100), status

---

## Payment Module

### Create Checkout [TENANT | LANDLORD]
POST {{baseUrl}}/payments
```json
{
  "requestId": "<request-id>",
  "type": "SECURITY_DEPOSIT|MONTHLY_RENT|MOVE_OUT_REFUND"
}
```

| Type             | Role     | Required Request Status                         | Notes                                                                                                                |
| ---------------- | -------- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| SECURITY_DEPOSIT | TENANT   | MOVE_IN_APPROVED                                | Auto-amount from property.securityDeposit                                                                            |
| MONTHLY_RENT     | TENANT   | MOVED_IN, MOVE_OUT_REQUESTED, MOVE_OUT_APPROVED | Auto-amount from property.monthlyRent; auto period (1st-last day of current month); duplicate for same month blocked |
| MOVE_OUT_REFUND  | LANDLORD | MOVE_OUT_APPROVED                               | Auto-amount = securityDeposit - damageAmount                                                                         |

- Invalid status for type → 400 Bad Request
- Wrong role for type → 403 Forbidden
- Monthly rent for same period already paid → 400 Bad Request

---

### Get My Payments [TENANT | LANDLORD]
GET {{baseUrl}}/payments?page=1&limit=10&status=PAID&type=MONTHLY_RENT
- Authenticated user → 200 OK (paginated with meta)
- Query: page, limit (max 100), status (PENDING|PAID|FAILED|REFUNDED), type, id

---

### Get My Payment [TENANT | LANDLORD]
GET {{baseUrl}}/payments/:id
- Owner → 200 OK
- Not owner → 403 Forbidden
- Response includes periodStart, periodEnd (only for MONTHLY_RENT)

---

### Stripe Webhook [STRIPE]
POST {{baseUrl}}/payments/webhook
- Stripe signature verification required (stripe-signature header)
- Raw body (express.raw)
- Handles: checkout.session.completed
- Local testing: `stripe listen --forward-to localhost:5000/api/payments/webhook`

---

## Review Module

### Create Review [TENANT]
POST {{baseUrl}}/reviews
```json
{
  "propertyId": "<property-id>",
  "requestId": "<request-id>",
  "rating": 5,
  "comment": "Optional comment"
}
```
- TENANT + request status MOVED_OUT + owns request → 201 Created
- Request not MOVED_OUT → 400 Bad Request
- Already reviewed → 400 Bad Request
- Not request owner → 403 Forbidden

---

### Get Property Reviews [PUBLIC]
GET {{baseUrl}}/properties/:id/reviews?page=1&limit=10
- Public → 200 OK (paginated with averageRating in meta)
- Query: page, limit (max 100)

---

## Admin Module

### Get Users [ADMIN]
GET {{baseUrl}}/admin/users?page=1&limit=10&search=&role=&isBanned=
- Admin → 200 OK (paginated with meta)
- Query: page, limit (max 100), search, role (TENANT|LANDLORD|ADMIN), isBanned (true|false)

---

### Ban/Unban User [ADMIN]
PATCH {{baseUrl}}/admin/users/:id
```json
{
  "isBanned": true,
  "banReason": "Optional reason"
}
```
- Admin → 200 OK
- Non-admin → 403 Forbidden

---

### Get Stats [ADMIN]
GET {{baseUrl}}/admin/stats
- Admin → 200 OK (users, properties, requests, payments counts)

---

### Get Properties [ADMIN]
GET {{baseUrl}}/admin/properties?page=1&limit=10&search=&status=&categoryId=
- Admin → 200 OK (paginated with meta)

---

### Get Requests [ADMIN]
GET {{baseUrl}}/admin/requests?page=1&limit=10&status=
- Admin → 200 OK (paginated with meta)
- Query: page, limit (max 100), status

---

### Get Payments [ADMIN]
GET {{baseUrl}}/admin/payments?page=1&limit=10&status=&type=
- Admin → 200 OK (paginated with meta)
- Query: page, limit (max 100), status (PENDING|PAID|FAILED|REFUNDED), type

---

## Cron Endpoint

### Cleanup Old Rejected Requests [PUBLIC* | ADMIN*]
GET {{baseUrl}}/cron/cleanup-rejected-requests
Header: Authorization: Bearer <CRON_SECRET> (required only if CRON_SECRET is set)

**Environment Variable:**
- Local: Optional — add `CRON_SECRET=your-secret-here` to `.env` to enable auth; if unset, endpoint works without auth
- Production (Vercel): Required — add `CRON_SECRET` in Vercel Dashboard → Settings → Environment Variables

- Removes `MOVE_IN_REJECTED` requests older than **7 days**
- Returns: `{ success: true, message: "Deleted X old MOVE_IN_REJECTED requests", deletedCount: X }`

**Local Testing (without CRON_SECRET):**
```bash
curl http://localhost:5000/api/cron/cleanup-rejected-requests
```

**Local Testing (with CRON_SECRET set):**
```bash
curl -H "Authorization: Bearer YOUR_CRON_SECRET" \
  http://localhost:5000/api/cron/cleanup-rejected-requests
```

**Production (Vercel Cron):**
- Configured in `vercel.json` to run daily at 2:00 AM UTC
- Endpoint: `GET /api/cron/cleanup-rejected-requests`
- Automatically includes `Authorization: Bearer <CRON_SECRET>` header
- CRON_SECRET must be set in Vercel environment variables

---

## Role-Based Access Summary

| Endpoint                            | Public | TENANT | LANDLORD | ADMIN  |
| ----------------------------------- | ------ | ------ | -------- | ------ |
| POST /auth/register                 | ✅ 201  | ✅ 201  | ✅ 201    | ❌ 403  |
| POST /auth/login                    | ✅ 200  | ✅ 200  | ✅ 200    | ✅ 200  |
| GET /auth/me                        | ❌ 401  | ✅ 200  | ✅ 200    | ✅ 200  |
| POST /auth/refresh                  | ❌ 401  | ✅ 200  | ✅ 200    | ✅ 200  |
| POST /auth/logout                   | ✅ 200  | ✅ 200  | ✅ 200    | ✅ 200  |
| GET /categories                     | ✅ 200  | ✅ 200  | ✅ 200    | ✅ 200  |
| GET /categories/:id                 | ✅ 200  | ✅ 200  | ✅ 200    | ✅ 200  |
| POST /categories                    | ❌ 401  | ❌ 403  | ❌ 403    | ✅ 201  |
| PUT /categories/:id                 | ❌ 401  | ❌ 403  | ❌ 403    | ✅ 200  |
| DELETE /categories/:id              | ❌ 401  | ❌ 403  | ❌ 403    | ✅ 200  |
| GET /properties                     | ✅ 200  | ✅ 200  | ✅ 200    | ✅ 200  |
| GET /properties/:id                 | ✅ 200  | ✅ 200  | ✅ 200    | ✅ 200  |
| GET /properties/:id/reviews         | ✅ 200  | ✅ 200  | ✅ 200    | ✅ 200  |
| POST /requests                      | ❌ 401  | ✅ 201  | ❌ 403    | ❌ 403  |
| GET /requests                       | ❌ 401  | ✅ 200  | ❌ 403    | ❌ 403  |
| GET /requests/:id                   | ❌ 401  | ✅ 200  | ❌ 403    | ❌ 403  |
| PATCH /requests/:id                 | ❌ 401  | ✅ 200* | ✅ 200*   | ❌ 403  |
| POST /payments                      | ❌ 401  | ✅ 201* | ✅ 201*   | ❌ 403  |
| GET /payments                       | ❌ 401  | ✅ 200  | ✅ 200    | ✅ 200  |
| GET /payments/:id                   | ❌ 401  | ✅ 200  | ✅ 200    | ✅ 200  |
| POST /reviews                       | ❌ 401  | ✅ 201  | ❌ 403    | ❌ 403  |
| GET /landlord/properties            | ❌ 401  | ❌ 403  | ✅ 200    | ❌ 403  |
| POST /landlord/properties           | ❌ 401  | ❌ 403  | ✅ 201    | ❌ 403  |
| PUT /landlord/properties/:id        | ❌ 401  | ❌ 403  | ✅ 200    | ❌ 403  |
| DELETE /landlord/properties/:id     | ❌ 401  | ❌ 403  | ✅ 200    | ❌ 403  |
| GET /landlord/requests              | ❌ 401  | ❌ 403  | ✅ 200    | ❌ 403  |
| GET /admin/users                    | ❌ 401  | ❌ 403  | ❌ 403    | ✅ 200  |
| PATCH /admin/users/:id              | ❌ 401  | ❌ 403  | ❌ 403    | ✅ 200  |
| GET /admin/stats                    | ❌ 401  | ❌ 403  | ❌ 403    | ✅ 200  |
| GET /admin/properties               | ❌ 401  | ❌ 403  | ❌ 403    | ✅ 200  |
| GET /admin/requests                 | ❌ 401  | ❌ 403  | ❌ 403    | ✅ 200  |
| GET /admin/payments                 | ❌ 401  | ❌ 403  | ❌ 403    | ✅ 200  |
| GET /cron/cleanup-rejected-requests | ✅ 200* | ❌ 403  | ❌ 403    | ✅ 200* |

*Public/ADMIN access to cron endpoint only when CRON_SECRET is not set. When CRON_SECRET is set, requires Authorization: Bearer <CRON_SECRET> header.
*Role-specific state transitions / payment types apply.