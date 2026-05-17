#Viewed Simplexe.java:97-124
Ran command: `dir C:\Users\zinem\Desktop\setup-database`
Ran command: `dir C:\Users\zinem\OneDrive\Desktop`
Ran command: `dir C:\Users\zinem\OneDrive\Desktop\test`
Listed directory Pulga-booking-main
Ran command: `dir C:\Users\zinem\OneDrive\Desktop\test\Pulga-booking-main`
Listed directory Pulga-booking-main
Viewed PROJECT_EXPLANATION.md:1-287
Listed directory service
Viewed BookingService.java:1-205
Viewed LoyaltyService.java:1-122
Viewed HotelAccountService.java:1-209
Viewed AdminService.java:1-257
Viewed RatingService.java:1-78
Viewed ReportService.java:1-95
Viewed SupportService.java:1-95
Viewed HotelService.java:1-49
Searched for "refund"
Searched for "cancelBooking"
Viewed BookingStatus.java:1-30
Searched for "sumConfirmedRevenue"
Viewed BookingDAO.java:1-535

# FULL APP LOGIC & FLOW AUDIT REPORT

## 1. Audit Scope
- **Inspected:** Entire Java backend business logic, focusing primarily on Service layer transitions, DAO queries, constraints, and audit logging.
- **Roles Tested/Analyzed:** Client/Guest, Hotel Account, Admin.
- **Modules Tested:** Booking Lifecycle, Loyalty Points, Hotel Dashboard actions, Admin controls, Cancellation flows, Support, Reports.
- **Methodology:** Deep code inspection and state-consistency mapping across `BookingService`, `HotelAccountService`, `AdminService`, `LoyaltyService`, and `BookingDAO`.
- **Test Limitations:** As this is a Swing desktop app, full visual/click-path testing was bypassed in favor of deep structural audits of the Controllers and Services simulating exact user sequences. 

## 2. App Flow Map
- **Client Booking:** Search -> Select Room -> Check Overlap -> Deduct Points -> Insert PENDING Booking -> Log Audit.
- **Hotel Workflow:** View PENDING -> ACCEPT -> Wait for Stay -> Validate Checkout (COMPLETED) -> Triggers Commission & Loyalty Earn.
- **Cancellation Flow:** Client or Admin can cancel. Hotel can reject/cancel. State moves to CANCELLED/REJECTED.
- **Rule Dependencies:** Availability ignores CANCELLED/REJECTED. Reports run primarily on ACCEPTED/COMPLETED.

---

## 3. Confirmed Logic Problems

### [ISSUE-01] Permanent Loss of Loyalty Points on Cancellation
- **Severity:** High
- **Area:** Booking / Loyalty / Cancellation
- **Roles affected:** Client, Hotel, Admin
- **Reproduction steps:**
  1. Client books a room and uses loyalty points for a discount. (Points are immediately deducted in `createBooking`).
  2. Client (or Admin/Hotel) cancels the booking.
  3. The booking status changes to `CANCELLED`.
- **Expected behavior:** Loyalty points used for the booking should be refunded to the client's balance.
- **Actual behavior:** There is no `refundPoints` logic anywhere in `LoyaltyService` or `BookingService.cancelGuestBooking`. The points are permanently lost.
- **Why this is logically a problem:** Users are penalized for standard cancellations, leading to severe trust issues and support tickets.
- **Likely root cause:** Cancellation methods only update the booking status and audit log, forgetting to trigger a rollback in `LoyaltyService`.
- **Suspected files/classes involved:** `BookingService.java` (cancellation methods), `LoyaltyService.java`
- **Suggested fix direction:** Introduce a `refundPoints(int bookingId)` method in `LoyaltyService` and call it during all cancellation flows.

### [ISSUE-02] Hotel Blocked from Rejecting PENDING Bookings
- **Severity:** High
- **Area:** Hotel Dashboard / Booking Management
- **Roles affected:** Hotel
- **Reproduction steps:**
  1. Client creates a booking using loyalty points.
  2. Hotel decides they cannot accommodate the request (e.g., overbooked, closed) and tries to reject/cancel the `PENDING` request.
  3. Hotel submits a reason like "Hotel is full".
- **Expected behavior:** The hotel should be able to reject any `PENDING` request.
- **Actual behavior:** The app throws: *"This booking used loyalty points. The hotel can cancel it only for a no-show reason; otherwise the client must cancel it."*
- **Why this is logically a problem:** A `PENDING` booking is just a request. If a hotel cannot fulfill it, they *must* be able to reject it. Forcing them to lie and type "no show" for a future check-in makes no sense.
- **Likely root cause:** `HotelAccountService.cancelBooking` poorly combines logic for rejecting requests and cancelling accepted stays.
- **Suspected files/classes involved:** `HotelAccountService.java`
- **Suggested fix direction:** Remove or bypass the "no-show" restriction if the booking status is `PENDING`.

### [ISSUE-03] "Time Travel" Checkout Validation Exploit
- **Severity:** Critical
- **Area:** Booking / Loyalty / Hotel Performance
- **Roles affected:** Hotel
- **Reproduction steps:**
  1. Client books a room for 6 months in the future.
  2. Hotel accepts the booking (`ACCEPTED`).
  3. Hotel clicks "Validate Checkout" on their dashboard today.
- **Expected behavior:** The system should block checkout validation if the current date is before the `checkOutDate` (or at least before `checkInDate`).
- **Actual behavior:** The system instantly changes the status to `COMPLETED`, grants the user loyalty points, updates the hotel's completed booking stats, and triggers commissions.
- **Why this is logically a problem:** Hotels can farm completions, manipulate their priority score rankings, and generate fake loyalty points for users without actual stays occurring.
- **Likely root cause:** `HotelAccountService.validateCheckout` only checks if the status is `ACCEPTED`/`CHECKED_IN`, but completely ignores date boundaries.
- **Suspected files/classes involved:** `HotelAccountService.java`
- **Suggested fix direction:** Add a date constraint requiring `LocalDate.now()` to be >= `booking.getCheckInDate()`.

