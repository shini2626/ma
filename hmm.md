# Build Log

## USER / HOTEL MANAGEMENT IMPROVEMENT AUDIT

What already exists:
- The app is an existing Java Swing/Maven/SQLite project with MVC-style controllers, services, DAOs, and models.
- Roles already use `CLIENT`, `HOTEL`, and `ADMIN`; legacy `GUEST` values are migrated to `CLIENT`.
- The admin dashboard already has panels for hotels, rooms, bookings, clients/users, support, reports, and logs.
- `UserManagementPanel` lists all users and currently supports adding admin users, adding hotel accounts, enabling users, disabling users, and refreshing.
- Hotel accounts are normal `users` rows with role `HOTEL`, linked to exactly one hotel through `hotels.hotel_user_id`.
- `HotelAccountService` already limits hotel users to their own hotel, rooms, bookings, stats, and room rankings.
- Admin hotel, room, booking, support, audit-log, report, and loyalty support flows already exist.
- Loyalty points already live on `users` and `loyalty_points_history`; support compensation, rating bonuses, booking point usage, and cancellation refunds are logged.
- Room management exists for admin and hotel users, with create/edit/status operations and active-booking guards.
- Search, booking, room, hotel, user, support, logs, reports, loyalty, and booking-list screens use Swing `JTable`, but most do not install consistent `TableRowSorter` behavior.

What is currently missing:
- The admin navigation calls the users panel `Clients`, while the panel title says `User Management`; this should become a clear `Users` section.
- The users panel still exposes `Add Admin`, which conflicts with the new one-admin-only rule.
- Admin user management does not yet support create-client, edit-user, delete/deactivate from the main users screen, direct give/take points, or linked-hotel inspection.
- Backend service rules currently allow creating additional admin users and only protect the last active admin, instead of protecting the single default `admin` account.
- Seed/migration logic does not yet deactivate legacy extra admin accounts automatically.
- Rooms do not yet have discount fields or a computed final/display price.
- Hotel and admin room dialogs do not yet expose discount controls.
- Table sorting is inconsistent and selected-row handling must convert view rows to model rows after sorting is enabled.

Files planned for modification:
- `src/main/java/com/hotelbooking/dao/DatabaseManager.java`
- `src/main/java/com/hotelbooking/dao/UserDAO.java`
- `src/main/java/com/hotelbooking/dao/RoomDAO.java`
- `src/main/java/com/hotelbooking/dao/LoyaltyDAO.java`
- `src/main/java/com/hotelbooking/model/Room.java`
- `src/main/java/com/hotelbooking/service/AdminService.java`
- `src/main/java/com/hotelbooking/service/HotelAccountService.java`
- `src/main/java/com/hotelbooking/controller/AdminController.java`
- `src/main/java/com/hotelbooking/view/AdminDashboardPanel.java`
- `src/main/java/com/hotelbooking/view/UserManagementPanel.java`
- `src/main/java/com/hotelbooking/view/RoomManagementPanel.java`
- `src/main/java/com/hotelbooking/view/HotelDashboardPanel.java`
- Major table panels such as hotel, booking, support, logs, reports, search, loyalty, and booking-list views for sorter installation.
- A small reusable table sorting helper under `src/main/java/com/hotelbooking/util/`.
- Documentation: `BUILD_LOG.md`, `PROJECT_EXPLANATION.md`, and `README.md`.

Admin-account restriction strategy:
- The platform supports exactly one administrator account: `admin / admin123`.
- Database migration/seed cleanup will keep the `admin` user active as `ADMIN`, reset its password to `admin123`, and deactivate any other `ADMIN` rows.
- `AdminService` will block creation of new admin users through normal admin tools.
- User edit/status/delete logic will reject changes to any `ADMIN` row, including the protected default admin.
- The users UI will remove the `Add Admin` action and only allow creation of `CLIENT` and `HOTEL` users.

Hotel self-management improvement strategy:
- Keep `HotelAccountService.getHotelForUser()` as the ownership gate.
- Continue forcing `saveOwnRoom()` to use the logged-in hotel id before validation.
- Add simple room discount fields to rooms and expose them in admin/hotel room dialogs.
- Use a simple percentage discount model. Final price = `base_price * (1 - discount_percent / 100)` when promotion is active; otherwise final price = base price.
- Validate discounts between 0 and 100 and keep active-booking protection for room deactivation.

Sorting implementation strategy:
- Add a reusable `TableSortUtil` based on `TableRowSorter`.
- Apply it to all major `JTable` list screens and default to ID ascending when an ID column exists.
- Use comparators that sort numbers, money strings, dates, date-times, enums, booleans, and text correctly.
- Update row-selection code to convert selected view rows to model rows before indexing backing lists.

Compatibility preservation:
- Existing DAOs/services/controllers remain in place; changes are additive or narrowly restrictive.
- Existing booking, support, loyalty, hotel-link, and active-booking guard logic remains intact.
- Schema changes are safe migrations using `ALTER TABLE ... ADD COLUMN` only when missing.
- Delete in user management will be implemented as safe deactivation to avoid breaking historical bookings, loyalty history, support tickets, and audit logs.

## Loyalty / Commission Upgrade Initial Audit

Current app state before this upgrade:
- Roles are now `CLIENT`, `HOTEL`, and `ADMIN`, with legacy `GUEST` values read as `CLIENT`.
- Client booking flow searches hotels, opens available rooms, creates a `PENDING` booking request, supports cancellation reasons, and allows rating after validated checkout.
- Hotel account logic exists through `hotels.hotel_user_id`; hotel users can see and manage only their own hotel rooms and bookings.
- Rating logic exists through `ratings`; one completed validated booking can receive one rating.
- Support/reclamation exists through `support_tickets`, with client/hotel creation and admin filtering by sender role/status/category.
- Audit logs exist through `audit_logs`, but the admin filter is limited to actor role, action text, date range, and actor id; it lacks target type and general text search.
- Reports include hotel revenue, hotel rating, room revenue, room booking count, occupancy estimate, and ratings oversight.
- Search sorting still orders primarily by hotel rating and price; it does not use hotel performance priority.
- Seed data is useful but still small: a few clients, five hotels, hotel accounts, bookings, ratings, tickets, and logs.

