# Audit Trail System
**ResortOS AI — Restaurant Booking App**

Every critical action in the app must be recorded for dispute resolution (e.g. "I never cancelled my booking"), operational accountability and personal-data compliance.

> Note: the Admin **Booking Log** (the reservation list in the PRD) is a business view. The **Audit Trail** described here is a separate, append-only security log stored in the `AuditLog` table.

## 1. Stored Log Format
Each activity log entry contains:
* `timestamp`: When the event happened (server time, UTC).
* `actorId`: ID of the User or Admin performing the action (`null` for anonymous events such as a failed login).
* `action`: Action type. Examples:
  * Guest: `REGISTER`, `LOGIN_SUCCESS`, `LOGIN_FAILED`, `UPDATE_PROFILE`, `CREATE_RESERVATION`, `RESCHEDULE_RESERVATION`, `CANCEL_RESERVATION`
  * Admin: `VIEW_RESERVATION_DETAIL`, `EXPORT_RESERVATIONS`
  * System: `SECURITY_ALERT`
* `resourceId`: ID of the entity changed or accessed (e.g. Reservation ID).
* `ipAddress` & `userAgent`: Device and network information.
* `metadata`: Small JSON payload with context (e.g. old/new slot on reschedule, export row count). Passwords and tokens are **never** logged.

## 2. Retention Policy
* Audit logs are **immutable**. A PostgreSQL trigger rejects every `UPDATE` and `DELETE` on the `AuditLog` table, for every role, including Admin.
* Logs are kept for at least **2 years** (covers typical chargeback/dispute windows and UU PDP accountability requirements). After that they are archived or purged by a DBA-run job outside the app.

## 3. Security Alerting
* The system detects anomalies and writes a `SECURITY_ALERT` audit entry plus a server warning log. In v1 the log is the notification channel. Email/Slack delivery is a future enhancement.
* Anomaly rules (v1):
  * **10 failed login attempts from the same IP within 5 minutes.**
  * **A single Admin export of more than 1,000 booking records** (possible bulk personal-data extraction).
