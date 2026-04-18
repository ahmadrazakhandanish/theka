# Land Theka Management System — SRS

**Version:** 2.0  
**Status:** Final Draft — Ready for Development  
**Audience:** Full-Stack Development Team  
**Platform:** Mobile (iOS / Android) + Web Application  
**Architecture:** Multi-Tenant SaaS — REST API + Mobile Clients  
**Tech Stack:** Node.js / Laravel API · React Native · PostgreSQL

---

## Table of Contents

1. [Project Overview & Goals](#1-project-overview--goals)
2. [Stakeholders & User Roles](#2-stakeholders--user-roles)
3. [System Architecture](#3-system-architecture)
4. [Database Schema](#4-database-schema)
5. [Core Functional Requirements](#5-core-functional-requirements)
   - 5.1 [Authentication & Onboarding](#51-authentication--onboarding)
   - 5.2 [Land Management](#52-land-management)
   - 5.3 [Contract Management](#53-contract-management)
   - 5.4 [Transaction System](#54-transaction-system)
   - 5.5 [Payment Split Engine](#55-payment-split-engine)
   - 5.6 [Proof & Verification System](#56-proof--verification-system)
   - 5.7 [Notifications](#57-notification-system)
   - 5.8 [Chat System](#58-chat-system)
   - 5.9 [Reports & Analytics](#59-reports--analytics)
6. [API Endpoint Reference](#6-api-endpoint-reference)
7. [Non-Functional Requirements](#7-non-functional-requirements)
8. [Security Requirements](#8-security-requirements)
9. [UI/UX Requirements](#9-uiux-requirements)
10. [Acceptance Criteria](#10-acceptance-criteria)
11. [Glossary](#11-glossary)

---

## 1. Project Overview & Goals

Land Theka Management System (LTMS) is a multi-tenant SaaS platform for agricultural land leasing in Pakistan. It digitizes the full lifecycle of land contracts — from registration and tenant assignment, through GPS-verified payment collection, to automatic distribution of proceeds among multiple landowners (partners) — all backed by an immutable audit trail that eliminates fraud.

### 1.1 Problem Statement

| Current Pain Point | LTMS Solution |
|----|-----|
| Verbal/paper contracts are lost or disputed | Digital contracts with signatures, stored permanently |
| Rent payments are unverified or denied | GPS-stamped, photo-proof transactions with multi-step confirmation |
| Partner share disputes | Automated split engine based on ownership percentages |
| No audit trail for modifications | Immutable ledger; every action logged with timestamp and actor |
| Fraud by intermediaries | Zero-trust: every transaction needs tenant, receiver, AND owner approval |

### 1.2 Success Metrics

- All rent transactions recorded with GPS proof inside the app
- Zero unresolved payment disputes due to the digital confirmation chain
- Partner shares auto-calculated correctly for 100% of transactions
- Full audit log accessible by Owner at any time
- System handles 50+ organizations with full data isolation

---

## 2. Stakeholders & User Roles

Each organization has its own isolated set of users. Roles are scoped per organization.

> **Note:** All roles below Super Admin are scoped to a single Organization. Users cannot see or access data from other organizations.

| Role | Description | Key Permissions |
|------|-------------|-----------------|
| **Super Admin** *(Platform)* | Platform operator. Manages all organizations. | Create orgs, billing, platform config |
| **Owner / Admin** *(Org)* | The landowner who set up the org. Full control. | Create lands, contracts, approve transactions, all reports, manage users |
| **Partner** | Co-owner of one or more lands. Receives share of rent. | View assigned contracts, view own payment splits, view proof |
| **Tenant** | The person leasing the land. Pays rent. | Create transactions, attach proof, view own contracts & payment history |
| **Accountant** *(Optional)* | Manages financials on behalf of owner. | View all financials, generate reports — cannot approve transactions |
| **Viewer** *(Optional)* | Read-only stakeholder (e.g. silent partner). | View reports and contracts only, no write access |

---

## 3. System Architecture

### 3.1 High-Level Architecture

| Layer | Technology | Responsibility |
|-------|-----------|----------------|
| Mobile App | React Native (iOS + Android) | Primary user interface for all roles |
| Web App | React.js | Owner/Accountant dashboard, reports, admin panel |
| API Server | Node.js (Express) or Laravel (PHP) | Business logic, authentication, data processing |
| Database | PostgreSQL | Primary relational data store |
| File Storage | AWS S3 / Cloudflare R2 | Proof images, voice notes, documents |
| Push Notifications | Firebase Cloud Messaging (FCM) | Real-time alerts for approvals and payments |
| Authentication | JWT + Refresh Tokens | Stateless auth with role claims |
| Cache / Queue | Redis | Session cache, background job queue for notifications |

### 3.2 Multi-Tenancy Model

Each organization is isolated at the database level using an `organization_id` foreign key on every table. Row-level security (RLS) is enforced in both PostgreSQL policies and the API middleware.

- Middleware validates: `JWT token → user → organization_id → requested resource belongs to same org`
- Super Admin bypass: a separate `super_admin` flag grants cross-org access
- No shared data between organizations except platform-level lookup tables (e.g. Pakistani provinces)

### 3.3 Mobile-First Considerations

- All API responses paginated (default page size: 20)
- Images compressed client-side before upload (max 1 MB per image)
- Offline queue: transaction drafts saved locally, synced when internet available
- GPS coordinates captured at moment of transaction creation — cannot be edited post-submission

---

## 4. Database Schema

All tables include `created_at`, `updated_at`, and `deleted_at` (soft delete). All IDs are UUIDs.

### 4.1 `organizations`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | Unique org identifier |
| name | VARCHAR(255) | NOT NULL | Organization display name |
| slug | VARCHAR(100) | UNIQUE, NOT NULL | URL-safe identifier |
| owner_user_id | UUID | FK → users | Primary owner account |
| proof_config | JSONB | NOT NULL | Required proof types: `{image, voice, cnic, signature, gps}` |
| max_partners | INT | DEFAULT 10 | Max allowed partners per contract |
| subscription_plan | VARCHAR(50) | DEFAULT 'basic' | Billing plan |
| is_active | BOOLEAN | DEFAULT true | Soft disable org |

### 4.2 `users`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| organization_id | UUID | FK → organizations | Scoping key |
| full_name | VARCHAR(255) | NOT NULL | |
| phone | VARCHAR(20) | UNIQUE, NOT NULL | Primary login identifier |
| cnic | VARCHAR(15) | NULLABLE | Pakistani national ID |
| role | ENUM | NOT NULL | `owner \| partner \| tenant \| accountant \| viewer` |
| pin_hash | VARCHAR(255) | NOT NULL | Bcrypt-hashed 4–6 digit PIN |
| fcm_token | TEXT | NULLABLE | Firebase push token |
| is_active | BOOLEAN | DEFAULT true | |

### 4.3 `lands`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| organization_id | UUID | FK → organizations | |
| name | VARCHAR(255) | NOT NULL | Land title / identifier |
| location_text | TEXT | NOT NULL | Human-readable address |
| latitude | DECIMAL(10,8) | NULLABLE | GPS of land location |
| longitude | DECIMAL(11,8) | NULLABLE | |
| total_acres | DECIMAL(10,3) | NOT NULL | Total leasable acreage |
| province | VARCHAR(100) | NOT NULL | |
| status | ENUM | DEFAULT 'available' | `available \| leased \| inactive` |

### 4.4 `land_partners` (Ownership)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| land_id | UUID | FK → lands | |
| partner_user_id | UUID | FK → users | |
| ownership_percent | DECIMAL(5,2) | NOT NULL, CHECK ≤ 100 | Must sum to 100% across all partners of a land |
| max_receivable_limit | DECIMAL(12,2) | NULLABLE | Cap on how much this partner can receive per payment cycle |

### 4.5 `contracts`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| organization_id | UUID | FK | |
| land_id | UUID | FK → lands | |
| tenant_user_id | UUID | FK → users | The renter |
| title | VARCHAR(255) | NOT NULL | |
| pricing_type | ENUM | NOT NULL | `per_acre \| fixed` |
| rate_per_acre | DECIMAL(12,2) | NULLABLE | Used if `per_acre` |
| fixed_amount | DECIMAL(12,2) | NULLABLE | Used if `fixed` |
| total_contract_value | DECIMAL(14,2) | NOT NULL | Auto-calculated |
| start_date | DATE | NOT NULL | |
| end_date | DATE | NOT NULL | |
| payment_frequency | ENUM | NOT NULL | `monthly \| quarterly \| biannual \| annual \| custom` |
| status | ENUM | DEFAULT 'draft' | `draft \| active \| expired \| terminated` |
| notes | TEXT | NULLABLE | Special clauses (stored as HTML) |

### 4.6 `transactions`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| organization_id | UUID | FK | |
| contract_id | UUID | FK → contracts | |
| tenant_user_id | UUID | FK → users | Who paid |
| amount | DECIMAL(12,2) | NOT NULL | |
| payment_type | ENUM | NOT NULL | `cash \| bank_transfer \| mobile_money \| cheque` |
| payment_date | DATE | NOT NULL | Date of actual payment |
| gps_lat | DECIMAL(10,8) | NOT NULL | Captured at submission — immutable |
| gps_lng | DECIMAL(11,8) | NOT NULL | |
| status | ENUM | DEFAULT 'pending' | `pending \| receiver_confirmed \| owner_approved \| rejected` |
| receiver_user_id | UUID | FK → users, NULLABLE | Who confirmed receipt |
| receiver_confirmed_at | TIMESTAMP | NULLABLE | |
| owner_approved_at | TIMESTAMP | NULLABLE | |
| rejection_reason | TEXT | NULLABLE | |
| period_from | DATE | NOT NULL | Which rental period this covers |
| period_to | DATE | NOT NULL | |
| reference_number | VARCHAR(100) | NULLABLE | Bank ref / cheque number |

### 4.7 `transaction_proofs`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| transaction_id | UUID | FK → transactions | |
| proof_type | ENUM | NOT NULL | `image \| voice_note \| cnic \| signature` |
| file_url | TEXT | NOT NULL | S3/R2 URL |
| file_size_kb | INT | NOT NULL | |
| uploaded_at | TIMESTAMP | DEFAULT NOW() | |

### 4.8 `payment_splits`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| transaction_id | UUID | FK → transactions | |
| partner_user_id | UUID | FK → users | |
| calculated_amount | DECIMAL(12,2) | NOT NULL | System-calculated share |
| actual_amount | DECIMAL(12,2) | NOT NULL | Final amount (may differ if manually overridden) |
| override_reason | TEXT | NULLABLE | Required if `actual_amount ≠ calculated_amount` |
| overridden_by | UUID | FK → users, NULLABLE | Owner who performed the override |

### 4.9 `audit_logs`

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | |
| organization_id | UUID | |
| actor_user_id | UUID | Who performed the action |
| action | VARCHAR(100) | e.g. `CONTRACT_CREATED`, `TRANSACTION_APPROVED` |
| entity_type | VARCHAR(50) | Table name: `contracts`, `transactions`, etc. |
| entity_id | UUID | ID of the affected record |
| old_value | JSONB | State before change |
| new_value | JSONB | State after change |
| ip_address | VARCHAR(45) | |
| created_at | TIMESTAMP | **Immutable — never updated or deleted** |

---

## 5. Core Functional Requirements

### 5.1 Authentication & Onboarding

#### 5.1.1 Organization Registration (Owner)

1. Owner enters: organization name, phone number, sets 6-digit PIN
2. System sends OTP via SMS for phone verification
3. After verification: org record created, owner account created, JWT issued
4. Owner completes org profile: proof configuration, optional logo

#### 5.1.2 User Invitation

- Owner invites partners/tenants by phone number
- Invited user receives SMS with a one-time link or app invite code
- On first login: user sets own PIN, completes profile, uploads CNIC if required

#### 5.1.3 Authentication Flow

- Login: phone number + PIN → JWT access token (15 min) + refresh token (30 days)
- Biometric login (Face ID / Fingerprint) supported after first PIN login
- Failed PIN: account locked after 5 consecutive failures; unlock via OTP
- All tokens invalidated on logout and on PIN change

> **Note:** Passwords are NOT used. PIN (4–6 digits) is the only credential, appropriate for the target user base.

---

### 5.2 Land Management

#### 5.2.1 Create / Edit Land

| Field | Required | Validation |
|-------|----------|------------|
| Land Name / Title | Yes | Max 255 chars |
| Province / District | Yes | Dropdown from lookup table |
| GPS Location (tap on map) | Optional | Stored as lat/lng decimal |
| Human-Readable Address | Yes | Free text, max 500 chars |
| Total Acres | Yes | Positive decimal, max 99,999.999 |
| Status | Yes | Default: `available` |

#### 5.2.2 Partner Assignment

- Owner assigns partners to a land with ownership percentages
- **System validates:** sum of all partner percentages for a land must equal exactly **100%**
- Block save if total is not 100%
- Each partner can optionally have a maximum receivable limit per payment cycle
- A partner can be assigned to multiple lands with different percentages on each

---

### 5.3 Contract Management

#### 5.3.1 Contract Creation

| Field | Required | Notes |
|-------|----------|-------|
| Contract Title | Yes | Auto-suggest: `LandName + Year` |
| Land | Yes | Dropdown of available lands |
| Tenant | Yes | Must be existing user in org with `role = tenant` |
| Pricing Type | Yes | Per Acre or Fixed Amount |
| Rate / Amount | Yes | Auto-calculates total if per acre |
| Start Date | Yes | Cannot be in the past for new contracts |
| End Date | Yes | Must be after start date |
| Payment Frequency | Yes | Monthly / Quarterly / Bi-Annual / Annual / Custom |
| Custom Payment Dates | If custom | Array of specific due dates |
| Notes / Clauses | No | Rich text, stored as HTML |

#### 5.3.2 Contract Lifecycle States

| State | Description | Who Can Change |
|-------|-------------|----------------|
| `draft` | Created but not yet activated | Owner |
| `active` | Contract is running, payments expected | Owner (activate) |
| `expired` | End date has passed | System (cron job, auto) |
| `terminated` | Ended early by owner | Owner only (mandatory reason required) |

> ⚠️ **Deleting contracts is NOT allowed.** Use `terminated` status with a mandatory reason.

---

### 5.4 Transaction System

#### 5.4.1 Transaction Lifecycle

Every transaction moves through a strict 4-state workflow. No state can be skipped.

| Step | Status | Actor | Action Required | System Action |
|------|--------|-------|-----------------|---------------|
| 1 | `PENDING` | Tenant | Submits payment with proof + GPS | Notifies owner & receiver |
| 2 | `RECEIVER_CONFIRMED` | Receiver | Confirms they received the money | Notifies owner |
| 3 | `OWNER_APPROVED` | Owner | Reviews proof and approves | Creates payment splits, notifies partners |
| — | `REJECTED` | Owner | Rejects with mandatory reason | Notifies tenant; no splits calculated |

#### 5.4.2 Transaction Creation (Tenant)

- Tenant selects an active contract
- Selects payment period (which installment this covers)
- Enters: amount paid, payment type, date, optional reference number
- GPS auto-captured — cannot be manually entered or overridden
- Attaches required proofs as configured by the organization (see §5.6)
- Submits — status becomes `PENDING`, all edits are locked

> ⚠️ **Tenant CANNOT edit a submitted transaction.** Owner must reject it for the tenant to resubmit.

#### 5.4.3 Partial Payments

- System tracks total amount due per contract period
- Tenant can submit multiple transactions for the same period (partial payments)
- Dashboard shows: **Total Due**, **Total Paid**, **Outstanding Balance** per contract

---

### 5.5 Payment Split Engine

#### 5.5.1 Automatic Calculation

Triggered immediately after Owner approves a transaction.

```
FOR each partner IN land_partners WHERE land = contract.land:
  base_share = transaction.amount × (partner.ownership_percent / 100)

  IF partner.max_receivable_limit IS NOT NULL:
    partner_already_received = SUM(payment_splits WHERE partner AND period)
    remaining_limit = max_receivable_limit - partner_already_received
    actual_share = MIN(base_share, remaining_limit)
  ELSE:
    actual_share = base_share

  INSERT payment_splits (
    transaction_id, partner_id,
    calculated_amount = base_share,
    actual_amount = actual_share
  )
```

#### 5.5.2 Manual Override

- Owner can override any partner's `actual_amount` after calculation
- Override requires mandatory text reason
- Override is logged in `audit_logs` with old and new values
- System does **not** auto-redistribute excess — owner must adjust all affected partners manually

---

### 5.6 Proof & Verification System

#### 5.6.1 Proof Types

Each organization configures which proofs are required in org settings.

| Proof Type | Format | Notes |
|------------|--------|-------|
| Image / Photo | JPEG/PNG, max 5 MB | Photo of cash handover, bank receipt screenshot, etc. |
| Voice Note | AAC/MP3, max 2 min | Spoken confirmation of payment details |
| CNIC Copy | JPEG/PNG, max 2 MB | National ID scan for identity verification |
| Handwritten Signature | PNG (drawn in-app) | Tenant signs on screen |
| GPS Location | Auto-captured lat/lng | **Cannot be disabled — always recorded** |

#### 5.6.2 Rules

- At least one proof type (beyond GPS) must be enabled per org
- All enabled proof types are mandatory per transaction — no optional proofs
- Files uploaded directly from app to S3 using pre-signed URLs — API never handles file bytes
- Files are private — accessed only via time-limited signed URLs (expire after 1 hour)

---

### 5.7 Notification System

| Trigger Event | Notified Roles | Channel |
|---------------|----------------|---------|
| New transaction submitted | Owner, Receiver | Push + In-app |
| Transaction receiver confirmed | Owner | Push + In-app |
| Transaction approved | Tenant, Partners | Push + In-app + SMS |
| Transaction rejected | Tenant | Push + In-app + SMS |
| Contract expiring in 30 days | Owner | Push + In-app |
| Contract expired | Owner, Tenant | Push + In-app |
| Payment due in 3 days | Tenant | Push + SMS |
| Payment overdue | Owner, Tenant | Push + SMS (daily) |
| Manual split override | Affected Partner | Push + In-app |

> **Note:** SMS fallback is used for critical events (approvals, rejections, overdue) even if push notification fails.

---

### 5.8 Chat System

- Every contract has an attached chat thread
- Every transaction has an attached chat thread (for disputes/clarifications)
- Participants: all users assigned to that contract (owner + partners + tenant)
- Messages are plain text + optional single image attachment
- Messages **cannot be deleted or edited** (audit integrity)
- Chat history is permanent and included in exported contract PDFs
- Unread badge counts shown on contract/transaction cards

---

### 5.9 Reports & Analytics

| Report | Available To | Description |
|--------|--------------|-------------|
| Organization Summary | Owner, Accountant | Total lands, active contracts, outstanding dues, total collected YTD |
| Contract Statement | Owner, Accountant, Tenant | Full payment history for a contract with proof links |
| Partner Earnings | Owner, Accountant, Partner (own) | Splits received per partner per time period |
| Pending Approvals | Owner | All transactions awaiting action |
| Overdue Payments | Owner, Accountant | Contracts with missed payment cycles |
| Audit Log Report | **Owner only** | Full activity log with filters by user, date, action type |

- All reports exportable as PDF
- Date range filters available on all reports

---

## 6. API Endpoint Reference

**Base URL:** `https://api.landtheka.com