Weak points to fix:
- No client loyalty points balance, points history, rewards, or points usage during booking.
- No booking fields for points discount or final payable amount.
- No hotel commission/performance tier fields or completion-time commission calculation.
- No priority score affecting search result order.
- Log filtering needs target type and text search so admin filtering is genuinely practical.
- Seed data must expand to 30 client accounts, 10 hotels, linked hotel accounts, richer bookings, support tickets, logs, ratings, loyalty data, and commission/performance data.

Likely files/classes to modify:
- `DatabaseManager`, `UserDAO`, `HotelDAO`, `BookingDAO`, `AuditLogDAO`
- `User`, `Hotel`, `Booking`, `BookingView`, `HotelSearchResult`
- `BookingService`, `HotelAccountService`, `ReportService`, new loyalty/performance services
- `BookingController`, `AdminController`, new/extended loyalty controller methods
- `BookingPanel`, `GuestDashboardPanel`, `HotelDashboardPanel`, `ReportPanel`, `AuditLogPanel`
- Documentation files: `README.md`, `PROJECT_EXPLANATION.md`, `BUILD_LOG.md`

Planned schema additions:
- User loyalty fields: `loyalty_points_balance`, `total_points_earned`, `total_points_used`
- `loyalty_points_history`
- `loyalty_rewards`
- Booking points and commission fields: `points_used`, `points_discount_amount`, `final_price_after_points`, `commission_rate_applied`, `platform_commission_amount`, `hotel_net_revenue`
- Hotel performance fields: `commission_rate`, `performance_tier`, `priority_score`, `total_generated_revenue`, `total_completed_bookings`, `total_platform_commission_paid`

Implementation plan:
- Add safe migrations and seed expansion first.
- Add loyalty/reward DAO and service logic, then connect booking summary to optional point usage.
- Add performance/commission logic on checkout completion and expose it in hotel/admin reports.
- Update search SQL ordering to use `priority_score`, then rating and price as tie-breaks.
- Extend audit log filtering with target type and text search.
- Update docs and run compile/launch smoke tests.

## Loyalty / Commission Upgrade Implementation

Schema changes added safely:
- `users`: `loyalty_points_balance`, `total_points_earned`, `total_points_used`
- `hotels`: `commission_rate`, `performance_tier`, `priority_score`, `total_generated_revenue`, `total_completed_bookings`, `total_platform_commission_paid`
- `bookings`: `points_used`, `points_discount_amount`, `final_price_after_points`, `commission_rate_applied`, `platform_commission_amount`, `hotel_net_revenue`
- New tables: `loyalty_points_history`, `loyalty_rewards`

New DAO/service/controller additions:
- `LoyaltyDAO`, `LoyaltyService`, `LoyaltyController`
- Extended `BookingDAO`, `HotelDAO`, `UserDAO`, `AuditLogDAO`
- Extended `BookingService` and `HotelAccountService` so booking uses points and checkout awards points/calculates commission.

Business rules implemented:
- 1 loyalty point equals 1 MAD discount.
- Clients earn `floor(final paid price / 20)` points after a completed validated stay.
- Points usage is optional during booking and cannot exceed the client balance or booking total.
- Commission is based on final payable value after points discount.
- Hotel tiers are `STANDARD` at 8%, `SILVER` at 6%, and `GOLD` at 4%.
- Hotel priority score uses completed booking count, generated revenue, and rating, then search results order by priority score, rating, and price.

UI changes:
- Client dashboard now includes `Loyalty / Rewards`.
- Booking summary now shows available points and allows optional point usage.
- Hotel dashboard stats now include performance tier, commission rate, platform commission paid, and priority score through existing overview cards.
- Admin reports now include commission tier overview and loyalty overview.
- Admin log filter now supports actor role, action type, date range, user id, target type, and text search.

Seed/mockup data changes:
- Demo clients are expanded to `client01` through `client30`.
- Demo hotels are expanded to 10 total with hotel accounts linked to each.
- Rooms, bookings, loyalty history, rewards, support tickets, ratings, and audit logs are populated with moderate demo data.
- Seed logic is guarded so known demo rows are not duplicated every launch.

Verification:
- `.\mvnw.cmd clean compile` passed after the implementation.
- `.\mvnw.cmd exec:java` stayed running during launch smoke testing, confirming startup migrations and seed logic did not crash.

## Log Filter Fix

Problem found:
- The log panel used a blank `null` combo-box value for role and raw text fields for every other filter. This made the UI unclear and made it look like filters were not being applied.
- The DAO text search did not use `COALESCE`, so nullable fields such as `target_type` could behave inconsistently in combined text searches.

Fix applied:
- Replaced the blank role filter with explicit `All`, `CLIENT`, `HOTEL`, and `ADMIN` choices.
- Added an explicit target-type filter with `All`, `USER`, `BOOKING`, `HOTEL`, `ROOM`, `SUPPORT_TICKET`, `RATING`, `LOYALTY_REWARD`, and `AUDIT_LOG`.
- Added a `Clear` button and result count label.
- Kept every filter optional, so admin can apply one filter, multiple filters, all filters, or none.
- Updated `AuditLogDAO` to normalize blank/`All` values to no filter and use `COALESCE` for action, target, and text search filters.

Verification:
- `.\mvnw.cmd clean compile` passed.
- DAO smoke checks confirmed separate and combined filters return different result counts: all logs, admin-only, hotel-only, target booking, commission text search, and hotel+booking+commission combination.

## Log Filter UI Fix

Problem found during visual testing:
- Selecting a role such as `HOTEL` did not immediately filter the table because filtering still depended on an apply action that could be pushed off-screen by the wide toolbar.
- Several filters were typed fields, which made demo use error-prone.

Fix applied:
- Rebuilt `AuditLogPanel` so all filter inputs are dropdowns populated from existing database values.
- Role, action, actor, from date, to date, target type, and description are now selectable choices.
- Every dropdown has an explicit `All` option.
- Changing any dropdown applies the filters immediately.
- `Clear` resets all filters to `All`.
- `Refresh Choices` reloads dropdown values from the current database.

Verification:
- `.\mvnw.cmd clean compile` passed.
- DAO check confirmed selecting `HOTEL` returns only hotel-role rows: `hotelOnly=true`.
- Combined filter check for `HOTEL + COMMISSION_CALCULATED + BOOKING + commission` returned matching rows.

## Log Filter Label, Date, And Access Fix

