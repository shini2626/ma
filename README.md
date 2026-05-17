# FULL APP LOGIC & FLOW AUDIT REPORT

## 1. Audit Scope
- Inspected Java/Swing/MVC code across `view`, `controller`, `service`, `dao`, `model`, and `DatabaseManager`.
- Tested roles: `CLIENT`, `HOTEL`, `ADMIN`.
- Tested modules: auth, search, booking, cancellation, checkout, ratings, loyalty points, support, logs, reports, hotel stats, seed data.
- Ran `.\mvnw.cmd clean compile`: passed.
- Ran temporary service/DAO probes against a backed-up database, then restored the original database.
- Limitation: I did not manually click every Swing button; most findings are confirmed by code inspection plus service/DAO execution.

## 2. App Flow Map
- Auth flow: login/register through `AuthController` -> `AuthService` -> `UserDAO`.
- Client flow: search hotels -> view rooms -> create `PENDING` booking -> optionally use points -> view/cancel/rate/support.
- Hotel flow: view own hotel -> manage own rooms -> accept/reject/cancel bookings -> validate checkout -> view stats/reports/support.
- Admin flow: manage hotels/rooms/users/bookings/support/logs/reports.
- Booking lifecycle intended: `PENDING -> ACCEPTED -> CHECKOUT/COMPLETED`, with `REJECTED`/`CANCELLED` side paths.
- Availability blocking statuses: `PENDING`, `ACCEPTED`, `CHECKED_IN`, `CHECKOUT_PENDING_VALIDATION`.
- Ratings allowed only when booking is `COMPLETED` and checkout was validated.
- Loyalty: points can be used during booking; completed stays earn points; ratings grant bonus points.

## 3. Confirmed Logic Problems

### [ISSUE-01] Admin Can Destroy Booking State Machine
- Severity: Critical
- Area: Booking / Admin
- Roles affected: Admin, Client, Hotel
- Reproduction steps:
  1. Pick an existing `COMPLETED` booking.
  2. Call/admin-use cancel booking.
  3. Call/admin-use confirm booking again.
- Expected behavior: Completed/cancelled bookings should not be moved back to active states.
- Actual behavior: Probe confirmed `COMPLETED -> CANCELLED -> ACCEPTED`.
- Why this is logically a problem: Completed stays can become active again after revenue, commission, points, and ratings already exist.
- Likely root cause: `AdminService.cancelBooking()` and `confirmBooking()` do not validate allowed source statuses.
- Suspected files/classes involved: `AdminService`, `AdminController`, `BookingManagementPanel`, `BookingDAO`.
- Suggested fix direction: Centralize booking transition validation and make admin follow the same state machine.

### [ISSUE-02] Hotel Can Validate Checkout Before Stay Happens
- Severity: Critical
- Area: Booking / Checkout / Loyalty / Commission
- Roles affected: Hotel, Client, Admin
- Reproduction steps:
  1. Create a future booking.
  2. Accept it as hotel.
  3. Immediately validate checkout.
- Expected behavior: Checkout validation should only be possible at/after checkout date.
- Actual behavior: Probe confirmed future booking became `COMPLETED`.
- Why this is logically a problem: Points, commission, revenue, and rating eligibility can happen before the stay.
- Likely root cause: `HotelAccountService.validateCheckout()` checks status only, not dates.
- Suspected files/classes involved: `HotelAccountService`, `HotelDashboardPanel`, `BookingDAO`.
- Suggested fix direction: Require current date >= checkout date, or introduce real check-in/checkout transition rules.

### [ISSUE-03] Seed Data Contains Active Double Bookings
- Severity: Critical
- Area: Seed Data / Availability / Demo Trust
- Roles affected: All
- Reproduction steps:
  1. Initialize database.
  2. Query active overlapping bookings by room/user.
- Expected behavior: Seeded active bookings should respect availability rules.
- Actual behavior: Restored DB has `22` overlapping active room-booking pairs and `56` overlapping active user-booking pairs.
- Why this is logically a problem: Demo data violates the app’s own booking rules and makes reports/search impossible to trust.
- Likely root cause: `DatabaseManager.seedAdditionalBookings()` inserts patterned bookings without overlap checks.
- Suspected files/classes involved: `DatabaseManager`.
- Suggested fix direction: Regenerate seed bookings using the same overlap rules as `BookingService`.

