# Bug Report

## Fixed Bugs

### Auth

1. **Duplicate username registration handling**
   - File: `app/routers/auth.py`
   - Issue: Registering an existing username within the same org returned a success response with the existing user's data instead of rejecting the request.
   - Fix: Raise `409 USERNAME_TAKEN` when the username already exists in the org.

2. **Refresh tokens not single-use**
   - File: `app/auth.py`, `app/routers/auth.py`
   - Issue: A refresh token could be reused multiple times to mint new token pairs (replay allowed), violating the single-use requirement.
   - Fix: Refresh tokens are invalidated after use; reuse now returns `401`.

3. **Access token expiry duration**
   - File: `app/auth.py`
   - Issue: Access tokens were effectively valid for far longer than the required 900 seconds.
   - Fix: Access token lifetime now uses `ACCESS_TOKEN_EXPIRE_MINUTES` (15 minutes) directly, expiring in exactly 900 seconds.

4. **Logout token revocation checking the wrong JWT claim**
   - File: `app/auth.py`
   - Issue: `revoke_access_token` stored the token's `jti` in the revoked set, but `get_token_payload` checked `payload["sub"]` (user id) against that set. Since `sub` and `jti` are never equal, a "logged out" access token continued to work until natural expiry.
   - Fix: `get_token_payload` now checks `payload.get("jti")` against the revoked set, so logout immediately invalidates the presented token.

### Bookings

5. **Missing ownership check in `get_booking`**
   - File: `app/routers/bookings.py`
   - Issue: Any member could view another member's booking by id; only org membership was checked, not ownership.
   - Fix: Added a check that non-admin callers may only view their own bookings; otherwise returns `404 BOOKING_NOT_FOUND`.

6. **Duplicate/unreachable code in `cancel_booking`**
   - File: `app/routers/bookings.py`
   - Issue: Leftover duplicate code after the refund logic left unreachable/duplicated statements in the cancellation path.
   - Fix: Removed the duplicate block during merge cleanup.

7. **Duplicate stats/notification call in `create_booking`**
   - File: `app/routers/bookings.py`
   - Issue: A leftover copy-pasted block caused `stats.record_create` and `notifications.notify_created` to run twice per booking, double-counting confirmed bookings and revenue in `/rooms/{id}/stats`.
   - Fix: Removed the duplicate block; each booking now updates stats/notifications exactly once.

8. **Booking minimum duration validation**
   - File: `app/routers/bookings.py`
   - Issue: Only the maximum duration was checked; durations below 1 hour were allowed.
   - Fix: Added a minimum duration check using `MIN_DURATION_HOURS`.

9. **Back-to-back booking overlap detection**
   - File: `app/routers/bookings.py`
   - Issue: The overlap check used `<=`, so bookings touching exactly at the boundary were incorrectly treated as conflicts.
   - Fix: Changed overlap logic to strict `<` comparisons, allowing back-to-back bookings.

10. **Booking list ordering**
    - File: `app/routers/bookings.py`
    - Issue: Results were sorted by descending `start_time` instead of ascending.
    - Fix: Changed ordering to ascending `start_time`, with ties broken by ascending `id`.

11. **Booking pagination offset**
    - File: `app/routers/bookings.py`
    - Issue: Pagination used `page * limit` as the offset, which skipped the first page's expected range.
    - Fix: Changed offset calculation to `(page - 1) * limit`.

12. **Booking list page size handling**
    - File: `app/routers/bookings.py`
    - Issue: The endpoint always used a hardcoded `.limit(10)` and ignored the caller's requested `limit`.
    - Fix: Changed to `.limit(limit)`.

13. **Booking detail start time field**
    - File: `app/routers/bookings.py`
    - Issue: The booking detail response overwrote `start_time` with `created_at`.
    - Fix: Changed to return `iso_utc(booking.start_time)`.

14. **Strict future booking start validation**
    - File: `app/routers/bookings.py`
    - Issue: Booking creation allowed a grace window that let `start_time` be slightly in the past.
    - Fix: `start_time` must now be strictly greater than the current time, with no grace window.

15. **Explicit end time ordering validation**
    - File: `app/routers/bookings.py`
    - Issue: Booking creation did not explicitly reject `end_time <= start_time`.
    - Fix: Added a validation error (`INVALID_BOOKING_WINDOW`) when `end_time` is not strictly after `start_time`.

16. **Timezone normalization for input datetimes**
    - File: `app/timeutils.py`
    - Issue: Offset-aware input datetimes had their `tzinfo` stripped without first converting to UTC, storing the wrong absolute moment in time.
    - Fix: Convert to UTC first, then strip `tzinfo` for naive storage.

