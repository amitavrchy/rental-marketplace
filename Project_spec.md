# Rental Marketplace (House & Car) — Project Spec

> Stack: **Next.js + NestJS + Prisma + PostgreSQL/PostGIS + Redis/BullMQ + Maplibre/Mapbox + Socket.IO**  
> Airbnb/Turo-style marketplace for homes and cars: geo search, availability calendar with **double-booking prevention**, dynamic pricing, deposits, reviews, host/renter verification.

---

## 1. Overview & Goals

Two listing verticals (`STAY`, `VEHICLE`) sharing booking core. Emphasis on geospatial queries, calendar integrity, pricing rules, and escrow-like deposit hold/release.

### Engineering goals
- PostGIS `geography` for radius search + map bounds.
- Booking overlap exclusion via Postgres range types / exclusion constraints.
- Dynamic pricing (seasonal, weekend, demand, length-of-stay discounts).
- Identity verification + damage deposit workflow.
- Messaging between host and renter.

---

## 2. Roles & Permissions

| Role | Permissions |
|------|-------------|
| `ADMIN` | Moderate listings, resolve disputes, force cancel |
| `HOST` | Listings, calendar, pricing, accept/decline (if manual), payouts |
| `RENTER` | Search, book, pay, review, claims |
| `SUPPORT` | Read-only + dispute tools |

Host must pass verification to publish.

---

## 3. Feature Modules

1. Auth + profiles + ID verification  
2. Listings (stay amenities / vehicle specs)  
3. Geo search & filters  
4. Availability calendar & blocks  
5. Pricing engine  
6. Booking & payments (hold deposit + rent)  
7. Check-in / check-out / inspections  
8. Reviews (two-sided)  
9. Messaging  
10. Host payouts  
11. Disputes & damage claims  
12. Admin moderation  

---

## 4. Advanced Business Logic

### 4.1 Double-booking prevention
Prefer Postgres:
```sql
-- booking period as tstzrange [start, end)
EXCLUDE USING gist (
  listing_id WITH =,
  period WITH &&
) WHERE (status IN ('PENDING','CONFIRMED','CHECKED_IN'))
```
Prisma: manage via raw migration; in app use transaction + lock listing advisory.

Flow: hold (`PENDING` + payment) → `CONFIRMED`. On payment fail delete/cancel pending quickly (TTL job).

### 4.2 Availability
- `CalendarBlock`: host blocks; maintenance.
- Min/max nights/days; preparation gap (`bufferHours`) between bookings.
- Vehicles: may allow hour-based rentals (`granularity=HOUR|DAY`).

Overlap check includes buffers:
`requestedStart - buffer < existingEnd && requestedEnd + buffer > existingStart`.

### 4.3 Dynamic pricing
Base nightly/hourly + rules sorted by priority:
- Weekend multiplier  
- Date range seasonal  
- Length-of-stay tiers (weekly/monthly discount)  
- Demand: occupancy rate last N days → surge cap  
- Custom date overrides (absolute price)

`quote(listingId, start, end)` returns line items: rent, cleaning, serviceFee, taxes, deposit; persist `PriceQuote` snapshot on booking.

### 4.4 Deposit
- Separate PaymentIntent or auth-hold; capture partial on damage claim; release after checkout + inspection window (e.g. 48h).

### 4.5 Edge cases
- Instant book vs request-to-book (host accept within X hours or auto-expire).  
- DST for stays: use listing timezone, nights counted by local dates.  
- Concurrent quotes → same slot: only one PENDING wins via exclusion constraint.  
- Host cancels: penalty + full refund + listing quality strike.  
- Renter no-show: policy-based refund; deposit rules.

---

## 5. State Machines

**Listing**: `DRAFT → IN_REVIEW → PUBLISHED → PAUSED | REJECTED | SUSPENDED`  
**Verification**: `UNVERIFIED → PENDING → VERIFIED | FAILED`  
**Booking**: `DRAFT_QUOTE → PENDING_PAYMENT → PENDING_HOST_ACCEPT → CONFIRMED → CHECKED_IN → COMPLETED | CANCELLED | EXPIRED`  
**Deposit**: `HELD → PARTIALLY_CAPTURED → RELEASED | CAPTURED`  
**Claim**: `OPEN → UNDER_REVIEW → APPROVED | DENIED | SETTLED`

---

## 6. Prisma Schema Draft