### [ISSUE-04] Loyalty Points Are Not Refunded On Cancellation/Rejection
- Severity: High
- Area: Loyalty / Booking Cancellation
- Roles affected: Client, Hotel, Admin
- Reproduction steps:
  1. Create a booking using 50 points.
  2. Points are deducted immediately.
  3. Cancel/reject the booking.
- Expected behavior: If a booking never becomes a completed stay, used points should be refunded unless an explicit penalty rule applies.
- Actual behavior: Probe confirmed points stayed deducted after hotel no-show cancellation; code has no refund path for client/admin/hotel cancellation.
- Why this is logically a problem: Clients can lose loyalty points on bookings that never produce a stay.
- Likely root cause: `LoyaltyService.usePoints()` runs at booking creation; cancellation services never reverse it.
- Suspected files/classes involved: `BookingService`, `LoyaltyService`, `LoyaltyDAO`, `HotelAccountService`, `AdminService`.
- Suggested fix direction: Add point reservation/refund logic, or only finalize point usage when booking is accepted/completed.

### [ISSUE-05] Support Compensation Update Is Not Atomic
- Severity: High
- Area: Support / Loyalty / Admin
- Roles affected: Admin, Client, Hotel
- Reproduction steps:
  1. Pick a hotel support ticket.
  2. Admin attempts reply with points compensation.
  3. Compensation fails because only clients can receive points.
- Expected behavior: Entire update should fail or roll back.
- Actual behavior: Probe confirmed ticket changed to `RESOLVED` with new response even though compensation failed.
- Why this is logically a problem: Admin sees an error but the ticket was still modified.
- Likely root cause: `SupportService.updateTicket()` updates ticket before validating/granting compensation, with no transaction.
- Suspected files/classes involved: `SupportService`, `SupportTicketDAO`, `LoyaltyService`.
- Suggested fix direction: Validate compensation first and wrap ticket update + points grant in one transaction.

### [ISSUE-06] Room Can Be Deactivated While Active Booking Exists
- Severity: High
- Area: Hotel Room Management / Booking
- Roles affected: Hotel, Admin, Client
- Reproduction steps:
  1. Pick a room with active `PENDING`/`ACCEPTED` booking.
  2. Hotel deactivates the room.
- Expected behavior: App should block deactivation or warn about active bookings.
- Actual behavior: Probe confirmed active room became inactive while active booking remained.
- Why this is logically a problem: Client has a valid stay in a room the hotel has removed from availability.
- Likely root cause: `HotelAccountService.setOwnRoomActive()` and admin room activation methods do not check active bookings.
- Suspected files/classes involved: `HotelAccountService`, `AdminService`, `RoomDAO`.
- Suggested fix direction: Block room deactivation while active bookings exist, or require migration/cancellation.

### [ISSUE-07] Reports Revenue Ignores Point Discounts
- Severity: High
- Area: Reports / Loyalty / Revenue
- Roles affected: Admin, Hotel
- Reproduction steps:
  1. Inspect revenue totals for revenue statuses.
  2. Compare `total_price` vs `final_price_after_points`.
- Expected behavior: Revenue should consistently use final paid value after points discount.
- Actual behavior: Restored DB showed `total=90140.00`, `final=89340.00`, diff `800.00`.
- Why this is logically a problem: Reports overstate revenue when loyalty points are used.
- Likely root cause: `BookingDAO.revenuePerHotel()`, `roomRevenueRanking()`, `hotelStats()` use `total_price`.
- Suspected files/classes involved: `BookingDAO`, `ReportService`, dashboards.
- Suggested fix direction: Use `COALESCE(NULLIF(final_price_after_points, 0), total_price)` everywhere revenue means paid value.

### [ISSUE-08] Hotel “Total Revenue” Counts Accepted Future Bookings
- Severity: Medium / High
- Area: Reports / Hotel Dashboard
- Roles affected: Hotel, Admin
- Reproduction steps:
  1. Open hotel stats or inspect `hotelStats()`.
  2. Observe revenue statuses include `ACCEPTED`, `CHECKED_IN`, `CHECKOUT_PENDING_VALIDATION`, `COMPLETED`.
