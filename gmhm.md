# Hotel Booking Desktop Application - Project Explanation

## 1. Project Overview

This project is a Java desktop application for hotel booking. A client can register, log in, search hotels, view rooms, book a room, view booking history, cancel with a required reason, create support tickets, and rate a hotel after checkout validation. A hotel account can manage only its own hotel rooms and bookings. An admin supervises users, hotels, hotel accounts, bookings, support tickets, audit logs, ratings, and reports.

The application is inspired by the business logic of hotel reservation platforms, but it uses original names, original screens, and simple Swing components suitable for a university project.

## 2. Why This Stack Was Chosen

The project uses plain Java and Swing because they are part of the standard desktop Java ecosystem and are easy to present in a classroom. SQLite is used because it stores data in one local file and does not require a server. JDBC is used because it shows how Java communicates directly with a relational database.

Maven is used to manage the SQLite JDBC dependency and to compile/run the project consistently.

## 3. Full Package And File Structure

The code is organized under `src/main/java/com/hotelbooking`.

- `model`: simple domain objects such as `User`, `Hotel`, `Room`, `Booking`, `Rating`, `SupportTicket`, and `AuditLog`.
- `model.enums`: enums for roles, booking statuses, and room types.
- `dao`: database access classes. SQL is placed here, not in Swing views.
- `service`: business rules such as login, booking, availability, pricing, and reports.
- `controller`: classes called by Swing views. Controllers connect UI events to services.
- `view`: Swing frames and panels.
- `util`: reusable helper classes for passwords, dates, validation, dialogs, and UI styling.

## 4. Explanation Of Important Classes

`Main` starts the application. It initializes the SQLite database, applies the Swing theme, and opens `LoginFrame`.

`DatabaseManager` creates the database tables and seed data. It also gives DAO classes JDBC connections.

`AuthService` handles login and registration rules. `BookingService` handles the most important booking rules, including date validation, capacity checks, price calculation, and transactional double-booking prevention.

`AdminService` groups admin operations for hotels, rooms, users, hotel accounts, and bookings. `HotelAccountService` enforces hotel ownership rules. `SupportService`, `RatingService`, and `AuditService` manage reclamations, checkout-based ratings, and structured logs. `ReportService` calculates summary and ranking values used in dashboards.

`TableSortUtil` is a small Swing helper that installs `TableRowSorter` on list screens. It compares ID, money, points, dates, percentages, enums, and text with the correct data meaning instead of plain string order.

## 4.1 Redesigned UI Architecture

The frontend now uses a shared dashboard visual system inspired by a professional management dashboard.

Important UI classes:

- `DashboardUI`: reusable Swing builders for the dark sidebar, top header, KPI cards, boxed widgets, compact tables, and profile/info cards.
- `UITheme`: shared colors, buttons, cards, and table styling.
- `AdminDashboardPanel`: admin shell with sidebar navigation, KPI dashboard, recent bookings, support snapshot, rankings, and logs snapshot.
- `HotelDashboardPanel`: hotel shell with performance KPIs, hotel profile, recent booking requests, room rankings, rooms page, booking page, support page, and reports page.
- `GuestDashboardPanel`: client shell with client profile/info card, loyalty points, spending, booking summary, rewards, support snapshot, and real pages for search, bookings, loyalty, and support.

The dashboard home page for each role is intentionally compact and no-scroll. Full lists remain in their dedicated pages so the main dashboard can fit in a normal desktop window.

The redesign preserves backend compatibility because it reuses the existing controllers and existing workflow panels. Navigation only changes which real panel is visible in the `CardLayout`.

## 5. Database Creation And Schema Explanation

The database file is `hotel_booking.db` in the project root. On first launch, `DatabaseManager.initializeDatabase()` creates:

- `users`
- `hotels`
- `rooms`
- `bookings`
- `extras`
- `booking_extras`
- `ratings`
- `support_tickets`
- `audit_logs`

The table fields match the Java model classes and DAO queries.

Safe migrations add `hotels.hotel_user_id`, room discount fields, and booking lifecycle metadata such as cancellation reason, cancellation actor, acceptance time, and checkout validation time. Existing `GUEST` values are converted to `CLIENT`, and existing `CONFIRMED` bookings are converted to `ACCEPTED`.

The one-admin rule is also enforced during migration. The `admin` account is kept active as `ADMIN` with password `admin123`; any other `ADMIN` rows are deactivated.

## 6. Relationships Between Tables