```prisma
model User {
  id String @id @default(cuid())
  email String @unique
  role String @default("RENTER")
  fullName String
  verificationStatus String @default("UNVERIFIED")
}
model Listing {
  id String @id @default(cuid())
  hostId String
  type String // STAY VEHICLE
  title String
  description String
  status String
  timezone String @default("Asia/Dhaka")
  // location
  address String?
  lat Float
  lng Float
  // store PostGIS geography via raw SQL column location geography(Point,4326)
  basePriceMinor Int
  currency String @default("BDT")
  cleaningFeeMinor Int @default(0)
  depositMinor Int @default(0)
  granularity String @default("DAY")
  bufferHours Int @default(0)
  instantBook Boolean @default(true)
  minDuration Int @default(1)
  maxDuration Int?
  metadata Json // amenities or vehicle specs
  @@index([type, status])
}
model CalendarBlock {
  id String @id @default(cuid())
  listingId String
  startAt DateTime
  endAt DateTime
  reason String?
}
model PricingRule {
  id String @id @default(cuid())
  listingId String
  type String
  priority Int @default(100)
  config Json
  startsOn DateTime? @db.Date
  endsOn DateTime? @db.Date
}
model PriceQuote {
  id String @id @default(cuid())
  listingId String
  renterId String
  startAt DateTime
  endAt DateTime
  breakdown Json
  totalMinor Int
  depositMinor Int
  expiresAt DateTime
}
model Booking {
  id String @id @default(cuid())
  listingId String
  renterId String
  status String
  startAt DateTime
  endAt DateTime
  // period tstzrange added in SQL migration
  currency String
  rentMinor Int
  feesMinor Int
  depositMinor Int
  totalMinor Int
  quoteId String?
  cancelReason String?
  @@index([listingId, startAt, endAt])
}
model Payment {
  id String @id @default(cuid())
  bookingId String
  kind String // RENT DEPOSIT
  provider String
  status String
  amountMinor Int
  providerRef String?
  idempotencyKey String @unique
}
model Review {
  id String @id @default(cuid())
  bookingId String @unique
  listingId String
  authorId String
  targetId String
  rating Int
  body String?
}
model Conversation {
  id String @id @default(cuid())
  bookingId String? @unique
  listingId String
}
model Message {
  id String @id @default(cuid())
  conversationId String
  senderId String
  body String
  createdAt DateTime @default(now())
}
model DamageClaim {
  id String @id @default(cuid())
  bookingId String
  status String
  amountRequestedMinor Int
  evidence Json
}
model Payout {
  id String @id @default(cuid())
  hostId String
  bookingId String @unique
  amountMinor Int
  status String
}
```

---

## 7. REST API

- `GET /search?bbox=...|lat&lng&radiusKm=&type=&start&end&priceMin&priceMax`  
- CRUD listings, calendar blocks, pricing rules  
- `POST /quotes` → `POST /bookings`  
- Payments webhooks  
- Host accept/decline  
- Check-in/out  
- Reviews after COMPLETED  
- Claims  
- Messaging endpoints + WS  

---

## 8. Real-time

- `conversation:{id}` messages  
- Host: `booking.request`  

---

## 9. Jobs

- Expire PENDING_PAYMENT / PENDING_HOST_ACCEPT  
- Auto-release deposits  
- Payout after checkout + claim window  
- Reindex listing geo (if denormalized)  
- Reminder before check-in  

---

## 10. Security

- Hide exact address until CONFIRMED (show approximate geo).  
- PII in verification docs encrypted / short-lived signed URLs.  
- Rate-limit search abuse.

---

## 11. Caching

- Redis cache search tiles short TTL; invalidate on publish/calendar change.  
- Quote cache not shared across users.

---

## 12. Testing

- Parallel booking same range → one success (exclusion constraint).  
- Buffer hours respected.  
- Pricing quote snapshot immutable after booking.  
- Instant vs manual accept expiry.  
- Geo radius contains expected fixtures (PostGIS test).

---

## 13. Folder Structure

```
rental-marketplace/
  Project_spec.md
  frontend/  # map search UX
  backend/
    prisma/migrations/...._postgis.sql
```

---

## 14. Env vars

`DATABASE_URL`, `REDIS_URL`, `JWT_*`, `APP_URL`, `CORS_ORIGINS`, `S3_*`, `MAP_TILES_TOKEN`, `STRIPE_*`, `SSLCOMMERZ_*`, `BKASH_*`, `DEPOSIT_RELEASE_HOURS`, `HOST_ACCEPT_HOURS`, `SEARCH_DEFAULT_RADIUS_KM`

---

## 15. Milestones

1. Auth + listings + PostGIS setup  
2. Search + calendar blocks  
3. Quotes + exclusion-constrained bookings  
4. Payments + deposits  
5. Messaging + reviews  
6. Claims + payouts  
7. Dynamic pricing polish + admin  

---

## 16. Acceptance Criteria

- [ ] Two overlapping CONFIRMED bookings impossible at DB level.  
- [ ] Map search returns only available listings for date range.  
- [ ] Exact address hidden until booking confirmed.  
- [ ] Deposit auto-releases when no claim filed.  
- [ ] Seasonal + weekend rules match fixture quotes.  
- [ ] Unverified host cannot publish.  
- [ ] Vehicle hourly booking granularity works with overlap rules.

---

## 17. Agent Notes

- Enable PostGIS extension in migration 0.  
- Use `ST_DWithin` for radius; `ST_MakeEnvelope` for bbox.  
- Keep money ints; durations computed in listing TZ.  
- Seed Dhaka area stays + cars with calendars.