- Expected behavior: Completed/paid revenue should be separate from expected/upcoming revenue.
- Actual behavior: Accepted future bookings are counted as revenue.
- Why this is logically a problem: Hotel can see money as earned before checkout/payment.
- Likely root cause: `REVENUE_STATUSES` is reused for expected and actual revenue.
- Suspected files/classes involved: `BookingDAO`, `ReportService`, `HotelDashboardPanel`.
- Suggested fix direction: Split “Expected Revenue” and “Completed Revenue”.

### [ISSUE-09] Search Service Allows Past-Date Searches
- Severity: Medium
- Area: Search / Date Validation
- Roles affected: Client
- Reproduction steps:
  1. Call hotel search with dates before today.
- Expected behavior: Past search should be blocked consistently at service level.
- Actual behavior: Probe confirmed service search allowed past dates.
- Why this is logically a problem: UI blocks it, but business layer does not; other callers can get meaningless historical availability.
- Likely root cause: `HotelService.searchHotels()` and `RoomService.getAvailableRooms()` only validate date order.
- Suspected files/classes involved: `HotelService`, `RoomService`, `SearchPanel`.
- Suggested fix direction: Move “check-in cannot be before today” into service validation.

### [ISSUE-10] Logout Is Not Logged
- Severity: Low / Medium
- Area: Logs / Audit
- Roles affected: Admin
- Reproduction steps:
  1. Login/logout.
  2. Check `audit_logs`.
- Expected behavior: Logout should be logged because docs/requirements say login/logout are important actions.
- Actual behavior: Restored DB has `0` `LOGOUT` entries.
- Why this is logically a problem: Session history is incomplete.
- Likely root cause: `MainFrame.logout()` does not call audit service.
- Suspected files/classes involved: `MainFrame`, `AuthService`, `AuditService`.
- Suggested fix direction: Add logout audit event.

### [ISSUE-11] Ratings Can Become Invalid If Admin Mutates Booking Later
- Severity: High
- Area: Ratings / Admin / Booking
- Roles affected: Client, Admin, Hotel
- Reproduction steps:
  1. Complete and rate a booking.
  2. Admin cancels or re-accepts that completed booking.
- Expected behavior: Rated/completed bookings should be immutable or require special reversal logic.
- Actual behavior: Admin state mutation can make rating attached to non-completed booking.
- Why this is logically a problem: Reports/rating averages can include ratings for bookings no longer completed.
- Likely root cause: Admin state changes bypass rating/checkout/commission consistency rules.
- Suspected files/classes involved: `AdminService`, `RatingService`, `BookingDAO`.
- Suggested fix direction: Lock completed/rated bookings from normal admin status mutation.

### [ISSUE-12] Admin Can Create New Hotel Account Over Existing Hotel Link
- Severity: Medium
- Area: Admin / Hotel Accounts
- Roles affected: Admin, Hotel
- Reproduction steps:
  1. Add hotel account for hotel that already has `hotel_user_id`.
- Expected behavior: App should block or confirm replacing the existing hotel account.
- Actual behavior: Code overwrites `hotels.hotel_user_id`, orphaning old hotel user.
- Why this is logically a problem: Existing hotel user loses hotel access silently.
- Likely root cause: `AdminService.addHotelUser()` does not check existing link.
- Suspected files/classes involved: `AdminService`, `UserManagementPanel`, `HotelDAO`.
- Suggested fix direction: Enforce one hotel account per hotel unless explicit replacement flow exists.

### [ISSUE-13] Admin User Creation Is Not Audited
- Severity: Medium
- Area: Logs / Admin
- Roles affected: Admin
- Reproduction steps:
  1. Create admin user through admin service.
- Expected behavior: Creating privileged account must be logged.
- Actual behavior: `addAdminUser()` inserts user but writes no audit log.
- Why this is logically a problem: Privilege changes are invisible in logs.
- Likely root cause: Missing audit call in `AdminService.addAdminUser()`.
- Suspected files/classes involved: `AdminService`.
- Suggested fix direction: Log admin creation and include actor.

