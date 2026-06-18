# UniRush — Software Design Document
*"Quick. Safe. Convenient."*

**Version:** 1.0 | **Date:** June 2026 | **Status:** Confidential Draft
**Stack:** React.js (Vite) + Express.js + MySQL
**Repos:** `supplytrack-backend` (refactored) · `supplytrack-frontend` (refactored)
**Team:** Emilly Sendikaddiwa · Matthias Imani · Eric Dickens · Ernest Ntare
**Institution:** Uganda Christian University — Group 17

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Overview](#2-system-overview)
3. [Refactoring Strategy](#3-refactoring-strategy)
4. [Database Design](#4-database-design)
5. [API Design](#5-api-design)
6. [Backend Architecture](#6-backend-architecture)
7. [Frontend Architecture](#7-frontend-architecture)
8. [Non-Functional Requirements](#8-non-functional-requirements)
9. [Security Design](#9-security-design)
10. [Platform Policies — Technical Enforcement](#10-platform-policies--technical-enforcement)
11. [Deployment & Environment](#11-deployment--environment)
12. [Testing Strategy](#12-testing-strategy)
13. [Open Issues & Decisions Pending](#13-open-issues--decisions-pending)
- [Appendix A: Glossary](#appendix-a-glossary)
- [Appendix B: References](#appendix-b-references)

---

## 1. Introduction

### 1.1 Purpose

This Software Design Document (SDD) defines the architecture, data model, API design, component structure, and implementation strategy for UniRush — a campus-focused delivery platform for Uganda Christian University. It serves as the authoritative technical reference for the development team throughout the refactoring and build process.

### 1.2 Scope

UniRush is a three-sided marketplace connecting Clients (students), Sellers (restaurants and shops), and Delivery Personnel (boda riders, car owners, and student agents). The system supports food delivery, package delivery, grocery runs, academic errands, and on-campus passenger transport.

This document covers the refactoring of two existing repositories:
- **supplytrack-backend** — an Express.js + MySQL API originally built for a supply tracking use case.
- **supplytrack-frontend** — a React.js (Vite) SPA with role-based routing for customer, supplier, station officer, and admin roles.

Both repositories retain their core architecture and are extended to serve the UniRush domain model.

### 1.3 Team

| Name | Role(s) |
|---|---|
| Emilly Sendikaddiwa | Quality Assurance & Front End Developer |
| Matthias Imani | Scrum Master & Back End Developer |
| Eric Dickens | Front End Developer |
| Ernest Ntare | Documentation, User Experience & Back End Developer |

### 1.4 Definitions & Acronyms

| Term | Definition |
|---|---|
| Client | A student or campus resident who places orders on the platform. |
| Seller / Vendor | A restaurant or shop registered to sell goods through UniRush. |
| Boda Rider | A motorcycle courier who fulfils delivery jobs. |
| Student Agent | An enrolled student who has activated a Rider or Driver role on their client account. |
| OTP | One-Time Password — 4-digit code used at delivery handoff to confirm receipt. |
| Escrow | Platform-held payment released only after OTP delivery confirmation. |
| Geofence | A GPS boundary; Student Agents without a permit are restricted to within 3 km of the university gate. |
| COD | Cash on Delivery — customer pays cash to the rider at the door. |
| MoMo | Mobile Money — MTN MoMo or Airtel Money, the primary payment method. |
| Dispatch Engine | Algorithm that matches riders to orders based on proximity and rating. |
| JWT | JSON Web Token used for stateless authentication. |
| RBAC | Role-Based Access Control — enforced by middleware on every protected route. |
| SPA | Single Page Application — the React frontend. |

---

## 2. System Overview

### 2.1 Platform Summary

UniRush is a web-first progressive application available at `unirush.ug` (client and rider) and `sellers.unirush.ug` (seller dashboard). The system is deployed as two independent Node.js processes: the Express.js REST API and the Vite-built React SPA. Both communicate over HTTPS.

### 2.2 User Roles

| Role | Who They Are | Key Identifier | Approval Time |
|---|---|---|---|
| Client | Student / campus resident placing orders | Student Reg. No. + Access No. | Minutes |
| Seller | Restaurant, shop, or supermarket | URA Tax ID (TIN) | 24–48 hours |
| Boda Rider | Professional motorcycle courier | NIN + Riding Permit | 24–48 hours |
| Car Owner | Professional driver for transport and bulk | NIN + Driving Permit + Logbook | 24–48 hours |
| Student Agent | Student with activated Rider or Driver role | Reg. No. + Vehicle plate | Minutes |
| Admin | Platform operator managing all system entities | Assigned internally | Instant |

### 2.3 Service Categories

| Category | Examples |
|---|---|
| Food & Drinks | Meals from campus cafeterias and nearby restaurants |
| Shopping | Grocery runs, stationery, personal items |
| Package Delivery | Documents, clothes, academic materials |
| Academic Errands | Printing, library book pickups, stationery |
| Transport | Boda passenger trips and car rides within campus zone |

### 2.4 High-Level Architecture

UniRush follows a layered three-tier architecture:

- **Presentation Tier** — React.js SPA (Vite build, served via CDN or static host). Role-based routing enforced client-side; all sensitive operations re-validated server-side.
- **Application Tier** — Express.js REST API. Routes are protected by JWT authentication middleware and RBAC role middleware. Business logic resides in controllers. All async errors are caught by a shared `asyncHandler` wrapper.
- **Data Tier** — MySQL 8 relational database accessed via the `mysql2` connection pool. All write operations spanning multiple tables use transactions with explicit rollback on failure.

The frontend communicates with the backend exclusively via Axios over a versioned REST API (`/api/...`). No direct database access from the browser. File uploads (profile photos, documents, item images) are handled by Multer middleware and stored in the `/uploads` directory, served as static files.

---

## 3. Refactoring Strategy

### 3.1 What Is Being Reused

The SupplyTrack repositories provide a solid, well-structured Express + React foundation. The following patterns carry over directly into UniRush with renaming and extension:

| SupplyTrack Component | Maps to UniRush Component | Change Required |
|---|---|---|
| `users` table | `users` table | Add: `student_id`, `access_no`, `university`, `role_status`, `verification_docs` |
| `supplier_profiles` | `seller_profiles` | Add: TIN, KCCA licence, health cert, `operating_hours`, `service_radius`, `location_lat/lng` |
| `orders` + `order_items` | `orders` + `order_items` | Add: `payment_method`, `otp_code`, `otp_confirmed_at`, `delivery_fee`, `platform_fee`, `escrow_status` |
| `order_tracking` | `order_tracking` | Reused as-is; status vocabulary updated to UniRush statuses |
| `pickup_stations` | `delivery_zones` | Replace pickup-station model with GPS zone model |
| `authController.js` | `authController.js` | Extend `register()` for multi-role and student verification |
| `authMiddleware.js` | `authMiddleware.js` | Reused as-is — JWT `protect` pattern unchanged |
| `roleMiddleware.js` | `roleMiddleware.js` | Reused as-is — `allowRoles()` pattern unchanged |
| `asyncHandler.js` | `asyncHandler.js` | Reused as-is |
| `App.jsx` routing | `App.jsx` | Add routes for seller, rider, agent, and transport dashboards |
| `AuthContext.jsx` | `AuthContext.jsx` | Extend with `role_status`, `active_role` (CLIENT/AGENT mode switch) |
| `DashboardLayout` | `DashboardLayout` | Reskin to UniRush design tokens; add Role Switcher chip |
| `ProtectedRoute` + `RoleBasedRoute` | `ProtectedRoute` + `RoleBasedRoute` | Reused as-is |

### 3.2 New Backend Modules

The following modules have no SupplyTrack equivalent and must be built from scratch:

- **riders module** — rider profiles, zone assignment, geofence enforcement, online/offline toggle, earnings wallet.
- **dispatch module** — order-to-rider matching engine: proximity ranking, 60-second cascade logic, surge pricing multiplier.
- **payments module** — escrow management, mobile money integration hooks, COD cash ledger, automatic disbursement on OTP confirmation.
- **transport module** — passenger trip requests, car owner matching, fare calculation.
- **wallet module** — in-app wallet balance, withdrawal requests, transaction history.
- **notifications module** — push notification dispatch and in-app notification persistence.
- **verification module** — document upload review queue, admin approval/rejection workflow.

### 3.3 New Frontend Modules

- Live order tracker — polling or WebSocket-based GPS map view during active delivery.
- Rider App interface — low-data optimised job card, QR scanner, OTP entry.
- Seller Dashboard — live incoming order panel, menu management, payout history.
- Role Switcher chip — CLIENT MODE / AGENT MODE toggle on home screen.
- Wallet & Earnings screens — balance card, withdrawal modal, transaction ledger.
- Admin operations map — real-time view of all active deliveries and trips.

### 3.4 Design System Changes

SupplyTrack uses Bootstrap 5. UniRush replaces it with a custom design system using the following token set defined in `src/styles/tokens.css`:

| Token | Value | Usage |
|---|---|---|
| `color.brand.primary` | `#003366` | Navigation bars, headers, primary buttons (dark variant) |
| `color.brand.accent` | `#FFCC33` | All CTA buttons, toggles, active states, interactive elements |
| `color.neutral.white` | `#FFFFFF` | Card backgrounds, form surfaces, modal backgrounds |
| `color.neutral.text` | `#1A1A1A` | Body text, labels, icons |
| `color.feedback.error` | `#D32F2F` | Error borders, destructive buttons, suspension banners |
| `color.feedback.success` | `#2E7D32` | Success banners, confirmed delivery states |
| `spacing.base` | `8dp / 8px` | All spacing as multiples of 8 |
| `radius.pill` | `28dp` | Primary CTA buttons |
| `radius.card` | `12dp` | Standard cards and input fields |

---

## 4. Database Design

### 4.1 Overview

UniRush uses MySQL 8 with the InnoDB storage engine. All tables use `AUTO_INCREMENT` integer primary keys. Foreign keys are declared with `ON DELETE CASCADE` or `ON DELETE SET NULL` depending on whether child records are meaningful without the parent.

### 4.2 Core Tables

#### 4.2.1 users
Extended from SupplyTrack. Adds student verification fields and multi-role support.

| Column | Type | Notes |
|---|---|---|
| `id` | INT PK AI | Primary key |
| `full_name` | VARCHAR(120) | Required |
| `email` | VARCHAR(160) UNIQUE | Optional; phone is primary identifier |
| `phone` | VARCHAR(40) UNIQUE | Primary account identifier; verified via SMS OTP |
| `password` | VARCHAR(255) | bcrypt hashed (10 rounds) |
| `role` | ENUM | `client`, `seller`, `boda_rider`, `car_owner`, `student_agent`, `admin` |
| `active_role` | ENUM(`client`,`agent`) | For student agents: current mode switch state |
| `student_id` | VARCHAR(40) | e.g. S25B23/047; for client and student agent roles |
| `access_number` | VARCHAR(40) | e.g. B36048; used for student verification |
| `university` | VARCHAR(120) | University name from dropdown |
| `verification_status` | ENUM | `pending`, `approved`, `rejected` |
| `status` | VARCHAR(20) | `active`, `suspended`, `deactivated` |
| `created_at` | TIMESTAMP | Auto-set |

#### 4.2.2 seller_profiles
Extended from SupplyTrack `supplier_profiles`. Adds location, legal, and scheduling fields.

| Column | Type | Notes |
|---|---|---|
| `id` | INT PK AI | |
| `user_id` | INT FK → users | One seller account per user |
| `business_name` | VARCHAR(160) | Display name on platform |
| `business_type` | VARCHAR(60) | restaurant, supermarket, boutique, pharmacy, etc. |
| `tin` | VARCHAR(40) | URA Tax Identification Number |
| `kcca_licence` | VARCHAR(60) | Trading licence number |
| `health_cert_url` | VARCHAR(255) | Uploaded health certificate (food sellers) |
| `location_lat` | DECIMAL(10,7) | GPS pin latitude |
| `location_lng` | DECIMAL(10,7) | GPS pin longitude |
| `address` | VARCHAR(255) | Written address |
| `operating_hours` | JSON | `{mon: {open:'08:00', close:'22:00'}, ...}` |
| `service_radius_km` | DECIMAL(4,1) | Maximum delivery radius in km |
| `payout_phone` | VARCHAR(40) | Mobile money number for payouts |
| `rating_avg` | DECIMAL(3,2) | Cached average; updated after each review |
| `status` | VARCHAR(20) | `active`, `inactive`, `suspended`, `under_review` |
| `created_at` | TIMESTAMP | |

#### 4.2.3 rider_profiles
New table. Covers professional riders and student agents on the motorcycle or car track.

| Column | Type | Notes |
|---|---|---|
| `id` | INT PK AI | |
| `user_id` | INT FK → users | |
| `vehicle_type` | ENUM | `boda`, `car` |
| `plate_number` | VARCHAR(20) | Registered vehicle plate |
| `permit_number` | VARCHAR(60) | Riding/driving permit; nullable for unpermitted student boda agents |
| `permit_url` | VARCHAR(255) | Uploaded permit document |
| `nin` | VARCHAR(40) | National ID; required for professional track |
| `police_clearance_url` | VARCHAR(255) | Professional riders only |
| `is_student_agent` | TINYINT(1) | `1` = student agent track |
| `campus_zone_km` | DECIMAL(4,1) | Enforced geofence radius (3.0 for unpermitted agents) |
| `online_status` | ENUM | `online`, `offline` |
| `current_lat` | DECIMAL(10,7) | Updated every 15 seconds when online |
| `current_lng` | DECIMAL(10,7) | Updated every 15 seconds when online |
| `rating_avg` | DECIMAL(3,2) | Cached average |
| `total_trips` | INT | Lifetime completed trips counter |
| `wallet_balance` | DECIMAL(12,2) | In-app wallet; withdrawable to MoMo |
| `status` | VARCHAR(20) | `active`, `suspended`, `deactivated` |
| `created_at` | TIMESTAMP | |

#### 4.2.4 items
Retained from SupplyTrack with minor additions.

| Column | Type | Notes |
|---|---|---|
| `id` | INT PK AI | |
| `seller_id` | INT FK → seller_profiles | |
| `category_id` | INT FK → item_categories | |
| `item_name` | VARCHAR(160) | |
| `description` | TEXT | |
| `price` | DECIMAL(12,2) | Price in UGX |
| `stock_quantity` | INT | 0 = out of stock |
| `unit` | VARCHAR(40) | e.g. piece, kg, litre |
| `image` | VARCHAR(255) | Path to uploaded image |
| `is_available` | TINYINT(1) | Seller can toggle without deleting |
| `status` | VARCHAR(20) | `active`, `inactive` |
| `created_at` | TIMESTAMP | |

#### 4.2.5 orders
Extended from SupplyTrack `orders`. Adds delivery model, payment model, OTP, and financial split fields.

| Column | Type | Notes |
|---|---|---|
| `id` | INT PK AI | |
| `order_number` | VARCHAR(80) UNIQUE | Format: `UR-{timestamp}-{seller_id}` |
| `client_id` | INT FK → users | |
| `seller_id` | INT FK → seller_profiles | |
| `rider_id` | INT FK → rider_profiles | Assigned after dispatch; nullable until then |
| `delivery_address` | TEXT | Free-text delivery instructions |
| `delivery_lat` | DECIMAL(10,7) | |
| `delivery_lng` | DECIMAL(10,7) | |
| `food_cost` | DECIMAL(12,2) | Sum of item totals |
| `delivery_fee` | DECIMAL(12,2) | Calculated by dispatch engine |
| `platform_fee` | DECIMAL(12,2) | 10% of food_cost + 20% of delivery_fee |
| `total_amount` | DECIMAL(12,2) | food_cost + delivery_fee |
| `payment_method` | ENUM | `mobile_money`, `cash_on_delivery`, `wallet` |
| `escrow_status` | ENUM | `held`, `released`, `refunded` |
| `otp_code` | VARCHAR(10) | 4-digit code; hashed before storage |
| `otp_confirmed_at` | TIMESTAMP NULL | Set when rider enters correct OTP |
| `cod_cash_collected` | DECIMAL(12,2) | Logged when rider marks cash received (COD only) |
| `cod_remitted_at` | TIMESTAMP NULL | Set when remittance confirmed (COD only) |
| `status` | VARCHAR(50) | See state machine in §6.3 |
| `customer_notes` | TEXT | |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | Auto-update |

#### 4.2.6 Supporting Tables

| Table | Origin | Key Changes |
|---|---|---|
| `order_items` | SupplyTrack | No changes; stores per-item quantity and `unit_price` snapshot |
| `order_tracking` | SupplyTrack | Status vocabulary updated; `current_location` becomes GPS coordinates |
| `item_categories` | SupplyTrack | Renamed categories aligned to UniRush service types |
| `carts` / `cart_items` | SupplyTrack | No changes |
| `notifications` | SupplyTrack | Add `type` ENUM: `order_update`, `job_alert`, `payment`, `system`, `dispute` |
| `audit_logs` | SupplyTrack | No changes |
| `ratings` | New | `user_id`, `target_id`, `target_type` (seller/rider/client), `order_id`, `stars`, `comment`, `created_at` |
| `disputes` | New | `order_id`, `raised_by`, `reason`, `photo_url`, `status` (open/resolved), `resolution`, `created_at` |
| `cash_ledger` | New | `rider_id`, `order_id`, `amount_collected`, `amount_remitted`, `remitted_at`, `status` (pending/cleared/overdue) |
| `transport_trips` | New | `client_id`, `rider_id`, `pickup_lat/lng`, `dest_lat/lng`, `vehicle_type`, `fare`, `status`, `started_at`, `completed_at` |

---

## 5. API Design

### 5.1 Base URL and Auth

- **Development:** `http://localhost:5000/api`
- **Production:** `https://api.unirush.ug/api`

All protected routes require: `Authorization: Bearer <jwt>`. Tokens are HS256 JWTs with 24-hour expiry. A refresh token endpoint (`POST /api/auth/refresh`) provides 7-day rolling sessions.

### 5.2 Retained Routes (from SupplyTrack)

| Method | Route | Auth | Description |
|---|---|---|---|
| POST | `/api/auth/register` | Public | Register new user (any role) |
| POST | `/api/auth/login` | Public | Login; returns JWT |
| GET | `/api/auth/me` | JWT | Get current user profile |
| GET | `/api/items` | Public | Browse active items with filtering |
| GET | `/api/items/:id` | Public | Item detail |
| POST | `/api/items` | Seller | Create item listing |
| PUT | `/api/items/:id` | Seller (owner) | Edit item |
| DELETE | `/api/items/:id` | Seller (owner) | Soft-delete item |
| GET | `/api/categories` | Public | List all categories |
| POST | `/api/categories` | Admin | Create category |
| GET | `/api/cart` | Client | Get current cart |
| POST | `/api/cart` | Client | Add item to cart |
| PUT | `/api/cart/:itemId` | Client | Update cart item quantity |
| DELETE | `/api/cart/:itemId` | Client | Remove item from cart |
| POST | `/api/orders` | Client | Place order from cart |
| GET | `/api/orders/my-orders` | Client | List client's own orders |
| GET | `/api/orders/:id` | JWT (scoped) | Order detail (client, seller, rider, admin) |
| PUT | `/api/orders/:id/cancel` | Client | Cancel pending order |
| PUT | `/api/orders/:id/confirm-received` | Client | Client confirms delivery after OTP |
| GET | `/api/tracking/:orderId` | JWT (scoped) | Full tracking timeline for an order |
| GET | `/api/profile` | JWT | Get own profile |
| PUT | `/api/profile` | JWT | Update own profile |
| GET | `/api/admin/stats` | Admin | Platform-wide statistics |
| GET | `/api/admin/users` | Admin | List all users |
| PUT | `/api/admin/users/:id/status` | Admin | Activate or suspend a user |
| GET | `/api/admin/orders` | Admin | List all orders |
| GET | `/api/admin/pickup-stations` | Admin | List delivery zones |
| POST | `/api/admin/pickup-stations` | Admin | Create delivery zone |
| PUT | `/api/admin/pickup-stations/:id` | Admin | Update delivery zone |

### 5.3 New Routes

| Method | Route | Auth | Description |
|---|---|---|---|
| POST | `/api/auth/refresh` | Refresh token | Issue new access token |
| POST | `/api/orders/:id/otp/verify` | Rider | Rider submits OTP; triggers escrow release and earnings credit |
| GET | `/api/seller/orders` | Seller | Live and historical orders for this seller |
| PUT | `/api/seller/orders/:id/accept` | Seller | Accept incoming order (3-minute window) |
| PUT | `/api/seller/orders/:id/decline` | Seller | Decline with required reason |
| PUT | `/api/seller/orders/:id/ready` | Seller | Mark order ready for pickup; triggers dispatch |
| GET | `/api/seller/profile` | Seller | Get own seller profile |
| PUT | `/api/seller/profile` | Seller | Update seller profile and operating hours |
| POST | `/api/seller/items/:id/toggle` | Seller | Toggle item availability without deleting |
| GET | `/api/riders/online` | Admin/Dispatch | List all online riders with live location |
| PUT | `/api/riders/status` | Rider/Agent | Toggle online/offline; update GPS location |
| GET | `/api/riders/jobs` | Rider/Agent | Available jobs near current location |
| POST | `/api/riders/jobs/:orderId/accept` | Rider/Agent | Accept a dispatched job |
| PUT | `/api/riders/jobs/:orderId/pickup` | Rider/Agent | Mark order as picked up from seller |
| PUT | `/api/riders/jobs/:orderId/deliver` | Rider/Agent | Confirm delivery after OTP entry |
| GET | `/api/riders/wallet` | Rider/Agent | Wallet balance and transaction history |
| POST | `/api/riders/wallet/withdraw` | Rider/Agent | Initiate MoMo withdrawal |
| GET | `/api/riders/cash-ledger` | Rider/Agent | View outstanding COD remittance obligations |
| POST | `/api/riders/cash-ledger/:id/remit` | Rider/Agent | Confirm COD remittance via MoMo |
| POST | `/api/transport/request` | Client | Request a boda or car trip |
| GET | `/api/transport/trips/my` | Client | Client's trip history |
| PUT | `/api/transport/trips/:id/complete` | Driver | Mark trip complete; trigger fare deduction |
| POST | `/api/payments/initiate` | Client | Initiate mobile money payment via gateway |
| POST | `/api/payments/callback` | Public (gateway) | Payment gateway webhook; updates escrow status |
| POST | `/api/ratings` | Client/Rider | Submit rating after completed order or trip |
| POST | `/api/disputes` | Client | Open a dispute on a delivered order |
| GET | `/api/disputes/:id` | JWT (scoped) | View dispute details |
| PUT | `/api/admin/disputes/:id/resolve` | Admin | Resolve dispute; issue refund if upheld |
| GET | `/api/admin/verifications` | Admin | Pending user verification document queue |
| PUT | `/api/admin/verifications/:userId/approve` | Admin | Approve user; activate role |
| PUT | `/api/admin/verifications/:userId/reject` | Admin | Reject with reason; user can re-submit |
| POST | `/api/notifications/push` | Admin/System | Send push notification to user or group |
| GET | `/api/notifications` | JWT | Get own notification inbox |
| PUT | `/api/notifications/:id/read` | JWT | Mark notification as read |
| POST | `/api/sos` | JWT | Trigger SOS; sends GPS to emergency team and next-of-kin |

---

## 6. Backend Architecture

### 6.1 Project Structure

```
unirush-backend/
├── server.js                         # Entry point; registers all routes and middleware
├── config/
│   └── db.js                         # mysql2 connection pool (retained as-is)
├── middleware/
│   ├── authMiddleware.js              # JWT verify; populates req.user (retained as-is)
│   ├── roleMiddleware.js              # allowRoles() RBAC factory (retained as-is)
│   ├── asyncHandler.js               # Error propagation wrapper (retained as-is)
│   ├── uploadMiddleware.js            # Multer config for photos and documents
│   └── geofenceMiddleware.js          # NEW: validates rider GPS vs campus zone
├── controllers/
│   ├── authController.js              # Extended: multi-role registration, student ID verify
│   ├── orderController.js             # Extended: OTP generation, escrow, COD ledger
│   ├── sellerController.js            # NEW: order acceptance, menu mgmt, seller profile
│   ├── riderController.js             # NEW: online toggle, job accept, OTP delivery, wallet
│   ├── dispatchController.js          # NEW: matching engine, surge pricing, cascade logic
│   ├── paymentController.js           # NEW: MoMo gateway initiation, webhook, disbursement
│   ├── transportController.js         # NEW: trip requests, car matching, fare calculation
│   ├── walletController.js            # NEW: balance queries, withdrawal requests
│   ├── ratingController.js            # NEW: submit and retrieve ratings
│   ├── disputeController.js           # NEW: open, view, and resolve disputes
│   ├── notificationController.js      # NEW: push dispatch, inbox, mark read
│   ├── profileController.js           # Retained; extended
│   ├── trackingController.js          # Retained as-is
│   ├── cartController.js              # Retained as-is
│   ├── itemController.js              # Retained; minor extensions
│   ├── categoryController.js          # Retained as-is
│   └── adminController.js             # Extended: verification queue, dispute mgmt, ops map
├── routes/                            # One route file per controller
├── services/
│   ├── dispatchService.js             # NEW: pure matching algorithm extracted from controller
│   ├── otpService.js                  # NEW: OTP generation, hashing, and validation
│   ├── paymentService.js              # NEW: MTN and Airtel MoMo API wrapper
│   └── geofenceService.js             # NEW: haversine distance calculation
├── database/
│   ├── schema.sql                     # Extended schema (all tables in §4)
│   └── seed.sql                       # Test data for local development
└── uploads/                           # Static file storage for images and documents
```

### 6.2 Authentication and Authorisation

- All protected routes pass through `protect` (JWT verify) first, then `allowRoles()` for role enforcement.
- JWT payload includes: `{ id, email, phone, role, active_role, full_name, verification_status }`.
- Token expiry: 24 hours access / 7 days refresh. Refresh tokens are stored server-side and are revocable.
- Student Agent role switching does not require re-authentication; it updates `active_role` on the `users` table and issues a new short-lived access token.
- After 5 failed login attempts, the account is locked for 30 minutes and the registered phone receives an alert SMS.

### 6.3 Order State Machine

Every order transition inserts a new `order_tracking` row.

| Status | Triggered By | Next Possible States |
|---|---|---|
| `Pending` | Client places order | `Accepted`, `Cancelled` |
| `Accepted` | Seller accepts (within 3 min) | `Preparing`, `Cancelled` |
| `Preparing` | Seller updates prep time | `Ready for Pickup` |
| `Ready for Pickup` | Seller taps Order Ready | `Picked Up` |
| `Picked Up` | Rider scans QR + taps Picked Up | `En Route` |
| `En Route` | Auto-set on pickup confirm | `Delivered` |
| `Delivered` | Rider enters OTP from client | `Completed` (30-min dispute window) |
| `Completed` | Client confirms or 30-min window expires | Terminal state |
| `Cancelled` | Client, seller (decline), or system timeout | Terminal state |

### 6.4 COD Cash Ledger Flow

1. Client places COD order. UniRush pays seller their share (`food_cost × 0.90`) from the platform float immediately.
2. Rider picks up and delivers. Customer pays the full `total_amount` in cash at the door.
3. System logs the full amount in the `cash_ledger` table against the rider's account.
4. Rider keeps `delivery_fee × 0.80`. They must remit the remainder within 24 hours via MoMo.
5. On confirmed remittance, the `cash_ledger` entry is cleared and the rider's wallet earnings are unlocked.
6. If no remittance within 24 hours, the rider's account is flagged and new job acceptance is blocked until cleared.

### 6.5 Dispatch Engine Logic

When a seller marks an order `Ready for Pickup`:

1. System identifies all online riders with `online_status = 'online'` within 3 km of the seller's GPS coordinates.
2. Riders are ranked: proximity (ascending) → rating (descending) → total trips (descending) → last active time.
3. Job is offered to the top-ranked rider. They have 60 seconds to accept.
4. If not accepted within 60 seconds, the job cascades to the next ranked rider.
5. Student Agent riders are included in the pool only for orders within their campus zone.
6. If no rider accepts within 5 minutes, the client and seller are notified with options to wait or cancel.
7. Surge pricing of 1.2×–1.5× applies during 12:00–2:00 PM and 6:00–9:00 PM.

---

## 7. Frontend Architecture

### 7.1 Project Structure

```
unirush-frontend/
├── index.html
├── src/
│   ├── main.jsx                        # React root; wraps in AuthProvider + BrowserRouter
│   ├── App.jsx                         # Full route tree (all roles)
│   ├── styles/
│   │   └── tokens.css                  # CSS custom properties for design token set
│   ├── context/
│   │   ├── AuthContext.jsx             # Extended: active_role, switchRole(), verification_status
│   │   └── CartContext.jsx             # Retained and lightly extended
│   ├── layouts/
│   │   ├── PublicLayout.jsx            # Retained; reskinned to UniRush design
│   │   ├── DashboardLayout.jsx         # Extended: Role Switcher chip, UniRush nav
│   │   └── RiderLayout.jsx             # NEW: low-data optimised layout for rider views
│   ├── components/
│   │   ├── ProtectedRoute.jsx          # Retained as-is
│   │   ├── RoleBasedRoute.jsx          # Retained as-is
│   │   ├── Navbar.jsx                  # Reskinned to UniRush design
│   │   ├── OrderStatusBadge.jsx        # Updated with UniRush status vocabulary
│   │   ├── TrackingTimeline.jsx        # Retained; reskinned
│   │   ├── RoleSwitcher.jsx            # NEW: CLIENT MODE / AGENT MODE chip
│   │   ├── LiveMap.jsx                 # NEW: GPS map component using Leaflet (OpenStreetMap)
│   │   ├── OtpEntry.jsx                # NEW: 4-digit OTP input (Courier New display)
│   │   ├── RiderJobCard.jsx            # NEW: 60-second countdown accept card
│   │   ├── WalletCard.jsx              # NEW: balance display with withdraw button
│   │   └── SosButton.jsx               # NEW: tap-and-hold emergency SOS
│   ├── pages/
│   │   ├── public/                     # Home, About, BrowseItems, ItemDetails, Login, Register
│   │   ├── client/                     # Dashboard, Cart, Checkout, MyOrders, OrderDetails, TrackOrder, TransportRequest
│   │   ├── seller/                     # Dashboard, LiveOrders, MenuManagement, AddItem, EditItem, Payouts, SellerProfile
│   │   ├── rider/                      # Dashboard, GoOnline, ActiveJob, Wallet, CashLedger, RiderProfile
│   │   └── admin/                      # Dashboard, ManageUsers, ManageOrders, ManageItems, VerificationQueue, DisputeCenter, OperationsMap
│   ├── services/                       # One Axios service file per API domain
│   ├── hooks/
│   │   ├── useOrders.js
│   │   ├── useRiderLocation.js         # Polls rider GPS every 5s during active delivery
│   │   ├── useWallet.js
│   │   └── useNotifications.js
│   └── utils/
│       └── formatting.js              # UGX currency, date, distance, and fare formatters
```

### 7.2 Routing Structure

| Role | Base Path | Key Pages |
|---|---|---|
| Client | `/client` | Dashboard, Cart, Checkout, Orders, Track, Transport |
| Seller | `/seller` | Dashboard, Live Orders, Menu, Add/Edit Item, Payouts, Profile |
| Boda Rider / Agent | `/rider` | Dashboard, Go Online, Active Job, Wallet, Cash Ledger, Profile |
| Car Owner / Agent | `/driver` | Dashboard, Go Online, Active Trip, Wallet, Profile |
| Admin | `/admin` | Dashboard, Users, Orders, Items, Verifications, Disputes, Ops Map |
| Public | `/` | Home, Browse Items, Item Detail, Login, Register |

### 7.3 State Management

- **Global auth state:** `AuthContext` (`user`, `active_role`, `isAuthenticated`, `login`, `logout`, `switchRole`).
- **Cart state:** `CartContext` (`items`, `addItem`, `removeItem`, `clearCart`).
- **Server state:** Axios services called directly from components and custom hooks. No Redux or Zustand in scope for v1.
- **Live tracking:** `useRiderLocation` polls `GET /api/riders/online/:id` every 5 seconds during active delivery. WebSocket upgrade planned for v2.

### 7.4 Key Component Specifications

#### RoleSwitcher
Persistent chip at the top of the home screen for dual-role accounts. Tapping triggers a confirmation dialog if the other role has an active order or delivery pending. On confirm, calls `authService.switchRole()`, updates `AuthContext`, re-routes to the appropriate dashboard, and changes the app chrome colour (deep blue for Agent, white for Client).

#### RiderJobCard
Displayed when the dispatch engine pushes a job to an online rider. Shows: seller location, distance, estimated earnings, item type. A circular SVG arc timer depletes over 60 seconds and shifts from gold to red in the final 15 seconds. Decline requires tap-and-hold to prevent accidental rejection.

#### OtpEntry
A 4-digit input rendered in Courier New 20pt Bold. Each box briefly scales on digit entry. A prominent warning reads: *"Do not share this code until you have physically received your order."* The rider enters the OTP they collect from the client in person.

#### SosButton
A red circular button (56 × 56 dp) visible on all active delivery and trip screens. Requires a 1.5-second tap-and-hold to activate (countdown ring on hold). On activation: full-screen red overlay, GPS location sent to platform emergency team and next-of-kin via `POST /api/sos`. Three rapid haptic pulses confirm activation.

---

## 8. Non-Functional Requirements

| Requirement | Target | Implementation Approach |
|---|---|---|
| Performance | API response < 500ms p95; page load < 3s | mysql2 connection pooling; Vite production build with code splitting; static asset CDN |
| Availability | 99.5% uptime | PM2 process manager with auto-restart; health check at `/api/health` |
| Scalability | Up to 500 concurrent users | Stateless JWT auth enables horizontal scaling; DB indices on FK columns and frequent query fields |
| Security | JWT auth; bcrypt; HTTPS; RBAC | All routes behind `protect` + `allowRoles`; passwords hashed at 10 rounds; secrets in `.env`; CORS restricted to known origins |
| Data Privacy | PDPA 2019 compliant | Location data purged after 72 hours (GPS tracking) / 90 days (delivery history); phone numbers never sent to other users |
| Usability | WCAG AA; accessible to non-technical users | 48×48dp minimum touch targets; WCAG AA contrast ratios; TalkBack and VoiceOver support; text scale to 200% |
| Maintainability | Modular, documented code | One controller per domain; `asyncHandler` wraps all async; JSDoc on service functions |
| Offline Resilience | Rider app graceful offline | Last sync time banner; accepted job cached locally; OTP entry queued for reconnect |

---

## 9. Security Design

### 9.1 Authentication
- Phone number is the primary identifier, verified via 6-digit SMS OTP at registration.
- Passwords are bcrypt-hashed at 10 rounds. Never stored or logged in plaintext.
- JWT access tokens expire in 24 hours; refresh tokens expire in 7 days and are stored server-side (revocable).
- Biometric login offered after first successful login on supported devices.
- 5 failed login attempts → 30-minute lockout + alert SMS to registered phone.

### 9.2 Authorisation
- Every protected route: `protect` (JWT verify) → `allowRoles(...)` → controller.
- Controllers perform a second ownership check for scoped resources (a seller can only edit their own items; a client can only view their own orders).
- Student Agent geofence enforced both client-side (GPS alert) and server-side (`geofenceMiddleware` validates coordinates on every job accept request).

### 9.3 Data Protection
- All API communication over HTTPS (TLS 1.2+).
- Delivery addresses and phone numbers shared with the assigned rider only during the active delivery window.
- All in-app messaging is masked; no phone numbers ever sent in API responses to other users.
- Delivery location history: stored max 90 days for dispute resolution, then purged by a scheduled nightly job.
- GPS tracking data: not stored beyond 72 hours (PDPA 2019 compliance).

### 9.4 Fraud Prevention
- Marking an order complete without OTP entry is rejected by the API and logged as a policy violation.
- Duplicate account detection on both phone number and student ID at registration.
- Dispute honesty prompt required before submission; false disputes result in account suspension.
- COD non-remittance within 24 hours triggers automatic account flag and job acceptance suspension.

---

## 10. Platform Policies — Technical Enforcement

| Policy | Applies To | Technical Enforcement |
|---|---|---|
| One account per person | All users | Unique constraint on `phone` and `student_id`; duplicate detection at registration |
| Student Agent campus zone (3 km) | Student Agents without permit | `geofenceMiddleware` on job accept endpoint; geofence alert in rider app; GPS validated server-side |
| COD 24-hour remittance deadline | Riders (COD orders) | Cron job every 30 min; flags overdue `cash_ledger` entries; disables job acceptance for flagged riders |
| OTP required at handoff | Riders | Order cannot transition to `Delivered` without valid OTP hash match; API rejects without `otp_code` |
| Seller 3-minute order acceptance | Sellers | Backend auto-cancels after 3 min if not accepted; push notification sent at order receipt |
| Rating thresholds | Sellers and Riders | Background job checks 30-day rolling average; warning at 3.5 stars; unlisting/flag at 3.0 stars |
| Conduct tiers (minor/serious/zero-tolerance) | All users | `status` field on `users` table (`active`, `suspended`, `deactivated`); suspension blocks login |
| Permit expiry monitoring | Riders | Permit expiry tracked in `rider_profiles`; alert 30 days before expiry; expired permit suspends account |
| Data privacy (location) | Platform | Nightly DELETE on `order_tracking` records > 90 days; GPS records purged after 72 hours |

---

## 11. Deployment & Environment

### 11.1 Development Environment

| Service | Tool / Version |
|---|---|
| Backend runtime | Node.js 20 LTS |
| Frontend build | Vite 8 + React 18 |
| Database | MySQL 8 (local or Docker) |
| Process manager (dev) | nodemon |
| Package manager | npm |
| API testing | Postman or Thunder Client |
| Version control | Git; two repositories (`unirush-backend`, `unirush-frontend`) |

### 11.2 Environment Variables (Backend)

| Variable | Description |
|---|---|
| `PORT` | Server port (default: 5000) |
| `DB_HOST` | MySQL host |
| `DB_USER` | MySQL username |
| `DB_PASSWORD` | MySQL password |
| `DB_NAME` | Database name (`unirush`) |
| `DB_PORT` | MySQL port (default: 3306) |
| `JWT_SECRET` | Secret for JWT signing (minimum 32 characters) |
| `JWT_REFRESH_SECRET` | Secret for refresh token signing |
| `MTN_MOMO_API_KEY` | MTN Mobile Money API key |
| `AIRTEL_MONEY_API_KEY` | Airtel Money API key |
| `PAYMENT_CALLBACK_URL` | Publicly accessible URL for payment gateway webhooks |
| `SOS_EMERGENCY_PHONE` | Platform emergency team phone number |

### 11.3 Production Deployment (Recommended)

- **Backend:** VPS (DigitalOcean Droplet or AWS EC2); managed with PM2; reverse-proxied via Nginx; SSL via Let's Encrypt.
- **Frontend:** `npm run build`; static files served via Netlify, Vercel, or Nginx static server at `unirush.ug`.
- **Database:** Managed MySQL instance (PlanetScale, Railway, or self-hosted) with automated backups.
- **File uploads:** Migrated from local `/uploads` to cloud object storage (AWS S3 or Cloudinary) for v1 production.

---

## 12. Testing Strategy

| Test Type | Scope | Tool / Approach |
|---|---|---|
| Unit tests | Service functions (dispatch, OTP, geofence, payment splitting) | Jest; pure function inputs and outputs |
| Integration tests | API route + controller + database | Supertest with a test MySQL database; happy path and error cases per endpoint |
| Role-based access tests | All protected routes | Supertest; each route tested with no token, wrong role, and correct role |
| Order state machine tests | All valid and invalid state transitions | Jest; simulate transition sequences and verify `order_tracking` rows |
| COD ledger tests | Cash collection, remittance, flag logic | Jest + Supertest; simulate 24-hour timeout with mocked cron |
| Frontend component tests | OtpEntry, RiderJobCard, RoleSwitcher, SosButton | Vitest + React Testing Library |
| Manual QA | Full user journeys per role | Emilly Sendikaddiwa leads QA; checklist per sprint |
| Accessibility audit | All screens at 200% font, TalkBack, VoiceOver | Manual device testing + axe-core automated scan |

---

## 13. Open Issues & Decisions Pending

| # | Issue | Owner | Status |
|---|---|---|---|
| 1 | Real-time rider GPS: polling (5s) vs WebSocket. Polling is simpler for v1; WebSocket gives better UX. | Matthias Imani | Deferred to v2 |
| 2 | MoMo integration: MTN and Airtel APIs require sandbox credentials before payment module can be built. | Matthias Imani | In progress |
| 3 | Push notifications: Firebase Cloud Messaging (FCM) is recommended. Requires Google service account setup. | Eric Dickens | Pending setup |
| 4 | Map provider: Leaflet + OpenStreetMap (free, offline-capable) vs Google Maps (cost implications). | Ernest Ntare | Leaflet selected |
| 5 | Student ID verification: automated UCU records API integration vs manual photo ID review. | Ernest Ntare | Manual review for v1 |
| 6 | Surge pricing: exact multiplier logic (1.2×–1.5×) and peak hour detection algorithm not yet defined. | Matthias Imani | To be defined |
| 7 | Platform float for COD: initial reserve amount and replenishment process to be agreed with Finance. | Team Lead | Pending |

---

## Appendix A: Glossary

| Term | Definition |
|---|---|
| `asyncHandler` | Express middleware wrapper that catches async errors and forwards them to the error handler without try/catch in every controller. |
| Dispatch Engine | Server-side algorithm that matches a ready order to the nearest available, highest-rated rider within 3 km. |
| Escrow | The state where a client's payment is held by the platform and not yet released to the seller or rider. |
| Geofence | A virtual GPS boundary. UniRush uses haversine distance to enforce the 3 km campus zone for unpermitted Student Agents. |
| OTP | One-Time Password. A 4-digit code generated server-side, hashed before storage, and collected from the client in person at delivery. |
| Platform Float | The operating reserve UniRush maintains to front restaurant payments on COD orders before the rider remits cash. |
| RBAC | Role-Based Access Control. The `allowRoles()` middleware factory enforces which roles can access which routes. |
| Role Switcher | The in-app toggle allowing a Student Agent to switch between CLIENT MODE (ordering) and AGENT MODE (earning). |
| Surge Pricing | A temporary delivery fee multiplier (1.2×–1.5×) applied during peak hours (12:00–2:00 PM and 6:00–9:00 PM) to attract more riders online. |

---

## Appendix B: References

- UniRush Project Brief v1.0 — June 2026
- UniRush Pricing, Revenue & Payment Policy v1.0 — June 2026
- UniRush Complete App Flow, Operations & Policies Guide v2.0 — June 2026
- UniRush UI/UX Design Documentation v2.0 — June 2026
- SupplyTrack Backend Repository: github.com/Musasizi/supplytrack-backend
- SupplyTrack Frontend Repository: github.com/Musasizi/supplytrack-frontend
- Express.js Documentation: expressjs.com
- React Documentation: react.dev
- MySQL 8.0 Reference Manual: dev.mysql.com/doc
- Uganda Personal Data Protection Act 2019
- Uganda Traffic and Road Safety Act Cap 361

---

*UniRush · Quick. Safe. Convenient. · Group 17 · SDD v1.0 · June 2026 · Confidential*