Problem found:
- The labels `Role` and `Actor` were unclear. `Role` meant actor role (`CLIENT`, `HOTEL`, `ADMIN`), while `Actor` meant the exact user id that performed the action.
- Date filters as dropdowns were less useful than manual date input.
- The admin log access rule was enforced by dashboard routing, but not explicitly checked at the controller method that reads logs.

Fix applied:
- Renamed `Role` to `Actor Role`.
- Renamed `Actor` to `Actor User`.
- Restored manual `From` and `To` date fields with visible `yyyy-MM-dd` labels and tooltips.
- Open-ended dates are supported by the DAO:
  - `From` only means from that date at `00:00:00` until now/latest records.
  - `To` only means everything before that date at `23:59:59`.
  - Both dates means the inclusive range.
- Added `AdminController.requireAdmin()` and routed log reads through `getAuditLogs(User admin, ...)`.

Verification:
- `.\mvnw.cmd clean compile` passed.
- DAO date checks passed for from-only, to-only, and range queries on current seed data.

## Log Date Field Visibility Fix

Problem found:
- The log filter toolbar was too wide, so the `To` date field could be pushed off-screen and look missing.
- Date fields were hard to distinguish from dropdown-based filters.

Fix applied:
- Rebuilt the log filter toolbar with a compact `GridBagLayout` split across three rows.
- `From (yyyy-MM-dd)` and `To (yyyy-MM-dd)` are now visible editable text fields.
- Added a visible `Apply` button for date filtering while dropdown filters still auto-apply.

Verification:
- `.\mvnw.cmd clean compile` passed.

## Log Date Mask And Filter Simplification

Problem found:
- Date typing was still awkward and users could type malformed dates.
- The Description filter made the toolbar too crowded and was not needed for the requested filtering workflow.

Fix applied:
- Replaced plain date text fields with masked `yyyy-MM-dd` fields. Dashes are inserted automatically after the year and month.
- Partial dates are rejected before filtering.
- Impossible dates are rejected before filtering.
- Removed the Description filter from the log UI.
- Kept all remaining filters optional: actor role, action, actor user, from date, to date, and target can be used alone or in any combination.

Verification:
- `.\mvnw.cmd clean compile` passed.

## Log Date Range Validation Fix

Problem found:
- The log filter allowed a `To` date earlier than the `From` date.

Fix applied:
- Added validation in `AuditLogPanel` before querying logs.
- If both dates are entered and `To` is earlier than `From`, the app shows: `To date cannot be earlier than From date.`

Verification:
- `.\mvnw.cmd clean compile` passed.

## Log Filter Layout Organization

Problem found:
- The log filter controls were visually stretched across the whole panel and hard to scan.

Fix applied:
- Reorganized the log filter area into a title plus compact filter grid.
- Standardized control widths for role, action, actor user, dates, and target.
- Kept action buttons and result count grouped on a separate row.

Verification:
- `.\mvnw.cmd clean compile` passed.

## Upgrade Initial Audit: Three-Role Platform Extension

Existing project state before the upgrade:
- Packages already followed a clear MVC-style structure: `model`, `model.enums`, `dao`, `service`, `controller`, `view`, and `util`.
- Existing roles were `GUEST` and `ADMIN`; the guest role represented normal booking clients.
- Existing booking statuses were `PENDING`, `CONFIRMED`, `CANCELLED`, and `COMPLETED`.
- Existing schema had `users`, `hotels`, `rooms`, `bookings`, `extras`, and `booking_extras`.
- Existing booking flow created confirmed bookings directly, allowed guest cancellation before check-in, and let admin confirm/cancel bookings.
- Existing admin dashboard had overview, hotel management, room management, booking management, user management, and basic reports.

Upgrade work planned from the audit:
- Replace the role model with `CLIENT`, `HOTEL`, and `ADMIN`, while reading legacy `GUEST` as `CLIENT`.
- Link hotel accounts to real hotel rows through `hotels.hotel_user_id`.
- Extend bookings with acceptance, cancellation reason, cancellation actor, and checkout validation fields.
- Add `ratings`, `support_tickets`, and `audit_logs` tables.
- Add hotel-owned dashboard logic, support/reclamation workflow, audit log filtering, rating-after-checkout flow, and expanded ranking reports.
- Add new model, DAO, service, controller, and Swing view classes without bypassing the existing layered architecture.

## Upgrade Step 1: Schema, Role, And Model Expansion

Files changed or added:
- Extended `UserRole` to `CLIENT`, `HOTEL`, `ADMIN`.
- Extended `BookingStatus` to the operational lifecycle: `PENDING`, `ACCEPTED`, `REJECTED`, `CANCELLED`, `CHECKED_IN`, `CHECKOUT_PENDING_VALIDATION`, `COMPLETED`.
- Added models for `Rating`, `SupportTicket`, and `AuditLog`.
- Added support enums `SupportCategory` and `SupportStatus`.
- Extended `Hotel`, `Booking`, and `BookingView` with hotel account, cancellation, checkout, and rating-view fields.
- Updated `DatabaseManager` with additive migrations and new tables.

Safe migration decisions:
- Existing `GUEST` values are migrated to `CLIENT`.
- Existing `CONFIRMED` bookings are migrated to `ACCEPTED`.
- New columns are added with guarded `ALTER TABLE` checks.
- Existing data is preserved; no destructive reset is used.

## Upgrade Step 2: DAO, Service, And Controller Additions

Files added:
- `AuditLogDAO`, `SupportTicketDAO`, `RatingDAO`
- `AuditService`, `SupportService`, `RatingService`, `HotelAccountService`
- `HotelController`, `SupportController`

Files extended:
- `BookingDAO` now supports hotel-specific booking queries, cancellation metadata, hotel acceptance, checkout validation, rankings, hotel stats, and rating-aware booking views.
- `HotelDAO` now supports `hotel_user_id` linking.
- `RoomDAO` and `HotelDAO` now block availability with the new active booking statuses.
- `BookingService` now creates client bookings as `PENDING`, requires cancellation reasons, and logs booking actions.
- `AdminService` now supports hotel account creation, reason-based cancellation, and logging.
- `ReportService` now exposes hotel rating and room revenue rankings.

## Upgrade Step 3: Swing UI Integration

Files added:
- `HotelDashboardPanel`
- `SupportPanel`
- `SupportManagementPanel`
- `AuditLogPanel`

