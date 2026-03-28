# Common Requirements Patterns

Reusable requirement templates for typical system features. Adapt these to the specific project — they're starting points, not copy-paste solutions.

## Table of Contents

1. [Authentication and Authorization](#authentication-and-authorization)
2. [CRUD Operations](#crud-operations)
3. [Search and Filtering](#search-and-filtering)
4. [Notifications](#notifications)
5. [File Handling](#file-handling)
6. [Payments](#payments)
7. [Reporting and Analytics](#reporting-and-analytics)
8. [Audit and Logging](#audit-and-logging)
9. [Integration with External Services](#integration-with-external-services)
10. [Batch Processing](#batch-processing)

---

## Authentication and Authorization

### Registration

| ID | Name | Description | Priority |
|----|------|-------------|----------|
| RF-AUTH-001 | User Registration | The system shall allow new users to register by providing [required fields]. [Password complexity rules]. | High |
| RF-AUTH-002 | Email Verification | The system shall send a verification email upon registration. The verification link shall expire after [time period]. | High |
| RF-AUTH-003 | Duplicate Prevention | The system shall reject registration if the email is already associated with an existing account. | High |

### Login and Session

| ID | Name | Description | Priority |
|----|------|-------------|----------|
| RF-AUTH-010 | User Login | The system shall authenticate users with [credentials] and issue a [token type] valid for [duration]. | High |
| RF-AUTH-011 | Failed Login Lockout | The system shall lock the account after [N] consecutive failed login attempts for [duration]. | High |
| RF-AUTH-012 | Password Reset | The system shall allow users to reset their password via a time-limited email link (expires in [time]). | High |
| RF-AUTH-013 | Session Invalidation | The system shall invalidate all active sessions when the user changes their password. | Medium |

### Authorization

| ID | Name | Description | Priority |
|----|------|-------------|----------|
| RF-AUTH-020 | Role-Based Access | The system shall enforce role-based access control with the following roles: [role list with permissions]. | High |
| RF-AUTH-021 | Resource Ownership | The system shall restrict access to resources so that users can only [view/edit/delete] their own [resource], unless they have [admin/manager] role. | High |

---

## CRUD Operations

Template for any managed entity. Replace `<Entity>` with the actual name.

| ID | Name | Description | Priority |
|----|------|-------------|----------|
| RF-[MOD]-001 | Create [Entity] | The system shall allow [actor] to create a new [entity] by providing [required fields]. [Validation rules]. | High |
| RF-[MOD]-002 | View [Entity] | The system shall display [entity] details including [field list] to [authorized actors]. | High |
| RF-[MOD]-003 | List [Entities] | The system shall display a paginated list of [entities] showing [summary fields], sorted by [default sort] with [page size] items per page. | High |
| RF-[MOD]-004 | Update [Entity] | The system shall allow [actor] to update [editable fields] of an existing [entity]. [Validation rules]. | High |
| RF-[MOD]-005 | Delete [Entity] | The system shall allow [actor] to [soft/hard] delete a [entity]. [Confirmation or conditions required]. | Medium |

### Validation Pattern

For each create/update operation, specify:
- Required vs optional fields
- Format constraints (email format, phone pattern, max length)
- Business rules (date ranges, value limits, uniqueness)
- Error messages for each validation failure

---

## Search and Filtering

| ID | Name | Description | Priority |
|----|------|-------------|----------|
| RF-SRCH-001 | Text Search | The system shall allow users to search [entities] by [searchable fields] using partial text matching. Results shall be returned within [time] for datasets up to [size]. | High |
| RF-SRCH-002 | Filter by Criteria | The system shall allow filtering [entities] by: [filter list with types — date range, enum, numeric range, etc.]. Multiple filters shall be combinable (AND logic). | Medium |
| RF-SRCH-003 | Sort Results | The system shall allow sorting results by [sortable fields] in ascending or descending order. | Medium |
| RF-SRCH-004 | Pagination | The system shall paginate results with configurable page size (default: [N], max: [M]). Response shall include total count and page metadata. | High |

---

## Notifications

| ID | Name | Description | Priority |
|----|------|-------------|----------|
| RF-NOTIF-001 | Email Notifications | The system shall send email notifications for the following events: [event list with template description]. | High |
| RF-NOTIF-002 | In-App Notifications | The system shall display real-time in-app notifications for [events]. Unread count shall be visible in the UI. | Medium |
| RF-NOTIF-003 | Notification Preferences | The system shall allow users to configure their notification preferences per channel (email, in-app, push) and event type. | Low |
| RF-NOTIF-004 | Notification History | The system shall retain notification history for [retention period] and allow users to view past notifications. | Low |

---

## File Handling

| ID | Name | Description | Priority |
|----|------|-------------|----------|
| RF-FILE-001 | File Upload | The system shall allow [actor] to upload files of type [allowed types] with a maximum size of [size]. Files shall be scanned for [malware/validity] before storage. | High |
| RF-FILE-002 | File Download | The system shall allow [authorized actors] to download files. Downloads shall be served via [presigned URLs / streaming] with appropriate Content-Type headers. | High |
| RF-FILE-003 | File Storage | The system shall store uploaded files in [storage system] organized by [structure]. File metadata (name, size, type, upload date, uploader) shall be persisted in the database. | High |
| RF-FILE-004 | Image Processing | The system shall generate thumbnails of [dimensions] for uploaded images and serve optimized versions based on the requesting device. | Medium |

---

## Payments

| ID | Name | Description | Priority |
|----|------|-------------|----------|
| RF-PAY-001 | Payment Processing | The system shall process payments via [payment gateway] supporting [payment methods]. | High |
| RF-PAY-002 | Payment Confirmation | The system shall display a payment confirmation with transaction ID, amount, and date upon successful processing. A confirmation email shall also be sent. | High |
| RF-PAY-003 | Refund Processing | The system shall allow [actor] to initiate refunds within [time period] of the original transaction. Refunds shall be processed to the original payment method. | Medium |
| RF-PAY-004 | Payment History | The system shall display a chronological list of all transactions for a user, including status (pending, completed, failed, refunded), amount, and date. | Medium |
| RF-PAY-005 | Idempotent Transactions | The system shall prevent duplicate charges by using idempotency keys. If a payment request is retried with the same key, it shall return the original result. | High |

---

## Reporting and Analytics

| ID | Name | Description | Priority |
|----|------|-------------|----------|
| RF-RPT-001 | Dashboard | The system shall display a dashboard with [KPI list] for the [time period], updated [frequency]. | Medium |
| RF-RPT-002 | Report Generation | The system shall allow [actor] to generate reports for [report types] with configurable date ranges and filters. Reports shall be exportable as [formats: PDF, CSV, Excel]. | Medium |
| RF-RPT-003 | Scheduled Reports | The system shall allow [actor] to schedule recurring reports delivered via email at [frequency options]. | Low |

---

## Audit and Logging

| ID | Name | Description | Priority |
|----|------|-------------|----------|
| RF-AUD-001 | Audit Trail | The system shall log all [create/update/delete] operations on [audited entities] with: timestamp, user ID, action type, previous value, and new value. | High |
| RF-AUD-002 | Audit Log Access | The system shall allow [authorized actors] to view the audit trail filtered by entity, user, date range, and action type. | Medium |
| RF-AUD-003 | Immutable Logs | Audit logs shall be immutable — once written, they cannot be modified or deleted by any user, including administrators. | High |
| RF-AUD-004 | Log Retention | The system shall retain audit logs for [retention period] in compliance with [regulation]. Logs older than [period] shall be archived to [cold storage]. | Medium |

---

## Integration with External Services

| ID | Name | Description | Priority |
|----|------|-------------|----------|
| RF-INT-001 | [Service] Integration | The system shall integrate with [external service] via [API type: REST/gRPC/SOAP/webhook] to [purpose]. | High |
| RF-INT-002 | Retry Policy | When [external service] is unavailable, the system shall retry [N] times with [backoff strategy] before marking the operation as failed and [fallback action]. | High |
| RF-INT-003 | Circuit Breaker | The system shall implement circuit breaker pattern for [service] calls: open after [N] consecutive failures, half-open after [duration], close after [N] successful probes. | Medium |
| RF-INT-004 | Webhook Handling | The system shall accept webhooks from [service] at endpoint [path], validate the signature using [method], and process events [synchronously/asynchronously]. | Medium |

---

## Batch Processing

| ID | Name | Description | Priority |
|----|------|-------------|----------|
| RF-BATCH-001 | Batch Import | The system shall allow [actor] to import [entity] records from [file format] files containing up to [volume] records. Invalid records shall be logged and skipped without aborting the batch. | Medium |
| RF-BATCH-002 | Batch Export | The system shall allow [actor] to export [entity] data as [format] with filters for [criteria]. Export shall run asynchronously and notify the user when complete. | Medium |
| RF-BATCH-003 | Scheduled Jobs | The system shall execute [job name] at [schedule/cron expression] to [purpose]. Job execution status and duration shall be logged. | Medium |
| RF-BATCH-004 | Job Monitoring | The system shall provide visibility into batch job status (running, completed, failed), execution time, records processed, and error count. | Low |