## 4. Strongly Suspected Problems
- Admin can disable own account or all admins: `setUserActive()` has no self-protection or last-admin protection. This could lock the platform.
- Booking point usage is not fully transactional: booking commit happens before `loyaltyService.usePoints()`. If point deduction fails after booking insert, booking may exist without wallet deduction.
- Support compensation duplicate prevention is fragile: `hasSupportCompensation()` searches description text with `LIKE "%support ticket #id%"` instead of storing a structured related ticket id.
- `CHECKED_IN` and `CHECKOUT_PENDING_VALIDATION` are statuses but no clear UI action moves bookings into those states. The lifecycle exists in enum but is not truly modeled in the app.
- Controller-level role enforcement is incomplete for many admin actions. UI hides pages, but service/controller methods often trust the caller.

## 5. Workflow Limitations / UX-Logic Frictions
- Client points are deducted when booking is only `PENDING`. This feels harsh because hotel may reject later.
- Hotel “no-show” cancellation is based on typing text containing `no show`. This is fragile and language-dependent.
- Admin booking management has “Accept Pending” but can cancel almost any non-cancelled booking from UI, including completed/rejected states.
- Client booking history shows `Total`, not clearly original vs points discount vs cash due.
- Support sender can see admin response, but no detailed conversation/history model exists.
- Reports mix expected revenue and completed revenue, making dashboard numbers hard to trust.
- Seed data has invalid overlaps, so demo dashboards can appear populated but logically wrong.

## 6. Inconsistent Rules / Contradictions
- UI blocks past search, but service allows it.
- Booking creation blocks overlapping active bookings, but seed data contains many overlaps.
- Commission uses final paid price after points, but reports often use original total price.
- Ratings require completed checkout, but admin can later change rated booking status.
- Booking statuses include `CHECKED_IN` and `CHECKOUT_PENDING_VALIDATION`, but the UI mostly jumps from `ACCEPTED` to `COMPLETED`.
- Admin logs require admin access, but many other admin actions do not enforce admin role in service/controller.

## 7. Data Consistency Findings
- Restored DB counts: `users=45`, `hotels=10`, `rooms=55`, `bookings=83`, `ratings=20`, `tickets=12`, `logs=66`.
- No invalid ratings in restored seed initially: `0`.
- No missing hotel-account links: `0`.
- No active bookings on inactive rooms/hotels initially: `0`.
- Active room overlaps in seed: `22`.
- Active user overlaps in seed: `56`.
- Revenue mismatch due to point discounts: `800 MAD` difference between original totals and final paid totals.
- Logout logs: `0`.

## 8. Highest Priority Fix Order
1. Fix admin booking state machine mutation.
2. Block future checkout validation.
3. Clean/regenerate overlapping seed bookings.
4. Add point refund/reservation logic for cancellations/rejections.
5. Make support compensation transactional.
6. Prevent deactivating rooms/hotels with active bookings.
7. Standardize revenue calculations on final paid amount.
8. Split expected vs completed revenue.
9. Add service-level past-date validation for search.
10. Add missing audit logs for logout and admin creation.

## 9. Suggested Categories for the Next Fix Prompt
- Booking state machine fixes
- Admin permission and transition guards
- Checkout/date validation fixes
- Cancellation and loyalty refund fixes
- Support compensation transaction fixes
- Reports/revenue consistency fixes
- Seed data cleanup fixes
- Room/hotel deactivation guardrails
- Audit log coverage fixes
- Status lifecycle completion fixes

## 10. Test Gaps / What Could Not Be Fully Verified
- I did not manually click every redesigned Swing screen.
- I did not test concurrent booking from two live app instances; overlap protection was inspected and tested through service-level logic.
- I did not verify every individual visual table refresh after every mutation.
- I did not test every possible support category/status combination manually.
- I did not perform long-running usage tests with many repeated logins/logouts.

## 11. Short Final Verdict
The app has a strong feature set and compiles cleanly, but it is not logically demo-safe yet.

The biggest risks are booking state corruption, future checkout completion, invalid seed double-bookings, inconsistent revenue calculations, and loyalty points being deducted without reliable refund logic.

It is close to demo-ready visually and structurally, but the business logic needs a focused state-machine and data-consistency pass before it can be trusted in front of a professor or real user.