17. **Reference code uniqueness under concurrency**
    - File: `app/routers/bookings.py`
    - Issue: Reference codes were not guaranteed unique under concurrent booking creation.
    - Fix: Added a uniqueness constraint / synchronization so codes remain unique under concurrent requests.

18. **Reference code generator race condition**
    - File: `app/services/reference.py`
    - Issue: `next_reference_code` read the counter, performed a simulated formatting delay, then wrote the incremented value back with no locking. Two concurrent booking creations could read the same counter value during the delay and both return the same reference code, violating the global uniqueness requirement. A stray trailing character after the return statement was also a syntax error.
    - Fix: Wrapped the read-delay-write sequence in a `threading.Lock`, and removed the stray trailing character.

19. **Refund threshold at 48 hours**
    - File: `app/routers/bookings.py`
    - Issue: The 100% refund condition used `> 48` instead of `>= 48` hours notice.
    - Fix: Changed the condition to `notice >= timedelta(hours=48)`.

20. **Refund policy for cancellations under 24 hours**
    - File: `app/routers/bookings.py`
    - Issue: Cancellations with less than 24 hours' notice incorrectly returned a 50% refund instead of 0%.
    - Fix: Corrected the final branch to return 0% refund for notice under 24 hours.

21. **Refund calculation rounding and ledger/response divergence**
    - File: `app/services/refunds.py`, `app/routers/bookings.py`
    - Issue: Two separate problems compounded each other: (a) `log_refund` computed the refund amount with `int(refund_dollars * 100)`, which truncates instead of rounding half-cents up as required; (b) the cancel endpoint independently recomputed the refund amount using Python's `round()` (which uses banker's rounding, not round-half-up) for the API response. Because the amount was computed twice, in two different ways, the value returned to the caller could diverge from the value stored in `RefundLog`, violating the requirement that they must be equal.
    - Fix: Introduced a single shared `calculate_refund_cents` helper using `Decimal` with `ROUND_HALF_UP`. Both `log_refund` and the cancel endpoint's response now use this same helper, guaranteeing correct rounding and exact agreement between the stored and returned amounts.

22. **Missing usage-report cache invalidation on booking creation**
    - File: `app/routers/bookings.py`
    - Issue: `create_booking` did not invalidate the cached usage report, so admins could see stale `confirmed_bookings`/`revenue_cents` data after a new booking was made.
    - Fix: Added `cache.invalidate_report(user.org_id)` to the end of `create_booking`.

### Admin

23. **Cross-org data leak in admin export**
    - File: `app/routers/admin.py`
    - Issue: `GET /admin/export` accepted a `room_id` query parameter and passed it straight to `generate_export`/`fetch_bookings_raw` without verifying the room belonged to the calling admin's organization. A `room_id` from a different org would return that org's booking data instead of a 404.
    - Fix: Added a check that the room exists and belongs to `admin.org_id` before generating the export; raises `404 ROOM_NOT_FOUND` otherwise.

### Concurrency / Rate Limiting

24. **Rate limiter race condition (bypassable limit)**
    - File: `app/services/ratelimit.py`
    - Issue: `record_and_check` read the user's request bucket, performed a simulated delay, then wrote the updated bucket back with no locking. Concurrent requests from the same user could read the same stale bucket and both write their own updated version, causing one request's timestamp to be silently dropped and the 20-requests-per-60-seconds limit to be bypassable under burst traffic.
    - Fix: Wrapped the read-delay-write sequence in a `threading.Lock`.

25. **Live room stats race condition**
    - File: `app/services/stats.py`
    - Issue: `record_create`/`record_cancel` read the current count/revenue, performed a simulated delay, then wrote back the updated values with no locking. Concurrent booking creations/cancellations on the same room could lose updates, leaving `/rooms/{id}/stats` inconsistent with actual bookings.
    - Fix: Wrapped both functions' read-delay-write sequence in a `threading.Lock`.

26. **Lock-ordering deadlock in notification service**
    - File: `app/services/notifications.py`
    - Issue: `notify_created` acquired `_email_lock` then nested `_audit_lock` inside it; `notify_cancelled` acquired `_audit_lock` then nested `_email_lock` inside it — the reverse order. A concurrent booking creation and cancellation could deadlock (each holding one lock while waiting on the other), hanging both request threads indefinitely.
    - Fix: Removed the nested locking; each function now acquires and releases each lock independently and sequentially, eliminating the circular wait.


- `48b28b8` — "remove duplicate column typo in models" (`app/models.py`)
- `22593f3` — "fix race condition in rate limiter" — appears to predate/overlap with fix #24 above; confirm no conflict between the two commits.
- `e94f17a2` — "invalidate availability cache when a booking is cancelled"
- `c1bd296` — "fix race conditions in booking creation and cancellation" (likely the booking-level lock)