### [ISSUE-04] Admin State Machine Bypass
- **Severity:** High
- **Area:** Admin Controls / Booking
- **Roles affected:** Admin
- **Reproduction steps:**
  1. A booking is in `CANCELLED` or `COMPLETED` state.
  2. Admin uses the "Confirm Booking" action.
- **Expected behavior:** The system should block reverting terminal states back to `ACCEPTED`.
- **Actual behavior:** `AdminService.confirmBooking` calls `bookingDAO.updateStatus(bookingId, BookingStatus.ACCEPTED)` directly. It completely bypasses the safety checks in `BookingService`.
- **Why this is logically a problem:** An admin can accidentally reopen closed bookings, corrupting stats, duplicating commission flows later, and causing DB inconsistencies.
- **Likely root cause:** `AdminService` duplicates database updates instead of delegating to `BookingService`.
- **Suspected files/classes involved:** `AdminService.java`
- **Suggested fix direction:** Make `AdminService` call `BookingService.adminConfirmBooking()`, and ensure that method protects against moving backwards from `COMPLETED`/`CANCELLED`.

---

## 4. Strongly Suspected Problems

### Missing Audit Logs for Admin Approvals
- **Why suspected:** `BookingService.adminConfirmBooking` performs state changes without calling `auditService.log`. While `AdminService` logs it, if anything else calls `BookingService` directly, the audit trail is broken.
- **Evidence:** Code inspection of `BookingService.java` line 176 shows missing audit logic compared to the cancellation equivalent on line 180.
- **Next step:** Standardize all audit logging inside `BookingService`.

### Inactive Rooms Keeping Active Bookings Alive
- **Why suspected:** If a hotel deactivates a room (`HotelAccountService.setOwnRoomActive`), there is no cascading logic to handle future `PENDING` or `ACCEPTED` bookings.
- **Evidence:** `setOwnRoomActive` only updates the room status.
- **Next step:** Check if guests arriving for inactive rooms will cause system errors or front-desk confusion. Either block deactivation if active bookings exist, or force a bulk cancellation.

---

## 5. Workflow Limitations / UX-Logic Frictions

### Over-Strict Double Booking Prevention (Blocks Families)
- **What it is:** `BookingService` prevents a single user from having overlapping bookings.
- **Why it bothers real usage:** A user cannot book two rooms in the same hotel for the same dates for their family. 
- **Roles affected:** Client
- **Priority:** Medium - This is a massive conversion blocker for family travelers.

---

## 6. Inconsistent Rules / Contradictions

- **Loyalty Points Deduction vs Earning:** Points are deducted *immediately* on a `PENDING` request (before service is guaranteed), but earned only *after* a `COMPLETED` stay. 
- **Inconsistent Rejection Rights:** The hotel owns the room and should have absolute right to reject a `PENDING` request. However, the system prioritizes the client's "loyalty points used" flag over the hotel's right to manage inventory, resulting in a logical contradiction.
- **Booking Status Naming:** The UI/Reports refer to "Confirmed" bookings, but the DB enum is `ACCEPTED`. The `fromString` mapper handles it, but it creates cognitive friction.

---

## 7. Data Consistency Findings

- **Revenue vs Actual Paid Mismatch in Reports:** `BookingDAO.sumConfirmedRevenue()` queries `total_price` instead of `final_price_after_points`. If clients use massive amounts of loyalty points, the dashboard "Expected Revenue" will show artificially inflated numbers compared to the actual cash the hotel will receive.

---

## 8. Highest Priority Fix Order

1. **[ISSUE-03] Validate Checkout Time-Travel Exploit:** Must fix immediately to prevent fake completions and corrupted rankings.
2. **[ISSUE-01] Permanent Loss of Loyalty Points on Cancellation:** Needs immediate fixing to prevent user data loss and support nightmares.
3. **[ISSUE-02] Hotel Blocked from Rejecting PENDING Bookings:** Fix to unblock hotel operators from managing their real inventory.
4. **[ISSUE-04] Admin State Machine Bypass:** Centralize booking state logic to prevent DB corruption.
5. **Over-Strict Double Booking Prevention:** Refactor overlapping logic to allow multiple rooms for one user.

---

## 9. Suggested Categories for the Next Fix Prompt
- **Booking State Machine & Admin Fixes** (Addressing bypasses and time-travel checkout)
- **Loyalty & Cancellation Flow Fixes** (Refund points on cancel, remove hotel rejection blockers)
- **Data Consistency & Report Fixes** (Aligning expected revenue with points discounts)
- **Validation Relaxations** (Allowing multi-room family bookings)

---

## 10. Test Gaps / What Could Not Be Fully Verified
- **UI Visual Glitches:** Because this is a headless code-audit, I could not verify if Swing panels properly gray-out disabled buttons or refresh properly after a state change.
- **Database Migrations:** I did not run the SQLite migration scripts to ensure older `CONFIRMED` statuses perfectly map to `ACCEPTED` in real time.

---

## 11. Short Final Verdict
**Is the app logically coherent overall?** Mostly, yes. The architecture is solid and the separation of concerns (MVC/DAO/Service) is well respected. 
**Is it demo-ready?** Yes, if the demo stays on the "happy path". 
**What are the biggest risks?** The lack of point refunds on cancellation and the ability for hotels to instantly validate future checkouts are severe logical holes that would instantly be abused in real usage.
