# Database Schema & Audit Tables
## Email Integration Platform

**Organization:** XYZ  
**Document Type:** Database Design  
**Scope:** Operational & Audit Persistence  
**Audience:** Architects, Developers, Security, Audit  

---

## 1. Purpose

This document defines the **database schema** for the Email Integration Platform, with a strong focus on **auditability, traceability, and operational diagnostics**.

The schema is designed to:
- Support asynchronous email processing
- Enable full lifecycle tracking
- Meet audit and compliance requirements
- Avoid storage of sensitive email content

---

## 2. Design Principles

- Append-only audit records
- No PII-heavy payload storage
- Correlation ID as primary trace key
- Immutable status transitions
- Optimized for read-heavy audit queries

---

## 3. Logical Data Model Overview

```text
Email Request
   |
   v
Email Message
   |
   v
Email Status History
   |
   v
Vendor Interaction
```

---

## 4. Tables Overview

| Table Name | Purpose |
|----------|--------|
| email_message | Core email request metadata |
| email_status_history | Status lifecycle tracking |
| vendor_interaction | Vendor-level delivery details |
| audit_event | Security & compliance audit trail |
| idempotency_key | Duplicate request prevention |

---

## 5. Table Definitions

---

### 5.1 email_message

Stores canonical metadata for each email request.

```sql
CREATE TABLE email_message (
  message_id           VARCHAR(64) PRIMARY KEY,
  correlation_id       VARCHAR(64) NOT NULL,
  source_system        VARCHAR(64) NOT NULL,
  email_type           VARCHAR(32) NOT NULL,
  priority             VARCHAR(16),
  created_at           TIMESTAMP NOT NULL,
  received_at          TIMESTAMP NOT NULL,
  current_status       VARCHAR(32) NOT NULL
);
```

**Notes**
- No email body or attachments stored
- Indexed by `message_id` and `correlation_id`

---

### 5.2 email_status_history

Tracks all state transitions for an email.

```sql
CREATE TABLE email_status_history (
  id                   BIGSERIAL PRIMARY KEY,
  message_id           VARCHAR(64) NOT NULL,
  status               VARCHAR(32) NOT NULL,
  status_reason        VARCHAR(256),
  occurred_at          TIMESTAMP NOT NULL,
  FOREIGN KEY (message_id) REFERENCES email_message(message_id)
);
```

**Example statuses**
- RECEIVED  
- VALIDATED  
- SENT  
- DELIVERED  
- FAILED  
- RETRIED  

---

### 5.3 vendor_interaction

Stores vendor-level delivery attempts.

```sql
CREATE TABLE vendor_interaction (
  id                   BIGSERIAL PRIMARY KEY,
  message_id           VARCHAR(64) NOT NULL,
  vendor_name          VARCHAR(64) NOT NULL,
  vendor_message_id    VARCHAR(128),
  response_code        VARCHAR(32),
  response_message     VARCHAR(256),
  attempt_number       INT NOT NULL,
  sent_at              TIMESTAMP,
  response_at          TIMESTAMP,
  FOREIGN KEY (message_id) REFERENCES email_message(message_id)
);
```

---

### 5.4 audit_event

Immutable audit trail for compliance.

```sql
CREATE TABLE audit_event (
  id                   BIGSERIAL PRIMARY KEY,
  correlation_id       VARCHAR(64) NOT NULL,
  event_type           VARCHAR(64) NOT NULL,
  actor                VARCHAR(64),
  event_description    VARCHAR(512),
  event_timestamp      TIMESTAMP NOT NULL
);
```

**Examples**
- EMAIL_RECEIVED
- EMAIL_SENT
- VENDOR_FAILURE
- AUTH_FAILURE
- CONFIG_CHANGE

---

### 5.5 idempotency_key

Prevents duplicate email submissions.

```sql
CREATE TABLE idempotency_key (
  idempotency_key      VARCHAR(128) PRIMARY KEY,
  message_id           VARCHAR(64) NOT NULL,
  created_at           TIMESTAMP NOT NULL
);
```

---

## 6. Indexing Strategy

```sql
CREATE INDEX idx_email_message_corr ON email_message(correlation_id);
CREATE INDEX idx_status_message ON email_status_history(message_id);
CREATE INDEX idx_vendor_message ON vendor_interaction(message_id);
CREATE INDEX idx_audit_corr ON audit_event(correlation_id);
```

---

## 7. Retention & Archival

| Table | Retention |
|-----|-----------|
| email_message | 90 days |
| email_status_history | 1 year |
| vendor_interaction | 1 year |
| audit_event | 7 years |
| idempotency_key | 7 days |

---

## 8. Security Considerations

- No email body persistence
- No raw PII storage
- Access via service accounts only
- Read-only access for audit users
- Encryption at rest enabled

---

## 9. Operational Queries (Examples)

```sql
-- Check email delivery status
SELECT current_status
FROM email_message
WHERE message_id = ?;

-- Full lifecycle
SELECT *
FROM email_status_history
WHERE message_id = ?
ORDER BY occurred_at;

-- Vendor failures
SELECT *
FROM vendor_interaction
WHERE response_code != '200';
```

---

## 10. Conclusion

This schema provides:
- Complete lifecycle traceability
- Strong compliance alignment
- Minimal sensitive data footprint
- Operational transparency

It supports high-throughput workloads while remaining audit-ready.

---

**Status:** Approved  
