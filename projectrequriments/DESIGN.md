
# Travel Booking Platform: Hotel Module Design (Phase 1)

This document defines the module layout, data model, booking flow, and API contract for the Phase 1 hotel booking module. It implements the requirements in `REQUIREMENTS.md`.

---

## 1. Module Boundaries

```
app/
  core/        config, security (JWT, password hashing), error handling, pagination
  auth/        register, login, current user
  hotels/      hotels, rooms, search, room listing
  bookings/    booking creation, cancellation, booking_nights, availability queries
  admin/       admin routers that call hotels and bookings services
```

**Dependency rule:** `bookings` depends on `hotels` (to read room price and capacity). `hotels` never imports `bookings`. Availability is computed from `booking_nights` inside the `bookings` module, and `hotels` requests it through a service function. Future cab and flight modules will be sibling packages that follow the same pattern.

---

## 2. Key Decisions

| Topic | Decision |
|---|---|
| Database | MySQL 8.0 (InnoDB) |
| Room model | Each room is one bookable unit with a `room_type` text field. No room-types table in Phase 1. |
| Money | Integer paise (4500 rupees = 450000). No floats. |
| Booking statuses | `CONFIRMED`, `CANCELLED`. Phase 2 adds `PENDING_PAYMENT`. |
| Stored price | A booking copies `nightly_rate_paise` and `total_paise` at creation time. |
| Stay definition | Nights `[check_in, check_out)`. Two bookings conflict if they share any night. |
| Checkout day | Free for the next guest, because the checkout night is not stored. |
| Cancelled bookings | Do not block dates. |
| Cancellation rule | Guest can cancel their own booking if check-in is at least 24 hours away (UTC). Cancelling sets status to `CANCELLED` and deletes the booking's `booking_nights` rows. The record is kept for history. |
| Refunds | Out of scope for Phase 1. |
| Deactivation | Deactivating a hotel or room blocks new bookings. Existing bookings are not cancelled automatically. |
| Admin creation | Created by a seed script or CLI command. Self-registration always creates a `GUEST`. |
| Non-owner access | A guest requesting another user's booking receives `404`, not `403`, so the API does not reveal that the booking    exists. |
| Timestamps | Stored in UTC. |

---

## 3. Entities

**User**
- `id`, `name`, `email` (unique), `password_hash`, `role` (`GUEST` or `ADMIN`), `created_at`

**Hotel**
- `id`, `name`, `city`, `address`, `description`, `is_active`, `created_at`

**Room**
- `id`, `hotel_id` (FK), `room_type`, `capacity`, `nightly_rate_paise`, `is_active`

**Booking**
- `id`, `user_id` (FK), `room_id` (FK), `check_in` (date), `check_out` (date), `guests`, `status`, `nightly_rate_paise`, `total_paise`, `idempotency_key`, `created_at`
- Unique: `(user_id, idempotency_key)`

**BookingNight**
- `id`, `room_id`, `stay_date`, `booking_id` (FK)
- Unique: `(room_id, stay_date)`. This is the double-booking guard.

**Relationships:** a hotel has many rooms; a user has many bookings; a room has many bookings; a booking has many booking nights (one per night stayed).

---

## 4. Validation Rules

- `check_out` must be after `check_in`.
- `check_in` must not be in the past (UTC date).
- Stay length must be 30 nights or fewer.
- `guests` must be at least 1 and at most the room's `capacity`.
- `Idempotency-Key` header is required on `POST /bookings`.

---

## 5. Booking Creation Flow

1. Validate the request (section 4).
2. Look up `(user_id, Idempotency-Key)`:
   - If a booking exists and the request body matches, return it with `200`.
   - If the body differs, return `422 IDEMPOTENCY_KEY_REUSED`.
3. Open a transaction.
4. Load the room. It must be active and belong to an active hotel. Otherwise return `409 ROOM_NOT_AVAILABLE`.
5. Insert one `booking_nights` row per night, ordered by `stay_date` to reduce deadlock risk.
6. Insert the booking row with the stored price (`nightly_rate_paise`, `total_paise`).
7. Commit.
   - On a duplicate-key error in step 5: roll back and return `409 ROOM_NOT_AVAILABLE`.
   - On a deadlock error: retry once, then return `409 ROOM_NOT_AVAILABLE`.