One hotel has many rooms. One room can have many bookings over time. One user can have many bookings. A booking connects a user, a hotel, and a room. Extras are optional and can be connected to bookings through `booking_extras`.

## 7. Authentication Flow

The user enters username and password in `LoginFrame`. The frame calls `AuthController.login()`. The controller calls `AuthService.login()`. The service asks `UserDAO.findByUsername()` for the account and checks the password with `PasswordUtil.verifyPassword()`.

If the login succeeds, `MainFrame` opens the client, hotel, or admin dashboard based on the user's role.

## 8. Search Flow

The guest enters city, check-in date, check-out date, guests, and optional max price in `SearchPanel`. The panel calls `GuestController.searchHotels()`. The controller calls `HotelService.searchHotels()`, which validates dates and asks the DAO layer for active hotels with available active rooms.

The results show hotel name, city, address, rating, lowest available price, and number of available rooms.

## 9. Booking Flow

The guest selects a hotel, views available rooms, selects a room, and confirms the booking. `BookingPanel` calls `BookingController.createBooking()`. The controller calls `BookingService.createBooking()`.

The booking service checks that the user is logged in, the room exists, the guest count fits the room capacity, the dates are valid, and the room is still available.

## 10. Availability Checking Flow

Availability is checked using active bookings only. A room is blocked only by bookings with status `PENDING`, `ACCEPTED`, `CHECKED_IN`, or `CHECKOUT_PENDING_VALIDATION`. Rejected, cancelled, and completed bookings do not block future searches.

The date overlap rule is:

```java
newCheckIn.isBefore(existingCheckOut) && newCheckOut.isAfter(existingCheckIn)
```

In SQL this is implemented as:

```sql
check_in_date < ? AND check_out_date > ?
```

## 11. Double-Booking Prevention Explanation

The most important method is `BookingService.createBooking()`. It opens one JDBC connection, disables auto-commit, rechecks room availability inside the same transaction, inserts the booking if available, and commits. If the room is no longer available, the transaction rolls back and the user sees a friendly error.

This is safer than checking availability in one step and inserting later in a separate step.

## 12. Guest Booking History Explanation

`MyBookingsPanel` shows only bookings for the logged-in guest. It calls `BookingController.getMyBookings()`, which uses `BookingService.getBookingsForUser()`.

The client can cancel a booking only if it belongs to them, has status `PENDING` or `ACCEPTED`, and the current date is before check-in. The UI opens a required reason dialog; empty reasons are rejected. The reason, actor role, actor id, and cancellation time are stored on the booking and logged.

## 13. Hotel Account Dashboard Explanation

Hotel users are normal `users` rows with role `HOTEL`, linked to one hotel through `hotels.hotel_user_id`. `HotelAccountService` always resolves the logged-in hotel first, then filters room and booking operations by that hotel id. This prevents hotel users from viewing or editing other hotels.

The hotel dashboard includes overview stats, own room management, own booking management, checkout validation, support/reclamation, and personal room rankings.

Hotel room management lets the hotel create and edit only its own rooms. A hotel can change the room number, type, capacity, base price, description, active status, and promotion discount. `HotelAccountService` always overwrites the room hotel id with the logged-in hotel id before saving, so a hotel account cannot move or edit another hotel's room.

Room discounts use a simple percentage model. Rooms store `discount_percent` and `promotion_active`. When promotion is active, the final room price is:

```text
final_price = base_price * (1 - discount_percent / 100)
```

If promotion is not active, final price is the base price. Search results, available-room lists, booking totals, and room management screens use the final price.

## 14. Support / Reclamation Workflow

Clients and hotels create tickets from their dashboard. Tickets store sender user, sender role, subject, message, category, status, response, creation time, and resolution time. Admins filter tickets by role, status, and category, respond to tickets, and update ticket status. Ticket creation and updates are written to `audit_logs`.

## 15. Audit Log Workflow

Important actions write structured rows to `audit_logs`: login, registration, booking creation, cancellation, hotel acceptance, checkout validation, room changes, rating submission, support creation/update, and admin management actions. Admins can filter logs by role, action text, date range, and user id.

## 16. Rating Rule

Ratings are stored in `ratings` and linked to a completed booking. A client can rate only when the booking is `COMPLETED`, the hotel has set `checkout_validated_at`, and no rating already exists for that booking. Hotel average rating is recalculated from valid ratings.

## 17. Ranking And Report Logic

Admin reports show hotel revenue ranking, hotel rating ranking, room revenue ranking, room booking-count ranking, occupancy estimate, and ratings oversight. Hotel dashboards show personal room revenue and booking rankings. Monthly revenue is used for the highest revenue period because it is deterministic and easy to explain from `check_in_date`.

