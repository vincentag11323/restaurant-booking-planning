# Workflow
**ResortOS AI — Restaurant Booking App**

## 1. Registration & Table Reservation Flow (Guest)
1. Guest opens the web app and **Registers** (Email + Password).
2. Guest completes their **Profile** (Full Name, Phone Number, Allergy Preferences).
3. Guest opens **Book a Table** and selects a **Branch** (`BRANCH_1` / `BRANCH_2`) and **Sector** (`MAIN_HALL` / `VIP_WING`).
4. Guest selects a date and one of the fixed **2.5-hour time slots** that still has capacity.
5. Guest enters **Party Size** (1–50) and optional **Special Requests / Dietary Notes**.
6. System verifies capacity, auto-allocates tables (Main Hall: ⌈party ÷ 2⌉ two-seat tables; VIP Wing: 1–4 rooms) and issues a **Reservation ID**.
7. System sends a **Booking Confirmation** to the guest.

## 2. Reservation Management Flow (Guest)
1. Guest opens **My Bookings** to see upcoming and past reservations (slot window, table/room codes, party size, status).
2. Guest can **Reschedule** a `CONFIRMED` booking before its slot starts. The system re-checks capacity for the new slot. Old tables are only released once the new ones are secured.
3. Guest can **Cancel** a `CONFIRMED` booking before its slot starts. This is a soft cancel: the record stays with status `CANCELLED` and its tables are released.

## 3. At the Restaurant (Guest & Staff Interaction)
1. Guest arrives and shows the **Reservation ID** to the host.
2. Host looks up the Reservation ID in the **Admin Booking Logs** (read-only) to verify the guest, the party size and the assigned tables/rooms.
3. Guest is seated at the allocated tables/rooms.
4. *(Future — not in v1)* A host-stand / POS integration moves the status to `SEATED` → `COMPLETED`, or `NO_SHOW`. The Admin app stays read-only.

## 4. Operations Monitoring Flow (Admin — Read Only)
1. Admin logs in. Admin accounts are created by seed script only; there is no public sign-up.
2. Admin reviews the **Dashboard**: bookings today, total covers, cancellations, sector utilization.
3. Admin filters and searches the **Booking Log** by branch, date range, status, sector, name, email, phone or Reservation ID.
4. Admin opens the **Detail Drawer** to inspect a single reservation.
5. Admin **Exports** the filtered log to CSV/JSON for accounting. Every export is recorded in the Audit Trail.
