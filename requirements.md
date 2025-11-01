**Airbnb Clone - Backend Feature Specifications**

Version: 1.0  
Date: 2025-11-01
Auther: Kaleab Ayele  

**1) User Authentication & Authorization**

**Purpose**

Manage user identity, account lifecycle, secure access tokens, password resets, and optional MFA. Support roles: guest, host, admin.

**Primary data model (summary)**

User {

id: UUID,

name: string,

email: string (unique),

phone: string (optional),

password_hash: string,

role: enum('guest','host','admin'),

is_email_verified: boolean,

created_at, updated_at

}

Session / Token store (redis): refresh_token -> user_id, expires_at

### Security

- Password hashing: bcrypt (cost >= 12) or argon2 (recommended).
- All endpoints must use HTTPS (TLS 1.2+).
- OWASP best practices: input sanitization, SQL parameterization, CSP for frontend, CSP not in backend but recommend headers.
- Rate limit: authentication endpoints (e.g., 10 requests/min per IP for login, 5/min for password-reset).
- JWT signing: RS256 (asymmetric) preferred, keep private key secure; short-lived access tokens (e.g., 15m), refresh tokens long-lived (e.g., 30d) and stored server-side for revocation.
- Multi-factor authentication (optional): TOTP or SMS provider. Store MFA secret encrypted.

### API Endpoints

#### Register

- **POST** /api/v1/auth/register
- **Auth**: none
- **Request**

{

"name": "Alice Greer",

"email": "<alice@example.com>",

"password": "P@ssw0rd!23",

"phone": "+15551234567",

"role": "host" // optional: default "guest"

}

 **Validation**

- email: valid RFC 5322-ish regex, unique
- password: min 10 chars, at least 1 uppercase, 1 lowercase, 1 digit, 1 special char (or company policy)
- name: non-empty, max 100 chars

 **Response**

- 201 Created with body:

{ "id": "uuid", "email": "<alice@example.com>", "is_email_verified": false }

- **Post-actions**
  - Send email verification with signed token (short expiry, e.g., 24h).
  - Save user record with password_hash.

#### Email verification

- **GET** /api/v1/auth/verify-email?token=&lt;token&gt;
- **Behavior**: verify token, set is_email_verified=true, redirect to configured URL.

#### Login

- **POST** /api/v1/auth/login
- **Request**

{ "email": "<alice@example.com>", "password": "P@ssw0rd!23", "device_info": "web-chrome" }

 **Validation**

- Rate limit per IP and per account.

 **Response**

- 200 OK

{

"access_token": "&lt;jwt&gt;",

"access_token_expires_in": 900,

"refresh_token": "&lt;opaque-token&gt;",

"refresh_token_expires_in": 2592000,

"user": { "id": "uuid", "email": "<alice@example.com>", "role": "host" }

}

- **Notes**
  - Store refresh token in secure DB or Redis with expiry and allow revocation.
  - Include jti in JWT to enable token revocation checks.

#### Refresh access token

- **POST** /api/v1/auth/refresh
- **Request**

{ "refresh_token": "&lt;opaque-token&gt;" }

- **Response**
  - 200 OK new access_token & optionally new refresh_token rotating refresh tokens.

#### Logout / Revoke

- **POST** /api/v1/auth/logout
- **Auth**: Bearer token
- **Request** { "refresh_token": "&lt;token&gt;" }
- **Action**: mark refresh token revoked, optionally blacklist current access token jti.

#### Password reset

- **POST** /api/v1/auth/password-reset/request
  - input: { "email": "<alice@example.com>" } -> send reset email with token.
- **POST** /api/v1/auth/password-reset/confirm
  - input: { "token": "...", "new_password": "..." } -> validate token, validate password rules, update password_hash, revoke tokens.

### Validation & error responses

- Use consistent error schema:

{ "error": { "code": "INVALID_INPUT", "message": "Password too weak", "fields": {"password":"min_length"} } }

- HTTP codes: 400 (validation), 401 (auth failed), 403 (forbidden), 404, 429 (rate), 500 (server).

**Performance & SLOs**

- Read operations (GET user profile): p95 latency ≤ 200 ms
- Auth login (write + hash): p95 latency ≤ 500 ms
- Auth throughput: scale to 2000 logins/min per region; recommend autoscaling and caching (Redis) for sessions.
- Availability: 99.95% SLA for auth service.

**Monitoring & Logging**

- Log failed login attempts, password reset requests, token refresh events.
- Track suspicious login patterns, integrate with anomaly detection.

**2) Property Management**

**Purpose**

Enable hosts to create, update, manage property listings, upload photos, set availability and pricing, and expose property search APIs for guests.

**Data model (summary)**

Property {

id: UUID,

host_id: UUID (FK->User.id),

title: string,

description: text,

address: string,

lat: decimal, lon: decimal,

price_per_night: decimal,

currency: string,

max_guests: integer,

amenities: \[string\],

created_at, updated_at,

status: enum('draft','published','pending_approval','suspended')

}

PropertyPhoto { id, property_id, file_url, order }