Files extended:
- `MainFrame` routes `CLIENT`, `HOTEL`, and `ADMIN` to separate dashboards.
- `GuestDashboardPanel` includes Support / Reclamation.
- `MyBookingsPanel` requires cancellation reasons and exposes hotel rating only after completed validated checkout.
- `BookingManagementPanel` requires admin cancellation reasons.
- `AdminDashboardPanel` includes Support and Logs sections.
- `ReportPanel` includes hotel rating ranking, room revenue ranking, and ratings oversight.
- `UserManagementPanel` can create hotel accounts linked to hotels.

## Upgrade Step 4: Verification

Commands run:
- `.\mvnw.cmd clean compile`
- Short launch smoke test with `.\mvnw.cmd exec:java`

Results:
- Compile succeeded with 65 Java source files.
- Swing launch stayed running, confirming startup and database migrations did not crash.
- The launch process was stopped after the smoke test.

Remaining manual demo checks:
- Click through each role in the visible Swing UI.
- Test hotel account isolation with seeded accounts such as `hotel_blue / hotel123`.
- Test cancellation reason dialogs from client, hotel, and admin flows.
- Test support ticket creation and admin response.
- Test rating after hotel checkout validation.

## Step 1: Created Maven Project Structure

Files created:
- `pom.xml`
- `mvnw`
- `mvnw.cmd`
- `.mvn/wrapper/maven-wrapper.properties`
- `src/main/java/com/hotelbooking/...`

Logic added:
- Java 17 Maven compilation.
- SQLite JDBC dependency.
- Exec Maven plugin configured to run `com.hotelbooking.Main`.
- Maven wrapper fallback because Maven is not installed on this machine.

Errors encountered:
- `mvn` was not available on PATH.

Fix applied:
- Added wrapper scripts that download Apache Maven into `.mvn/wrapper/`.

What remains to test:
- Run `.\mvnw.cmd clean compile` after source code is complete.

## Step 2: Added Models, Enums, And Utilities

Files created:
- `User`, `Hotel`, `Room`, `Booking`, `Extra`
- `HotelSearchResult`, `BookingView`
- `UserRole`, `BookingStatus`, `RoomType`
- `PasswordUtil`, `DateUtil`, `ValidationUtil`, `DialogUtil`, `UITheme`

Logic added:
- Domain classes mirror the database tables.
- Enums protect role, booking status, and room type values.
- `PasswordUtil.hashPassword()` adds SHA-256 hashing.
- `DateUtil.nightsBetween()` implements the number-of-nights calculation.
- `UITheme` centralizes colors, button styling, cards, titles, and table styling.

How UI buttons will use this:
- Swing views will call controllers and use `DialogUtil` for friendly messages.
- Views will use `UITheme` so styling is consistent.

Errors encountered:
- None in this step.

What remains to test:
- Compile after DAO, service, controller, and view layers are added.

## Step 3: Added Database Manager And DAO Layer

Files created:
- `DatabaseManager`
- `UserDAO`
- `HotelDAO`
- `RoomDAO`
- `BookingDAO`
- `ExtraDAO`

Logic added:
- `DatabaseManager.initializeDatabase()` creates all required tables and indexes.
- Seed data now includes admin, guest, five hotels, at least three rooms per hotel, extras, and a small safe sample booking.
- `HotelDAO.searchAvailableHotels()` returns only active hotels with active rooms available for selected dates.
- `RoomDAO.findAvailableRooms()` returns available rooms for a hotel/date/guest query.
- `BookingDAO.hasOverlappingBooking()` implements the required overlap rule.
- `BookingDAO.findViewsByUserId()` and `BookingDAO.findAllViews()` join bookings with users, hotels, and rooms for UI tables.

How requests travel:
- Services will call DAO methods.
- DAO methods are the only classes that contain SQL.
- Swing views will not call SQL directly.

Special note:
- The required SQL overlap condition was added with `check_in_date < ? AND check_out_date > ?` and status filter `PENDING`, `CONFIRMED`.

Errors encountered:
- None at creation time; compile will be checked after all layers exist.

What remains to test:
- Database creation on app startup.
- Seed insertion.
- Search and booking queries through services and UI.

## Step 4: Added Service Layer

Files created:
- `AuthService`
- `HotelService`
- `RoomService`
- `BookingService`
- `AdminService`
- `PricingService`
- `ReportService`

Logic added:
- `AuthService.login()` checks username, password, hashed password match, and disabled accounts.
- `AuthService.registerGuest()` validates registration and unique usernames.
- `PricingService.calculateTotalPrice()` calculates `number_of_nights * room_base_price`.
- `HotelService.searchHotels()` validates search criteria before DAO access.
- `BookingService.createBooking()` validates user, room, hotel, dates, capacity, and price.
- `BookingService.createBooking()` also starts a transaction, rechecks overlap, inserts the booking, and commits.
- `BookingService.cancelGuestBooking()` enforces ownership, status, and check-in date rules.
- `AdminService` handles hotel, room, booking, and user management operations.
- `ReportService` calculates overview and report values.

How buttons will be connected:
- Login and register buttons will call `AuthController`, which will call `AuthService`.
- Search buttons will call `GuestController`, which will call `HotelService` and `RoomService`.
- Booking buttons will call `BookingController`, which will call `BookingService`.
- Admin action buttons will call `AdminController`, which will call `AdminService` or `ReportService`.

Special note:
- The transaction-based double-booking prevention is implemented in `BookingService.createBooking()`.
- The service uses one JDBC connection, disables auto-commit, calls `BookingDAO.hasOverlappingBooking()`, inserts through `BookingDAO.insert()`, then commits or rolls back.

Errors encountered:
- None at creation time.

What remains to test:
- Compile services with controllers and views.
- Manually test overlapping booking rejection from UI.

## Step 5: Added Controllers And Startup Logic

Files created:
- `Main`
- `AuthController`
- `GuestController`
- `BookingController`
- `AdminController`

Logic added:
- `Main` initializes the database, applies the Swing look and feel, and opens `LoginFrame`.
- Controllers expose simple methods for Swing views.
- Controllers construct the service and DAO dependencies needed by each feature area.

How requests travel:
- A login button will call `AuthController.login()`.
- A guest search button will call `GuestController.searchHotels()`.
- A book button will call `BookingController.createBooking()`.
- Admin CRUD buttons will call `AdminController` methods.

