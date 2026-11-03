# Product Requirement Document (PRD)
**ResortOS AI — Hospitality Enterprise Platform**  
**Version:** 1.0  
**Role Focus:** User (Diner App), Admin (Booking Logs Only)  
**Tech Stack:** 100% TypeScript | Express.js API | React Frontend (Decoupled 2 Repositories)

---

## 1. Product Summary
ResortOS AI is a digital hospitality platform enabling restaurant guests (Users) to effortlessly discover menu staples, reserve tables, and manage their dining schedules across two premier locations (Branch 1 & Branch 2). Simultaneously, it provides restaurant administrators (Admin) with a streamlined, read-only audit log dashboard to view and track daily booking records across all dining sectors.

---

## 2. Objectives
* **Streamline Table Reservations:** Allow guests to book tables online while dynamically enforcing capacity and duration constraints (2.5-hour time blocks).
* **Automate Table Allocation:** Support automatic grouping of 2-seat tables for groups up to 100 guests in the Main Dining Hall, and modular room partitioning for up to 50 guests in the VIP Private Dining Sector.
* **Auditability & Operations Monitoring:** Provide administrative staff with real-time, read-only visibility into booking logs across both branches without allowing manual override clutter.
* **Maintain Architectural Simplicity (KISS):** Build a performant, decoupled architecture using Express.js and React with Playwright automated test suites.

---

## 3. User Personas & Key Features

### A. User (Diner App)
* **Authentication:**
  * Guest Registration & Login (Email/Password or Passcode).
  * Profile Management (Contact details, allergy preferences, spending history overview).
* **Table Booking Engine:**
  * Branch Selection (`BRANCH_1` or `BRANCH_2`).
  * Date and Time-Slot Selection (Strict 2.5-hour fixed duration blocks).
  * Party Size Input (1 to 50 guests).
  * Dynamic Allocation Rules:
    * **Main Hall ($\le 100$ pax):** Allocates $\lceil \text{Party Size} / 2 \rceil$ joined tables.
    * **VIP Wing ($\le 50$ pax):** Merges 1 to 4 VIP rooms based on group size.
  * Special Requests & Dietary Notes field.
* **Reservation Management:**
  * View Active Bookings (Upcoming time slot, assigned table/sector code, party size).
  * Reschedule Booking (Subject to slot capacity re-verification).
  * Soft-Cancel Reservation.
* **Digital Menu & Recommendations:**
  * Explore seasonal fine dining vs. à la carte menus (Chicken Steak, Beef Steak, Barramundi, Fish & Chips, Burgers).
  * AI-driven recommendations based on allergy and historical spending profiles.
* **Notifications:**
  * Booking confirmation alerts and 2.5-hour slot reminders.

### B. Admin (Booking Logs App — READ ONLY)
* **Dashboard & Summary Stats:**
  * Overview metrics: Total bookings today, total covers/guests, active cancellations, sector utilization rates.
* **Booking Log Viewer (Audit Trail):**
  * Real-time read-only log table displaying all historical and upcoming reservations.
  * Filter logs by: Branch (`BRANCH_1`, `BRANCH_2`), Date Range, Status (`CONFIRMED`, `SEATED`, `COMPLETED`, `CANCELLED`, `NO_SHOW`), and Sector (`MAIN_HALL`, `VIP_WING`).
  * Search logs by Customer Name, Email, Phone Number, or Reservation ID.
* **Log Detail View:**
  * Inspection drawer showing detailed audit data: Customer details, assigned table IDs/room codes, booking timestamp, slot window ($Start \rightarrow End$), and special notes.
* **Log Export:**
  * One-click export of filtered booking logs to CSV/JSON format for operational accounting.

---

## 4. Non-Functional & Architecture Requirements

### 4.1 System Structure (2 Loose Repositories)
1. **Backend Repository (`resortos-booking-api`):**
   * Express.js framework written strictly in 100% TypeScript.
   * PostgreSQL database managed via Prisma ORM.
   * Zod schema validation for all HTTP routes.
2. **Frontend Repository (`resortos-booking-web`):**
   * React (Vite) single-page app in 100% TypeScript.
   * Houses both the User App view and the Admin Log Viewer.
   * Styled cleanly using Tailwind CSS.

### 4.2 API Specifications & Error Handling
* Standardized HTTP status error structure:
  * `400 Bad Request`: Used for invalid payload input, capacity overflow, and table overlap conflicts (differentiated by explicit JSON `message` strings).
  * `404 Not Found`: Used strictly when a requested reservation record does not exist.
  * `200 OK` / `201 Created`: Successful query execution.

### 4.3 Quality Assurance & Playwright Testing
* Automated Playwright test suites mandated for both repositories:
  * **API Suite:** Validates full CRUD cycle, 2.5-hour slot computation, and 400/404 error code triggers.
  * **Frontend UI Suite:** Simulates user reservation flows, booking list interactions, form validations, and read-only admin filter operations.