Availability { property_id, date, is_available, min_stay, price_override }

### Functional capabilities

- CRUD on listings (host).
- Upload images to file storage (S3/GCS) (signed URLs).
- Availability calendar management (per-date).
- Search by geo-radius, price-range, dates (availability), amenities.
- Admin review and approval pipeline.

### API Endpoints & Specifications

#### Create property (Host)

- **POST** /api/v1/properties
- **Auth**: Bearer (role: host)
- **Request**

{

"title":"Sunny 2BR near beach",

"description":"...",

"address":"123 Ocean Ave, City",

"lat": -12.345,

"lon": 45.678,

"price_per_night": 120.50,

"currency":"USD",

"max_guests": 4,

"amenities": \["wifi","kitchen","ac"\]

}

- **Validation**
  - title: required, 5-200 chars
  - price_per_night: numeric, >= 0.0
  - max_guests: integer >= 1
  - coordinates: valid decimal range
- **Response**
  - 201 Created { "id": "uuid", "status": "draft" }
- **Notes**
  - Optionally set initial status: pending_approval if admin approval required.

#### Upload photos

- **POST** /api/v1/properties/{propertyId}/photos
- **Auth**: host
- **Flow**
  - API returns signed upload URL to S3 (PUT) and a callback endpoint to confirm upload.
- **Response**
  - 201 { "photo_id": "uuid", "file_url": "<https://s3/>..." }

#### Update property

- **PUT** /api/v1/properties/{id}
- **Auth**: host (owner) or admin
- **Validation**: same as create
- **Response**: 200 OK updated resource

#### Search properties

- **GET** /api/v1/properties/search?lat=..&lon=..&radius_km=5&checkin=YYYY-MM-DD&checkout=YYYY-MM-DD&min_price=&max_price=&amenities=wifi,kitchen
- **Response**

{

"total": 234,

"items": \[

{ "id":"...", "title":"...", "price_per_night":120, "lat":..,"lon":.., "thumbnail":"..." }

\],

"page":1, "per_page":20

}

- **Performance**: geo-index (PostGIS or DB indexes), cached results for common queries in Redis (TTL 30s-2m). P95 <= 200ms for cached; p95 <= 500ms for fresh.

#### Availability endpoints

- **GET** /api/v1/properties/{id}/availability?start=YYYY-MM-DD&end=YYYY-MM-DD
- **PUT** /api/v1/properties/{id}/availability (bulk update, host)
  - body: list of {date,is_available,price_override}

### Validation & business rules

- Property status must be published to appear in searches.
- Limits on number of photos (e.g., max 50).
- Address normalization optional (geocoding as background job).
- Hosts cannot publish if missing required fields or photos (configurable).

### Concurrency and consistency

- Use optimistic locking on property updates (updated_at or version).
- Availability updates should be atomic for the date ranges (use transactions).

### Performance & SLOs

- Read-heavy: scale read replicas for property reads.
- Expect search throughput: target 2000 req/s in peak (scale horizontally).
- Photo uploads: asynchronous confirmation via signed URLs; S3 handles scalability.
- Database indexing: geo index on (lat,lon), index on host_id, status, price.

### Security & privacy

- Only owner and admin can modify/delete listing.
- Sanitize HTML in descriptions to prevent XSS.
- Signed URLs for direct uploads; remove EXIF geo if privacy required.

### Audit & admin

- Keep audit log: creation/update/delete with user_id, timestamp.
- Admin endpoints for approval/rejection and suspending listings.

## 3) Booking & Reservation System

### Purpose

Enable searching availability, creating bookings, processing payments (through payment gateway), cancellations, and ensuring no double-bookings.

### Data model (summary)

Booking {

id: UUID,

user_id: UUID (guest),

property_id: UUID,

start_date: date,

end_date: date,

guests: integer,

total_price: decimal,

currency: string,

status: enum('pending','confirmed','cancelled','completed','failed'),

created_at, updated_at,

payment_id: UUID (nullable)

}

Calendar (or Availability store): property_id, date, booking_id (nullable), is_available

Payment { id, booking_id, amount, currency, status, gateway_ref, created_at }

### Business rules

- Booking creation must check availability for all dates in requested range.
- Hold/Reserve: when creating booking, dates should be **locked or reserved** until payment confirmation - prevent race conditions.
- Cancellation policy: define flexible/strict policy; compute refund window and amounts.
- Commission fee: platform % deducted at payout time.

### Concurrency & consistency (critical)

- **Strong requirement**: avoid double-booking.
  - Approaches:
    - **Transactional row-level locking**: lock calendar rows for the date range in a DB transaction (FOR UPDATE). If any row is already booked, fail.
    - **Optimistic concurrency**: use versioning and retry if conflict detected (more complex).
    - **Distributed lock**: use Redis RedLock for cross-app instances to lock property_id for booking window.
  - Use database transactions spanning availability check + insert booking + mark calendar rows = all-or-nothing.

### API Endpoints

#### Create booking (start of flow)

- **POST** /api/v1/bookings
- **Auth**: Bearer (guest)
- **Request**