Errors encountered:
- None at creation time.

What remains to test:
- Add Swing views and then compile.

## Step 6: Added Login, Registration, Guest Search, Details, Booking, And History UI

Files created:
- `LoginFrame`
- `RegisterFrame`
- `MainFrame`
- `GuestDashboardPanel`
- `SearchPanel`
- `HotelDetailsPanel`
- `BookingPanel`
- `MyBookingsPanel`

Logic added:
- Login button calls `AuthController.login()` and opens `MainFrame`.
- Register button opens `RegisterFrame`; registration calls `AuthController.registerGuest()`.
- Guest dashboard uses tabs for hotel search and booking history.
- Search button validates date/number input and calls `GuestController.searchHotels()`.
- View Details button loads available rooms through `GuestController.getAvailableRooms()`.
- Book Now opens `BookingPanel`.
- Confirm Booking calls `BookingController.createBooking()`.
- My Bookings refreshes through `BookingController.getMyBookings()`.
- Cancel Selected calls `BookingController.cancelMyBooking()`.

How requests travel:
- Guest action buttons now follow Swing panel -> controller -> service -> DAO -> SQLite.

Errors encountered:
- None at creation time.

What remains to test:
- Add admin panels.
- Compile and fix any integration errors.

## Step 7: Added Admin Dashboard And Management Panels

Files created:
- `AdminDashboardPanel`
- `HotelManagementPanel`
- `RoomManagementPanel`
- `BookingManagementPanel`
- `UserManagementPanel`
- `ReportPanel`

Logic added:
- Admin dashboard uses a left sidebar and card-based content area.
- Overview loads dashboard statistics through `AdminController.getDashboardStats()`.
- Hotel management supports add, edit, activate, deactivate, filter, and refresh.
- Room management supports add, edit, activate, deactivate, hotel filtering, and refresh.
- Booking management supports all-booking view, status/date filters, pending confirmation, cancellation, and guest details.
- User management supports viewing users, adding admin users, enabling users, and disabling users.
- Reports show bookings per hotel, revenue per hotel, most booked rooms, and occupancy estimate.

How buttons were wired:
- Every admin button calls `AdminController`, which calls `AdminService` or `ReportService`.
- Forms validate through service methods before DAO writes.

Errors encountered:
- None at creation time.

What remains to test:
- Compile.
- Launch app.
- Test guest and admin flows.

## Step 8: First Compile And Runtime Dependency Fix

Command run:
- `.\mvnw.cmd clean compile`

Result:
- Maven wrapper downloaded Apache Maven successfully.
- SQLite JDBC dependency downloaded successfully.
- Java compilation succeeded for all source files.

Error found during smoke testing:
- JShell could not load `org.sqlite.JDBC` because `org.slf4j.LoggerFactory` was missing.

Fix applied:
- Added `org.slf4j:slf4j-simple` to `pom.xml` so the SQLite JDBC driver has a runtime logging binding.

What remains to test:
- Re-run compile after dependency change.
- Re-run smoke checks.

## Step 9: Recompiled And Ran Core Smoke Tests

Commands run:
- `.\mvnw.cmd clean compile`
- JShell service smoke test using compiled classes and Maven dependencies.
- Short UI launch check using `.\mvnw.cmd exec:java`.

Compile result:
- Build succeeded.
- 47 Java source files compiled.

Smoke test logic checked:
- `DatabaseManager.initializeDatabase()` created `hotel_booking.db`.
- Admin login works with `admin/admin123`.
- Guest login works with `guest/guest123`.
- Guest registration works with a unique username.
- Disabled user login is blocked.
- Guest hotel search returns available hotels.
- Hotel details flow returns available rooms.
- Booking creation inserts a confirmed booking.
- Attempting the same room and overlapping dates is rejected.
- Guest cancellation changes booking status to `CANCELLED`.
- Cancelled booking frees room availability, verified by booking the same room/date range again.
- Guest booking history loads.
- Admin dashboard statistics load.
- Admin hotel, room, booking, user, and report data loads.
- Admin add/edit hotel actions work through `AdminController.saveHotel()`.
- Admin activate/deactivate hotel works through `AdminController.setHotelActive()`.
- Admin add/edit room actions work through `AdminController.saveRoom()`.
- Admin activate/deactivate room works through `AdminController.setRoomActive()`.
- Admin booking cancellation works through `AdminController.cancelBooking()`.

UI launch check:
- `.\mvnw.cmd exec:java` stayed running for 6 seconds, which confirms the Swing app starts.
- The process was stopped afterward so no background Java process remained.

Errors found:
- SQLite JDBC needed SLF4J at runtime.

Fix applied:
- Added `slf4j-simple` dependency to `pom.xml`.

Current remaining test notes:
- Manual professor/demo testing should still click through the visible Swing UI, but automated service smoke checks passed.

Cleanup:
- Removed the smoke-test database file and reinitialized `hotel_booking.db`.
- Clean seed check after reset reported 5 hotels, 15 rooms, 1 sample booking, and 2 users.

## Final Checklist

- Project compiles: passed.
- App launches: passed with short UI smoke test.
- SQLite database auto-creates: passed.
- Seed data auto-inserts: passed.
- Login works: passed for admin and guest.
- Register works: passed.
- Disabled user cannot log in: passed.
- Guest search works: passed.
- Hotel details and available rooms work: passed.
- Booking creation works: passed.
- Double-booking protection works: passed.
- Guest booking history works: passed.
- Guest cancellation works: passed.
- Cancelled booking frees availability: passed.
- Admin dashboard and reports load: passed.
- Admin hotel add/edit/deactivate: passed.
- Admin room add/edit/deactivate: passed.
- Admin booking cancellation: passed.
- User enable/disable: passed.
- Required documentation exists: passed.
- No unfinished implementation markers were found in the application source.

## Search, Date, Availability, And Points Fix

User request:
- Make guest search inputs harder to misuse.
- Limit city input to known Moroccan city choices.
- Make check-in/check-out dates use the same guided `yyyy-MM-dd` style as the logs filter.
- Prevent invalid date ranges and past check-ins.
- Allow searching without a max price.
- Prevent room double-booking and prevent one client from holding overlapping active stays.
- Clarify loyalty point payment behavior where 1 point = 1 MAD.

