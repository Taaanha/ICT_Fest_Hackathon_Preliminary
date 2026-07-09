# Bug Report

## Fixed Bugs

1. Access token expiry duration
   - File: `app/auth.py`
   - Issue: Access tokens were created with `15 * 60` minutes, so they lasted 15 hours instead of 15 minutes.
   - Fix: Changed access token lifetime to use `ACCESS_TOKEN_EXPIRE_MINUTES` directly.

2. Duplicate username registration handling
   - File: `app/routers/auth.py`
   - Issue: Registering an existing username in the same organization returned a success response with existing user data.
   - Fix: Changed the behavior to raise `409 USERNAME_TAKEN`.

3. Booking minimum duration validation
   - File: `app/routers/bookings.py`
   - Issue: Booking duration checked only the maximum limit and allowed durations below 1 hour.
   - Fix: Added the minimum duration check using `MIN_DURATION_HOURS`.

4. Back-to-back booking overlap detection
   - File: `app/routers/bookings.py`
   - Issue: The overlap check used `<=`, so bookings touching at the boundary were treated as conflicts.
   - Fix: Changed the overlap logic to use strict `<` comparisons.

5. Booking list ordering
   - File: `app/routers/bookings.py`
   - Issue: Booking list results were sorted by descending `start_time`.
   - Fix: Changed ordering to ascending `start_time`.

6. Booking pagination offset
   - File: `app/routers/bookings.py`
   - Issue: Pagination used `page * limit`, which skipped the first page correctly expected range.
   - Fix: Changed offset to `(page - 1) * limit`.

7. Booking detail start time field
   - File: `app/routers/bookings.py`
   - Issue: Booking detail response overwrote `start_time` with `created_at`.
   - Fix: Changed it to return `iso_utc(booking.start_time)`.

8. Refund threshold at 48 hours
   - File: `app/routers/bookings.py`
   - Issue: The 100% refund condition used `> 48` instead of `>= 48`.
   - Fix: Changed the condition to `notice_hours >= 48`.

9. Booking list page size handling
   - File: `app/routers/bookings.py`
   - Issue: Booking list always used `.limit(10)` and ignored the requested `limit`.
   - Fix: Changed it to `.limit(limit)`.

10. Timezone normalization for input datetimes
    - File: `app/timeutils.py`
    - Issue: Offset-aware datetimes had their timezone stripped without conversion, storing the wrong UTC moment.
    - Fix: Convert to UTC first, then remove `tzinfo` for storage.

11. Strict future booking start validation
    - File: `app/routers/bookings.py`
    - Issue: Booking creation allowed a 5-minute grace window for past start times.
    - Fix: Changed validation to require `start_time` to be strictly greater than `now`.

12. Explicit end time ordering validation
    - File: `app/routers/bookings.py`
    - Issue: Booking creation did not explicitly reject `end_time <= start_time`.
    - Fix: Added validation to require `end_time` to be strictly after `start_time`.