---

## 6. Cancellation Flow

1. Load the booking. It must belong to the caller, or return `404`.
2. If status is already `CANCELLED`, or check-in is less than 24 hours away (UTC), return `409 BOOKING_NOT_CANCELLABLE`.
3. In one transaction: set status to `CANCELLED` and delete the booking's `booking_nights` rows.
4. Return the updated booking with `200`.

---

## 7. Endpoints

All paths are under `/api/v1`. Every endpoint except register and login requires a JWT.

| Method | Path | Who | Purpose | Success |
|---|---|---|---|---|
| POST | `/auth/register` | Public | Create guest account | 201 |
| POST | `/auth/login` | Public | Return access token | 200 |
| GET | `/users/me` | Any | Current user profile | 200 |
| GET | `/hotels` | Any | Search by city, dates, guests, price; paginated and sortable | 200 |
| GET | `/hotels/{hotel_id}` | Any | Hotel details | 200 |
| GET | `/hotels/{hotel_id}/rooms` | Any | Rooms with availability for given dates | 200 |
| POST | `/bookings` | Guest | Create booking (requires `Idempotency-Key`) | 201 (200 on replay) |
| GET | `/bookings` | Guest | Own bookings, paginated | 200 |
| GET | `/bookings/{booking_id}` | Guest (owner) | One booking | 200 |
| POST | `/bookings/{booking_id}/cancel` | Guest (owner) | Cancel if eligible | 200 |
| POST | `/admin/hotels` | Admin | Create hotel | 201 |
| PATCH | `/admin/hotels/{hotel_id}` | Admin | Update fields, including `is_active` | 200 |
| POST | `/admin/hotels/{hotel_id}/rooms` | Admin | Create room | 201 |
| PATCH | `/admin/rooms/{room_id}` | Admin | Update fields, including `is_active` | 200 |
| GET | `/admin/bookings` | Admin | All bookings; filter by status; paginated | 200 |

Deactivation uses PATCH with `is_active`, not DELETE, so booking history is preserved.

---

## 8. Example

**Request**

```http
POST /api/v1/bookings
Authorization: Bearer <token>
Idempotency-Key: 7f3c2a9e-1b4d-4e8a-9c2f-5d6e7a8b9c0d
Content-Type: application/json

{
  "room_id": 42,
  "check_in": "2026-11-01",
  "check_out": "2026-11-04",
  "guests": 2
}
```

**Response (201)**

json
{
  "id": 981,
  "room_id": 42,
  "check_in": "2026-11-01",
  "check_out": "2026-11-04",
  "guests": 2,
  "status": "CONFIRMED",
  "nightly_rate_paise": 450000,
  "total_paise": 1350000,
  "created_at": "2026-10-26T09:15:00Z"
}


---

## 9. Error Codes

All errors use the shape `{"code": "...", "message": "..."}`.

| HTTP | Code | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | Missing or invalid token |
| 403 | `FORBIDDEN` | Authenticated but not an admin, on admin routes |
| 404 | `NOT_FOUND` | Resource missing, or not owned by the caller |
| 409 | `ROOM_NOT_AVAILABLE` | Overlapping night already booked, or room/hotel inactive |
| 409 | `BOOKING_NOT_CANCELLABLE` | Inside the 24-hour window, or already cancelled |
| 422 | `VALIDATION_ERROR` | Invalid input, including bad dates or guest count over capacity |
| 422 | `IDEMPOTENCY_KEY_REUSED` | Same key sent with a different body |

---

## 10. Concurrency Strategy Summary

- **Double booking:** unique `(room_id, stay_date)` on `booking_nights`, enforced by MySQL inside one transaction. No check-then-insert in application code.
- **Retries:** `Idempotency-Key` with unique `(user_id, idempotency_key)`.
- **Deadlocks:** consistent insert order by date, with one retry.



## 11. Open Items

- Search response shape and sort options are to be finalized in Session 3 or 4.
- Pagination defaults (page size, maximum) are to be set during API implementation.