Files changed:
- `src/main/java/com/hotelbooking/view/SearchPanel.java`
- `src/main/java/com/hotelbooking/view/BookingPanel.java`
- `src/main/java/com/hotelbooking/service/BookingService.java`
- `src/main/java/com/hotelbooking/dao/BookingDAO.java`

Implementation notes:
- Replaced free-text city search with a scrollable combo box of Moroccan cities plus an all-cities option.
- Replaced date text boxes with masked `yyyy-MM-dd` inputs.
- Added UI validation for real dates, check-in not before today, and check-out after check-in.
- Replaced guests free text with a numeric spinner.
- Kept max price optional; blank max price now searches without a price limit.
- Added service-level protection so clients cannot create overlapping active bookings across any hotel/room.
- Room availability remains blocked only by `PENDING`, `ACCEPTED`, `CHECKED_IN`, and `CHECKOUT_PENDING_VALIDATION`; after checkout validation the `COMPLETED` booking no longer blocks availability.
- Expanded the booking summary and confirmation dialog with points discount, cash due at hotel, and a points receipt line.

Testing:
- `.\mvnw.cmd clean compile` passed after the change.
- Short `.\mvnw.cmd exec:java` launch smoke test stayed running for 8 seconds, confirming the Swing app starts.
- The smoke-test Java process was stopped afterward.

## Booking Summary, Points Compensation, And Points Log Filter Fix

User request:
- Fix the cramped booking summary appearance.
- Keep loyalty points usable before confirmation as a partial cash discount.
- Make point-backed bookings protected from normal hotel cancellation.
- Award points for rating hotels.
- Let admins grant points compensation while replying to support tickets.
- Add a logs filter focused on points balance actions.

Files changed:
- `src/main/java/com/hotelbooking/view/BookingPanel.java`
- `src/main/java/com/hotelbooking/service/HotelAccountService.java`
- `src/main/java/com/hotelbooking/service/LoyaltyService.java`
- `src/main/java/com/hotelbooking/dao/LoyaltyDAO.java`
- `src/main/java/com/hotelbooking/service/RatingService.java`
- `src/main/java/com/hotelbooking/controller/BookingController.java`
- `src/main/java/com/hotelbooking/service/SupportService.java`
- `src/main/java/com/hotelbooking/dao/SupportTicketDAO.java`
- `src/main/java/com/hotelbooking/controller/AdminController.java`
- `src/main/java/com/hotelbooking/view/SupportManagementPanel.java`
- `src/main/java/com/hotelbooking/view/AuditLogPanel.java`

Implementation notes:
- Replaced the booking summary grid with a `GridBagLayout` so labels, values, and the points spinner stay readable.
- The client can still choose any allowed number of points before confirmation; 1 point equals 1 MAD and the live cash-due amount updates immediately.
- Used points are deducted from the client wallet when the booking is created.
- Completed stays continue to earn points after hotel checkout validation.
- Submitting a valid hotel rating now grants a small rating bonus and logs it as a points balance action.
- Hotel accounts cannot cancel a booking that used points unless the cancellation reason clearly indicates no-show.
- Admin support replies now include an optional points compensation checkbox and amount spinner for client tickets.
- Support compensation updates the user balance, writes points history, and writes an audit log entry.
- Action Logs now have a `Log Type` filter with `Points Balance Logs`.

Testing:
- `.\mvnw.cmd clean compile` passed after these changes.
- Short `.\mvnw.cmd exec:java` launch smoke test stayed running for 8 seconds, confirming the app starts.
- The smoke-test Java process was stopped afterward.

## UI REDESIGN AUDIT

Current UI structure:
- `MainFrame` routes users by role to `AdminDashboardPanel`, `HotelDashboardPanel`, or `GuestDashboardPanel`.
- Admin currently has a simple left sidebar and `CardLayout` pages for overview, hotels, rooms, bookings, users, reports, support, and logs.
- Hotel currently has a blue top header and `JTabbedPane` pages for overview, rooms, bookings, support, and reports.
- Guest/client currently has a blue top header and `JTabbedPane` pages for hotel search, my bookings, loyalty/rewards, and support.
- Shared styling is mostly in `UITheme`, with basic colors, buttons, cards, titles, and table styling.
- Existing business logic is already wired through controllers: `AdminController`, `HotelController`, `GuestController`, `BookingController`, `SupportController`, and `LoyaltyController`.

Visual weak points:
- Dashboards do not share one visual system.
- Guest dashboard has no true dashboard/profile home view.
- Admin sidebar is functional but does not match the dark navy menu style from the reference images.
- Hotel dashboard uses tabs instead of the reference-style sidebar/header dashboard shell.
- KPI cards are plain white boxes rather than colorful dashboard cards.
- Overview pages lack boxed widgets such as recent activity, rankings, support snapshots, and profile/info blocks.

Redesign plan:
- Keep all backend, controllers, DAOs, and services intact.
- Add reusable Swing dashboard UI components for a dark sidebar, top header, KPI cards, white widgets, compact tables, and profile/info widgets.
- Restyle `AdminDashboardPanel`, `HotelDashboardPanel`, and `GuestDashboardPanel` around a shared `CardLayout` dashboard shell.
- Keep existing full-feature panels for management workflows, but place them inside the new shell navigation.
- Add no-scroll overview dashboards for admin, hotel, and client using compact KPI rows and boxed widgets.
- Build icons with bundled Swing drawing/text components only, avoiding external or missing image resources.
- Preserve every existing button action by reusing existing panels and controller methods.

Screens to redesign:
- Full layout restructure: `AdminDashboardPanel`, `HotelDashboardPanel`, `GuestDashboardPanel`.
- Shared style upgrade: `UITheme` plus new reusable dashboard component helpers.
- Existing workflow panels such as search, bookings, support, logs, reports, hotels, rooms, and users remain connected and are shown as secondary pages.

Compatibility strategy:
- Navigation changes only swap visible panels through `CardLayout`.
- Existing panels are reused for actual workflows, so booking/search/support/log/report logic remains connected.
- New overview widgets read data through existing controllers and models.
- No fake dashboard buttons will be added; menu items show existing pages or overview pages.

## UI Redesign Implementation