## 18. Client Loyalty Points

Clients have a points balance, total points earned, and total points used on the `users` table. Every points movement is also stored in `loyalty_points_history`.

The conversion rule is simple: `1 point = 1 MAD` discount. During booking, the client can choose how many points to use. The app prevents using more points than the client owns and prevents reducing the final payable price below zero.

After a hotel validates checkout and the booking becomes `COMPLETED`, the client earns points with:

```text
earned_points = floor(final_paid_price_after_points / 20)
```

Rewards such as `FREE_HOTEL_RIDE` are stored in `loyalty_rewards`. The current implementation models the business benefit and display state without adding transport logistics.

## 19. Hotel Commission And Performance

Hotel commission is calculated when checkout is validated. Commission uses the final payable booking value after loyalty-point discounts.

```text
platform_commission = final_paid_price * commission_rate
hotel_net_revenue = final_paid_price - platform_commission
```

The hotel tier controls the commission rate:

- `STANDARD`: 8%
- `SILVER`: 6%
- `GOLD`: 4%

Hotel performance fields are stored on `hotels`: completed booking count, generated revenue, total platform commission, commission rate, tier, and priority score.

## 20. Search Priority

Hotel search still respects city, dates, guest count, price, active hotel, active room, and availability filters. Within matching results, hotels are ordered by:

1. `priority_score` descending
2. hotel rating descending
3. lowest available price ascending

The priority score is deterministic and combines completed booking count, generated revenue, and rating. This makes high-performing hotels appear first without showing non-matching hotels.

## 21. Improved Log Filters

The admin log panel now filters by actor role, action type, date range, actor user id, target type, and text search across action/target/description. The DAO query applies all selected filters to the SQL result set, so the displayed logs genuinely change with each filter.

## 22. Mockup Seed Data

The seed logic now adds moderate demo data for presentation: 30 client accounts, 10 hotels, linked hotel users, multiple rooms per hotel, bookings across statuses, completed stays with ratings, loyalty balances/history/rewards, support tickets in several statuses, and logs for admin/client/hotel actions. Seed checks avoid duplicating the known demo data every launch.

## 23. Logic Stabilization Rules

The booking lifecycle is now enforced through `BookingTransitionPolicy` before services update the database.

Allowed normal transitions:

- `PENDING` can become `ACCEPTED`, `REJECTED`, or `CANCELLED`.
- `ACCEPTED` can become `CANCELLED` or `COMPLETED`.
- `COMPLETED`, `CANCELLED`, and `REJECTED` are terminal in normal admin, hotel, and client flows.

`CHECKED_IN` and `CHECKOUT_PENDING_VALIDATION` are still treated as active/live statuses for availability and checkout validation, but the main UI keeps the simpler pending/accepted/completed flow.

Checkout validation also checks the stay date. A booking cannot become `COMPLETED` before its checkout date. This prevents future stays from triggering ratings, commission, or loyalty earning too early.

## 24. Loyalty Refund And Transaction Rules

Clients can use loyalty points during booking. One point is one MAD discount. Booking creation and point deduction now happen in one transaction so the database cannot save only one side of the operation.

If a booking that used points is cancelled or rejected before completion, those points are refunded through a `REFUNDED_AFTER_CANCELLATION` history entry. The refund is idempotent, so repeated cancellation attempts cannot refund the same booking twice.

Completed stays do not refund used points. Instead, completed validated stays earn points once using:

```text
floor(final payable MAD / 20)
```

Rating bonuses and support compensation also write loyalty history and audit logs.

## 25. Revenue, Commission, And Reports

Actual completed revenue uses the final payable amount after loyalty discounts:

```text
COALESCE(final_price_after_points, total_price)
```

Legacy bookings that had no points and stored `0` as the final price are normalized during migration by setting `final_price_after_points = total_price`.

Reports now separate:

- `Completed Revenue`: realized revenue from completed stays only.
- `Expected Revenue`: accepted/live future booking value.

Hotel commission is calculated only when checkout is validated and uses the same final payable amount after points discount. This keeps hotel revenue, platform commission, rankings, and dashboard KPIs consistent.

## 26. Safety Guardrails

Room and hotel deactivation is blocked while active bookings exist. Active means `PENDING`, `ACCEPTED`, `CHECKED_IN`, or `CHECKOUT_PENDING_VALIDATION`.

Admin safety checks prevent:

- creating any additional admin account
- editing, disabling, deleting, or otherwise changing admin accounts from the Users panel
- silently overwriting a hotel's linked hotel user account

Creating a hotel account now checks the hotel link first and saves the new user plus hotel link in one transaction.

The admin `Users` panel is for CLIENT and HOTEL account management. It can create client accounts, create hotel accounts linked to unclaimed hotels, edit profile data, enable/disable users, safely deactivate users instead of physically deleting rows, inspect linked hotels, and give or take client loyalty points. Points adjustments are written to `loyalty_points_history` and `audit_logs`; HOTEL users do not use client loyalty balances.

The platform supports exactly one administrator account: `admin / admin123`.

## 26.1 Table Organization

Major list screens use column-header organization through `TableRowSorter`. Tables with an ID column open sorted by ID ascending. Clicking a column header sorts ascending, clicking again sorts descending, and numeric/date/money columns are compared as values instead of text.

This applies to the Users, Hotels, Rooms, Bookings, Support, Logs, Reports, Loyalty, client booking history, hotel room lists, hotel booking lists, hotel reports, search results, and hotel-details room tables.

## 27. Support Compensation Atomicity

Support ticket updates with compensation are atomic. If point compensation is invalid or fails, the ticket response/status update is rolled back too.

Duplicate support compensation is prevented with the structured `support_ticket_id` column in `loyalty_points_history`, not by searching description text.

## 28. Seed Data Integrity

Fresh demo bookings now use overlap checks before inserting active `PENDING` or `ACCEPTED` bookings. Existing small demo databases are repaired by cancelling later duplicate active overlaps with a clear demo cleanup reason. Completed historical stays are not changed by this repair.

## 13. Admin Dashboard Explanation

The admin dashboard uses a left sidebar and a content area. It includes:

- Overview statistics
- Hotel management
- Room management
- Booking management
- User management
- Reports

All admin actions go through `AdminController`, then `AdminService`, then DAO classes.

## 14. Controller To Service To DAO To DB Flow

The project follows this request flow:

Swing button -> Controller -> Service -> DAO -> SQLite database -> DAO -> Service -> Controller -> Swing update

This keeps SQL out of views and keeps business rules out of Swing panels.

## 15. Button-By-Button Explanation Of Major Screens

Login:
- Login button checks credentials.
- Register button opens the registration screen.

Guest search:
- Search button validates criteria and loads hotel results.
- View Details button opens the selected hotel and available rooms.
- Book Now button opens the booking confirmation panel.

Guest bookings:
- Refresh reloads booking history.
- Cancel Selected applies the cancellation rules.

Admin:
- Sidebar buttons switch between sections.
- Add/Edit buttons open forms and save through services.
- Deactivate buttons keep records but mark hotels or rooms inactive.
- Confirm/Cancel buttons update booking status.
- Enable/Disable buttons update user status.

## 16. Important Methods For Oral Defense

- `DatabaseManager.initializeDatabase()`
- `AuthService.login()`
- `HotelService.searchHotels()`
- `BookingService.createBooking()`
- `BookingDAO.hasOverlappingBooking()`
- `PricingService.calculateTotalPrice()`
- `BookingService.cancelGuestBooking()`
- `ReportService.getDashboardStats()`

## 17. How To Run The Project

Use Maven:

```bash
mvn clean compile
mvn exec:java
```

Or use the included wrapper on Windows:

```powershell
.\mvnw.cmd clean compile
.\mvnw.cmd exec:java
```

## 18. How To Inspect The SQLite Database

Open `hotel_booking.db` with any SQLite browser, such as DB Browser for SQLite. You can inspect the `users`, `hotels`, `rooms`, and `bookings` tables to verify how the UI changes the database.

Useful checks:

- `users` contains the seeded admin and guest accounts.
- `hotels` contains the seeded hotels.
- `rooms` contains rooms linked to hotels by `hotel_id`.
- `bookings` shows the effect of guest booking, admin booking changes, and cancellation.
- A cancelled booking remains in the table, but its status becomes `CANCELLED`.

## 19. Future Possible Improvements

Future versions could add email confirmations, room images, printed invoices, localization, seasonal pricing, and a hotel staff role. These were intentionally excluded from this version to keep the project stable and easy to explain.

## Final Verification Notes

The project was compiled with the Maven wrapper using:

```powershell
.\mvnw.cmd clean compile
```

A smoke test also checked seeded login, guest registration, disabled-user login rejection, hotel search, available rooms, booking creation, overlapping booking rejection, cancellation, admin data loading, and reports.