{

"property_id": "uuid",

"start_date": "2025-12-01",

"end_date": "2025-12-05",

"guests": 2,

"payment_method_id": "pm_abc", // tokenized/opaque from client

"idempotency_key": "uuid-or-random" // recommended

}

- **Flow**
  - Validate input & user.
  - Verify dates (start < end, min/max stay).
  - Check property status == published.
  - Check availability for each date.
  - Begin DB transaction:
    - create booking with status: pending
    - mark calendar dates as reserved (or create calendar entries pointing to booking)
  - Call Payment Service (or create payment intent at gateway).
    - Use external payment provider (Stripe/PayPal). Use idempotency_key on both service and gateway.
  - On successful charge: set booking status: confirmed, attach payment_id.
    - Notify host and guest via Notification Service.
  - On payment failure: rollback or set booking status: failed and release reserved dates.
- **Response**
  - 201 Created { "id":"uuid", "status":"pending", "amount": 480.00, "currency":"USD" }
  - Final confirmed result returned after payment or via webhook.
- **Validation**
  - Booking length limits, guest count <= property.max_guests
  - Idempotency key required for clients creating bookings to avoid duplicate charges

#### Get booking

- **GET** /api/v1/bookings/{id}
- **Auth**: user is owner (guest) or host of property or admin
- **Response**

{ "id":"...", "property_id":"...", "status":"confirmed", "start_date":"...", "end_date":"...", "total_price": 480 }

#### Cancel booking

- **POST** /api/v1/bookings/{id}/cancel
- **Auth**: guest or host depending on policy
- **Flow**
  - Check cancellation window policy.
  - If refund required, issue refund via payment gateway (async webhook).
  - Release calendar dates.
  - Update booking status to cancelled.

#### Host confirm / manage booking

- **POST** /api/v1/bookings/{id}/confirm (if host approval required)
- Host can message guest or approve/decline.

#### Webhook / Payment confirmation

- **POST** /api/v1/webhooks/payment
- Accept events from payment gateway (charge.succeeded, payment_intent.succeeded, refund.succeeded) with signature verification.
- On success, mark payment and booking accordingly.

### Payment integration notes

- Do not store card numbers. Use gateway tokenization (PCI compliance).
- Use payment_intent/authorize-capture patterns if you want to authorize first then capture on checkin.
- Keep audit trail of payment.gateway_ref, status, timestamps.

### Validation & errors

- Standard error schema as above.
- Handle expired cards, 3DS required flows: respond to client to complete 3DS.

### Concurrency & throughput

- Booking throughput under heavy load requires:
  - optimistic short locks or partitioning by property_id to avoid contending locks.
  - Suggest queueing non-critical tasks (notifications, review updates) into background workers.
- SLOs:
  - Availability checks (read) p95 <= 150ms.
  - Booking creation p95 <= 700ms (includes payment); if payment external slower, respond with pending status and complete via webhook.
  - System should support at least 200 concurrent booking attempts across properties without failing.

### Idempotency & retries

- Require idempotency key for booking creation and use it server-side (store last request result for given key).
- Retry safely on transient gateway errors; do not duplicate confirmed bookings.

### Cancellation & refunds

- Refund logic may be asynchronous. Provide endpoint to check refund status.
- Track refund state in Payment entity.

### Data retention & audit

- Keep booking history; soft delete for compliance.
- Store audit logs for booking creation/changes with actor and timestamp.

### Notifications & outputs

- On booking state changes, emit events for Notification Service: email, push, SMS.
- Provide webhooks to external systems (e.g., host PMS) if required.

## Cross-Feature Non-Functional Requirements

### API conventions

- Versioned endpoints (/api/v1/...).
- Use JSON for request/response. Use proper Content-Type and Accept.
- Use HATEOAS links optional; prefer simple REST with clear resources.
- Standard pagination (limit/offset or cursor) with per_page and page.

### Validation & error handling

- Consistent error format across services.
- Provide machine-friendly error codes AND human message.

### Observability

- Distributed tracing (OpenTelemetry), correlation ids on every request.
- Metrics: request counts, latency histograms, error rates, DB slow queries.
- Alerts: high error rates, DB CPU, payment gateway failures > threshold.

### Security

- TLS 1.2+, secure headers, protection against CSRF on session flows if any.
- Input validation to prevent SQL injection; use ORM prepared statements.
- Secrets in vault (DB credentials, payment keys).
- Role-based access control middleware on resource endpoints.

### Scalability & caching

- Use Redis for sessions, rate-limits, and caching search/availability results (short TTL).
- Use read replicas for heavy-read DB workloads.
- Background workers (RabbitMQ/Kafka) to process long-running ops (image processing, payout calculation, email sending).

### Backups & DR

- Regular DB backups, test restore.
- Payment and booking state replicated across regions if global service.

## Appendix - Example error schema

{

"error": {

"code": "BOOKING_DATE_CONFLICT",

"message": "Some dates are no longer available",

"details": { "conflicting_dates": \["2025-12-02", "2025-12-03"\] }

}

}