Files changed:
- `src/main/java/com/hotelbooking/view/DashboardUI.java`
- `src/main/java/com/hotelbooking/util/UITheme.java`
- `src/main/java/com/hotelbooking/view/AdminDashboardPanel.java`
- `src/main/java/com/hotelbooking/view/HotelDashboardPanel.java`
- `src/main/java/com/hotelbooking/view/GuestDashboardPanel.java`
- `src/main/java/com/hotelbooking/view/MainFrame.java`
- `README.md`
- `PROJECT_EXPLANATION.md`

Reference image translation:
- Yellow top brand block plus dark navy sidebar became the shared Pulga sidebar.
- Reference KPI cards became role-specific dashboard stat cards with green, red, blue, and yellow cards.
- Reference white content boxes became reusable dashboard widgets.
- Reference profile section became the client `My Information` card and hotel profile widget.
- Reference notice/result panels became compact support, booking, ranking, and log widgets.

No-scroll approach:
- Each dashboard home page uses one KPI row and a compact widget grid.
- Tables show only small recent/ranking snapshots on the dashboard.
- Full workflows remain in dedicated sidebar pages.

Compatibility:
- Existing controllers and panels remain connected.
- Search, booking, support, logs, reports, rooms, hotels, and users still use the existing backend methods.
- No external icon files were added; sidebar and KPI icons are bundled Swing text badges, avoiding missing resource issues.

Testing:
- `.\mvnw.cmd clean compile` passed after the UI redesign.
- Short `.\mvnw.cmd exec:java` launch smoke test stayed running for 8 seconds, confirming the app starts with the redesigned UI shell.
- The smoke-test Java processes were stopped afterward.

## LOGIC STABILIZATION AUDIT

Issue-to-code mapping:
- Booking transitions are currently controlled in `BookingService.cancelGuestBooking()`, `BookingService.adminConfirmBooking()`, `BookingService.adminCancelBooking()`, `AdminService.confirmBooking()`, `AdminService.cancelBooking()`, `HotelAccountService.acceptBooking()`, `HotelAccountService.cancelBooking()`, `HotelAccountService.validateCheckout()`, and low-level `BookingDAO.updateStatus()/acceptBooking()/cancelBooking()/validateCheckout()`.
- Loyalty point deduction/earning is controlled by `BookingService.createBooking()`, `LoyaltyService.usePoints()`, `LoyaltyService.earnForCompletedStay()`, and `LoyaltyDAO`.
- Report revenue calculations are mainly in `BookingDAO.sumConfirmedRevenue()`, `revenuePerHotel()`, `roomRevenueRanking()`, `hotelStats()`, `highestMonthRevenue()`, and `ReportService`.
- Audit logging is controlled by `AuditService`, called from `AuthService`, `BookingService`, `AdminService`, `HotelAccountService`, `SupportService`, `RatingService`, and some loyalty methods. Logout is currently not logged from `MainFrame`.
- Support compensation is controlled by `SupportService.updateTicket()`, `LoyaltyService.grantSupportCompensation()`, `LoyaltyDAO.hasSupportCompensation()`, and `SupportManagementPanel`.
- Room/hotel activation is controlled by `HotelAccountService.setOwnRoomActive()`, `AdminService.setRoomActive()`, and `AdminService.setHotelActive()`.
- Hotel account linking is controlled by `AdminService.addHotelUser()` and `HotelDAO.update()`.
- Seed generation is controlled by `DatabaseManager.seedData()`, especially `seedAdditionalBookings()` and related demo seed helpers.

Planned changes:
- Add a centralized booking transition policy so client, hotel, and admin actions share terminal-state and role-aware validation.
- Add checkout-date validation before completing stays.
- Make cancellation/rejection refund used points idempotently.
- Make support response + compensation atomic and replace text-matching compensation checks with a structured `support_ticket_id` history link.
- Block room/hotel deactivation while active bookings exist.
- Correct actual revenue to use final paid amount after points discounts and separate expected revenue from completed revenue.
- Add service-level past-date validation for searches.
- Add logout and admin-user-creation audit logs.
- Prevent silent hotel account link overwrite.
- Prevent unsafe admin self-disable and last-admin disable.
- Repair/fix seed generation so fresh demo data avoids active overlaps.

Preserved behavior:
- Existing MVC/DAO/service/controller structure remains.
- Existing Swing panels and dashboards remain, with only small clarity changes.
- Existing roles and existing seeded demo accounts remain.
- Booking search, booking creation, support, reports, logs, and room management remain connected to the same controllers.

## Logic Stabilization Implementation

Files changed:
- `src/main/java/com/hotelbooking/service/BookingTransitionPolicy.java`
- `src/main/java/com/hotelbooking/service/BookingService.java`
- `src/main/java/com/hotelbooking/service/HotelAccountService.java`
- `src/main/java/com/hotelbooking/service/AdminService.java`
- `src/main/java/com/hotelbooking/service/SupportService.java`
- `src/main/java/com/hotelbooking/service/LoyaltyService.java`
- `src/main/java/com/hotelbooking/service/HotelService.java`
- `src/main/java/com/hotelbooking/service/RoomService.java`
- `src/main/java/com/hotelbooking/service/ReportService.java`
- `src/main/java/com/hotelbooking/dao/BookingDAO.java`
- `src/main/java/com/hotelbooking/dao/LoyaltyDAO.java`
- `src/main/java/com/hotelbooking/dao/SupportTicketDAO.java`
- `src/main/java/com/hotelbooking/dao/UserDAO.java`
- `src/main/java/com/hotelbooking/dao/HotelDAO.java`
- `src/main/java/com/hotelbooking/dao/DatabaseManager.java`
- `src/main/java/com/hotelbooking/view/MainFrame.java`
- `src/main/java/com/hotelbooking/view/AdminDashboardPanel.java`
- `src/main/java/com/hotelbooking/view/HotelDashboardPanel.java`
- `src/main/java/com/hotelbooking/view/MyBookingsPanel.java`
- `README.md`
- `PROJECT_EXPLANATION.md`

