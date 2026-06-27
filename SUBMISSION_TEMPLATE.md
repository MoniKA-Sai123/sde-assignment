# Post-Call Processing Pipeline — Design Document

**Author:** S N Monika
**Date:** 27 June 2026

---

# 1. Assumptions

The following assumptions were made while designing the solution:

1. The platform processes approximately 100,000 completed calls during peak campaign periods.
2. Each interaction belongs to exactly one customer and one campaign.
3. The LLM provider enforces both request-per-minute (RPM) and token-per-minute (TPM) limits.
4. Customers may have different subscription plans, resulting in different processing priorities.
5. High-value outcomes (for example, successful sales or callback requests) require immediate processing, while low-priority interactions can be deferred.
6. Call recordings may not be immediately available after a call ends.
7. Temporary infrastructure failures (Redis, worker restart, network issues) should not result in permanent data loss.

---

# 2. Problem Diagnosis

The current implementation works for small workloads but fails during high-volume campaign runs.

The major issues are:

* Every completed call immediately triggers an LLM request without considering API limits.
* A fixed `asyncio.sleep(45)` assumes recordings become available after exactly 45 seconds, causing missed recordings when delays occur.
* Redis is used as the task broker, so worker failures or Redis restarts may cause task loss.
* The circuit breaker completely pauses outbound processing instead of reducing processing gradually.
* Limited logging makes production debugging difficult.

The root problem is the absence of rate-limit-aware scheduling and durable processing.

---

# 3. Architecture Overview

```
Call End Webhook
        │
        ▼
Persist Interaction
        │
        ▼
Priority Queue
        │
        ▼
Rate Limit Scheduler
        │
 ┌──────┴─────────┐
 │                │
 ▼                ▼
Urgent Queue   Deferred Queue
 │                │
 └──────┬─────────┘
        ▼
Recording Poller
        ▼
LLM Processing
        ▼
CRM Update
        ▼
Audit Logging
        ▼
Completed
```

### Key Design Decisions

1. Introduced a centralized rate-limit-aware scheduler before every LLM request.
2. Added per-customer token budgeting to prevent one customer from consuming all available capacity.
3. Replaced the fixed recording delay with retry polling using exponential backoff.
4. Added structured audit logging for complete interaction traceability.

---

# 4. Rate Limit Management

The scheduler maintains the available request-per-minute and token-per-minute budget.

Before sending an LLM request, it checks:

* Remaining requests
* Remaining token budget
* Customer allocation
* Request priority

If sufficient capacity exists, the request is processed immediately.

Otherwise, the request remains queued until capacity becomes available.

This prevents API rate-limit errors and avoids unnecessary retries.

---

# 5. Per-Customer Token Budgeting

Each customer receives a configurable token allocation.

Example:

* Customer A → 40%
* Customer B → 30%
* Customer C → 20%
* Shared Pool → 10%

Customers always receive their reserved allocation.

If unused capacity exists, it can temporarily be borrowed by other customers.

Once the original customer becomes active again, its reserved capacity is restored.

---

# 6. Differentiated Processing

Interactions are divided into two categories.

**Immediate Processing**

* Successful sales
* Callback requests
* Escalations
* High-value customers

**Deferred Processing**

* No answer
* Spam
* Short conversations
* Low-priority interactions

Urgent interactions receive higher scheduling priority while deferred work is processed whenever capacity becomes available.

---

# 7. Recording Pipeline

Instead of waiting a fixed 45 seconds, the recording service continuously polls for recording availability.

Retry intervals increase using exponential backoff.

Example:

* Retry after 5 seconds
* Retry after 10 seconds
* Retry after 20 seconds
* Retry after 40 seconds

If the recording still cannot be retrieved, a structured error event is generated and the interaction is marked for manual investigation.

---

# 8. Reliability & Durability

Processing state is stored persistently.

Each interaction moves through defined states:

* Pending
* Processing
* Completed
* Failed
* Retrying

Workers can safely resume unfinished work after failures.

Every failed operation remains visible until successfully completed or manually resolved.

---

# 9. Auditability & Observability

Each interaction receives a unique correlation ID.

Every processing stage generates a structured log entry.

### Logged Fields

* interaction_id
* customer_id
* campaign_id
* processing_stage
* timestamp
* retry_count
* worker_id
* status
* error_message

### Alert Conditions

* Recording unavailable after maximum retries
* Rate-limit utilization above 90%
* Excessive processing queue length
* Consecutive worker failures
* CRM synchronization failures

---

# 10. Data Model

```sql
CREATE TABLE interaction_processing (
    interaction_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    campaign_id UUID NOT NULL,
    priority VARCHAR(20),
    status VARCHAR(30),
    retry_count INT DEFAULT 0,
    estimated_tokens INT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE TABLE audit_logs (
    id SERIAL PRIMARY KEY,
    interaction_id UUID,
    event_type VARCHAR(100),
    event_time TIMESTAMP,
    details JSONB
);
```

---

# 11. Security

Sensitive information includes:

* Customer information
* Phone numbers
* Call recordings
* Conversation transcripts
* CRM data

Protection measures:

* HTTPS for all communication
* Encryption at rest
* Role-based access control
* Limited log exposure for sensitive information
* Secure credential management using environment variables

---

# 12. API Interface

The existing `POST /session/{sid}/interaction/{iid}/end` endpoint is retained.

Keeping the existing API avoids breaking existing telephony integrations while allowing internal processing improvements without affecting external clients.

---

# 13. Trade-offs & Alternatives Considered

| Option                     | Why Considered        | Why Rejected / What Was Chosen     |
| -------------------------- | --------------------- | ---------------------------------- |
| Immediate LLM execution    | Simple implementation | Causes API rate-limit failures     |
| Fixed recording delay      | Easy implementation   | Misses delayed recordings          |
| Binary circuit breaker     | Protects LLM          | Stops all processing unnecessarily |
| Rate-limit-aware scheduler | Prevents overload     | Selected as the preferred approach |

---

# 14. Known Weaknesses

* Token estimation may not perfectly match actual LLM usage.
* Very large customer spikes may temporarily increase processing delay.
* Scheduler introduces additional implementation complexity.
* Advanced customer prioritization policies could be further improved.

---

# 15. What I Would Do With More Time

1. Implement adaptive scheduling based on historical traffic patterns.
2. Add automatic worker scaling during campaign peaks.
3. Build dashboards for queue health and rate-limit monitoring.
4. Add configurable customer priority policies through an admin interface.
5. Improve retry strategies for CRM integrations and external services.