Implemented fixes:
- Added `BookingTransitionPolicy` so admin, hotel, and client flows share the same state-machine checks.
- Terminal statuses `COMPLETED`, `CANCELLED`, and `REJECTED` can no longer be moved back into active booking states.
- Checkout validation now requires an accepted/live source status and `today >= check_out_date`.
- Booking creation with points now saves booking and point deduction in the same transaction.
- Cancellation/rejection now refunds used points in the same transaction and uses idempotent `REFUNDED_AFTER_CANCELLATION` history.
- Support ticket updates with compensation are now transactional; invalid compensation prevents the ticket update from being saved.
- Support compensation duplicate prevention now uses `loyalty_points_history.support_ticket_id` instead of description text matching.
- Room and hotel deactivation now blocks when active bookings exist.
- Completed revenue now uses `final_price_after_points`; expected revenue is separate and only counts accepted/live future bookings.
- Legacy no-point bookings with `final_price_after_points = 0` are normalized to `total_price` during migration, while real fully point-paid bookings can remain `0`.
- Search services reject past check-in dates even when called outside the UI.
- Logout is logged from `MainFrame`.
- Admin user creation is audited.
- Hotel account creation blocks existing hotel links and performs user insert plus hotel link in one transaction.
- Admin self-disable and last-active-admin disable are blocked.
- Seed generation now avoids active room/user overlaps and a small demo-only repair cancels later duplicate active seeded overlaps.

Business-rule decisions:
- Minimal lifecycle is enforced as `PENDING -> ACCEPTED/REJECTED/CANCELLED`, `ACCEPTED -> CANCELLED/COMPLETED`, with terminal statuses stable.
- `CHECKED_IN` and `CHECKOUT_PENDING_VALIDATION` remain recognized as active/live statuses for availability and checkout validation, but normal UI flows still use the simpler pending/accepted/completed path.
- Actual revenue means completed realized cash amount after points discount.
- Expected revenue means accepted/live upcoming value after points discount.
- Points used at booking time are reserved immediately, then refunded automatically if the booking is cancelled or rejected before completion.
- Commission and earned loyalty points are based on final payable value after points discount.

Seed cleanup strategy:
- Fresh seed data now checks active booking overlaps before inserting accepted/pending demo bookings.
- Existing demo databases with 150 or fewer bookings get a conservative repair: later duplicate active overlaps are marked `CANCELLED` with an explicit demo cleanup reason.
- Completed and historical bookings are not modified by the overlap repair.

Testing:
- `.\mvnw.cmd clean compile` passed after the stabilization changes.
- Temporary Java probes verified active seed overlaps are repaired to `0` room conflicts and `0` user conflicts.
- Verified completed bookings cannot be cancelled or accepted again.
- Verified a newly-created future accepted booking cannot be checkout-validated before checkout date.
- Verified past-date search is blocked at service level.
- Verified a point-backed client booking deducts points on creation and refunds them on cancellation.
- Verified active room deactivation is blocked.
- Verified invalid support compensation does not update the support ticket.
- Short `.\mvnw.cmd exec:java` launch smoke test stayed running for 8 seconds and was stopped afterward.

## Points Booking Validation Fix

User-reported issue:
- In the booking confirmation dialog, typing more loyalty points than the client balance could still confirm the booking because the `JSpinner` kept the invalid typed text uncommitted and `getValue()` returned the previous valid value.

Fix:
- Updated `BookingPanel.confirmBooking()` to parse and validate the visible points text before creating the booking.
- Added clear errors for blank input, non-number input, negative points, points above the client balance, and points above the maximum payable amount.
- The booking is not created until the typed points value is valid.

Testing:
- `.\mvnw.cmd clean compile` passed after the fix.

## User / Hotel Management Improvement Implementation

Files modified:
- `DatabaseManager`, `UserDAO`, `RoomDAO`, `HotelDAO`, and `LoyaltyDAO`
- `Room`
- `AdminService`, `HotelAccountService`, `AdminController`, and `BookingService`
- `TableSortUtil`
- `AdminDashboardPanel`, `DashboardUI`, `UserManagementPanel`, `RoomManagementPanel`, `HotelDashboardPanel`, `HotelManagementPanel`, `BookingManagementPanel`, `AuditLogPanel`, `SupportManagementPanel`, `SupportPanel`, `SearchPanel`, `HotelDetailsPanel`, `BookingPanel`, `MyBookingsPanel`, `LoyaltyPanel`, and `ReportPanel`
- `README.md`, `PROJECT_EXPLANATION.md`, and `BUILD_LOG.md`

Implemented changes:
- Renamed the admin navigation from `Clients` to `Users` and changed the users screen title to `Users`.
- Removed the visible Add Admin action and blocked admin creation in `AdminService`.
- Enforced the one-admin rule: the app supports exactly one administrator account, `admin / admin123`.
- Added database cleanup so the default `admin` stays active with the expected password and extra `ADMIN` rows are deactivated.
- Protected admin rows from user edit, enable/disable, and delete/deactivate actions.
- Added admin creation for CLIENT users and HOTEL users. Hotel-user creation links to an existing unclaimed hotel and does not overwrite an existing hotel account link.
- Added admin user editing for username, full name, email, phone, and active status while preserving role restrictions.
- Added safe delete behavior by deactivating CLIENT/HOTEL accounts rather than physically deleting rows with booking/support/log history.
- Added direct admin give/take points actions for CLIENT users only. Both actions update balances, write `loyalty_points_history`, and write audit logs.
- Added linked-hotel inspection to the Users table and Inspect dialog.
- Added room discount fields: `rooms.discount_percent` and `rooms.promotion_active`.
- Added percentage discount controls to admin room management and hotel own-room management.
- Updated booking/search/room display logic to use final discounted room price where booking totals or available prices are shown.
- Kept `HotelAccountService` as the hotel ownership gate so hotel users can manage only their own rooms.
- Added `TableSortUtil` and installed it on major list/table screens for column-click organization and ID-ascending defaults.
- Updated selected-row handling after sorting so actions operate on the correct backing object.

Discount/pricing approach:
- A room can have a promotion percentage from 0 to 100.
- If promotion is active, final price is `base_price * (1 - discount_percent / 100)`.
- If promotion is inactive, final price is the base price.
- Booking totals and search max-price filtering use the final room price.

Sorting approach:
- `TableSortUtil` uses `TableRowSorter`.
- It compares `Number` values directly and parses money strings such as `850.00 MAD`, percentages, dates, date-times, enums, and text.
- Tables with an ID column open sorted by ID ascending.

Compile/test notes:
- `.\mvnw.cmd clean compile` passed after implementation.
- A short `.\mvnw.cmd exec:java` launch smoke test stayed running for 8 seconds and was stopped afterward.
