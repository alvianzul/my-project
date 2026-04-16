# IBM MQ Backout Queue (BOQ) — Implementation & Operations Runbook

> **Document Type:** Operational Runbook  
> **Applies To:** IBM MQ — Distributed Platforms  
> **Status:** ![Active](https://img.shields.io/badge/Status-Active-brightgreen)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Definitions](#2-definitions)
3. [Preconditions](#3-preconditions)
4. [Design Overview](#4-design-overview)
5. [Implementation Procedure](#5-implementation-procedure)
6. [Application Behaviour Requirements](#6-application-behaviour-requirements)
7. [Verification Steps](#7-verification-steps)
8. [Runtime and Performance Considerations](#8-runtime-and-performance-considerations)
9. [Monitoring and Alerting](#9-monitoring-and-alerting)
10. [Operational Handling of Backout Messages](#10-operational-handling-of-backout-messages)
11. [Rollback / Disable Procedure](#11-rollback--disable-procedure)
12. [Risks and Mitigations](#12-risks-and-mitigations)
13. [Summary](#13-summary)
14. [Operational Checklist](#14-operational-checklist)
- [Appendix A — Reference Diagrams](#appendix-a--reference-diagrams)
- [Appendix B — Reference Links](#appendix-b--reference-links)

---

## 1. Overview

| Field | Value |
|---|---|
| **Runbook Name** | IBM MQ Backout Queue Management |
| **Purpose** | Provide a standardized procedure to configure, operate, and monitor IBM MQ Backout Queues to safely handle poison messages and prevent performance degradation. |
| **Applies To** | IBM MQ on Distributed Platforms — Native MQI and JMS-based applications using transactional (syncpoint) message processing |

---

## 2. Definitions

| Term | Description |
|---|---|
| **Poison Message** | A message that causes repeated application failures and transaction rollbacks, creating a processing loop. |
| **BackoutCount** | An MQMD field automatically incremented by the queue manager each time the message is backed out (rolled back). |
| **BOTHRESH** | Queue attribute defining the maximum number of rollback attempts before the message is moved to the backout queue. |
| **BOQNAME** | Queue attribute specifying the name of the destination backout queue. |
| **DLQ** | Dead Letter Queue — fallback destination for messages that cannot be placed on their target queue. |

---

## 3. Preconditions

> ℹ️ All preconditions below must be met before proceeding with implementation.

- ✅ Queue Manager is running and accessible
- ✅ Application uses syncpoint (transactional) message processing
- ✅ Required IBM MQ administrator privileges are available
- ✅ Sufficient disk space available for backout and DLQ queues

---

## 4. Design Overview

The following steps describe the logical sequence when a poison message is encountered:

1. Application retrieves a message from the main queue under syncpoint.
2. Processing fails — the application issues a **ROLLBACK**.
3. The queue manager automatically increments `MQMD.BackoutCount`.
4. If `BackoutCount` exceeds `BOTHRESH`, the message is diverted to the configured backout queue (`BOQNAME`) and the unit of work is committed.
5. The application continues processing subsequent messages normally.

```
Application
    │
    ▼ MQGET + SYNCPOINT
MAIN.QUEUE
    │
    ├── [Success] ──► COMMIT
    │
    └── [Failure] ──► ROLLBACK
                          │
                          ▼
              Queue Manager increments BackoutCount
                          │
              ┌───────────▼────────────┐
              │  BackoutCount > BOTHRESH?│
              └────────────────────────┘
                    │           │
                   No          Yes
                    │           │
                    ▼           ▼
              (retry)    BOQNAME configured?
                               │           │
                              Yes          No
                               │           │
                               ▼           ▼
                        MAIN.QUEUE.BO   DLQ configured?
                               │           │         │
                               │          Yes        No
                               │           │         │
                               ▼           ▼         ▼
                        Operations   SYSTEM.DEAD. Message
                        Team         LETTER.QUEUE  Lost!
                        Investigation
```

> ⚠️ **Warning:** Non-transactional applications will not trigger backout processing. `MQGMO_SYNCPOINT` (or JMS transacted session) is **mandatory**.

---

## 5. Implementation Procedure

### 5.1 Define Required Queues

**Step 1:** Define the main application queue

```mqsc
DEFINE QLOCAL(MAIN.QUEUE)
```

**Step 2:** Define the dedicated backout queue

```mqsc
DEFINE QLOCAL(MAIN.QUEUE.BO)
```

> ⚠️ Do not share a backout queue between unrelated applications. Each application should have its own dedicated BOQ.

### 5.2 Configure Backout Attributes

Set the backout queue name and threshold on the main queue:

```mqsc
ALTER QLOCAL(MAIN.QUEUE) BOQNAME(MAIN.QUEUE.BO) BOTHRESH(5)
```

> 💡 **Recommended practice:** Start with `BOTHRESH(3–5)` and adjust based on the application's error-recovery characteristics and the business tolerance for retries.

### 5.3 Define Dead Letter Queue (Mandatory)

Configure a default DLQ at the queue manager level to prevent message loss if the backout queue is unavailable:

```mqsc
ALTER QMGR DEADQ(SYSTEM.DEAD.LETTER.QUEUE)
```

> ⚠️ The DLQ is a critical safety net. Without it, messages that cannot be moved to the backout queue may be **lost permanently**.

---

## 6. Application Behaviour Requirements

Applications must meet the following requirements for backout processing to function correctly:

- Retrieve messages using `MQGMO_SYNCPOINT` (or equivalent JMS transacted session / XA)
- Issue `ROLLBACK` on any processing failure
- Issue `COMMIT` on successful processing
- Not suppress `MQRC_BACKED_OUT` return codes
- JMS applications must use transacted sessions or XA transactions

> ⚠️ Applications that do not use syncpoint processing will not increment `BackoutCount`, and the backout mechanism will not engage. This is the **most common misconfiguration**.

---

## 7. Verification Steps

### 7.1 Confirm Queue Settings

Run the following command and verify the expected output:

```mqsc
DISPLAY QLOCAL(MAIN.QUEUE) BOTHRESH BOQNAME
```

**Expected output:**
```
BOTHRESH(5)
BOQNAME(MAIN.QUEUE.BO)
```

### 7.2 Simulate a Poison Message

1. Force the application to fail processing for a controlled test message.
2. Allow multiple rollback attempts to occur.
3. Verify the message appears in the backout queue (`MAIN.QUEUE.BO`) after `BOTHRESH` is exceeded.
4. Confirm the message is no longer present on the main queue.

---

## 8. Runtime and Performance Considerations

### 8.1 Normal Operation

✅ Under normal conditions (no poison messages), enabling a backout queue has **negligible performance impact** on the queue manager or application.

### 8.2 Failure Scenarios

When a message is backed out, the following minor overhead is incurred:

- `BackoutCount` is incremented in the MQMD on each rollback (minimal CPU cost).
- Once `BOTHRESH` is reached, an additional `PUT` operation and `COMMIT` are required to divert the message to the BOQ.

> ℹ️ The performance cost of enabling a backout queue is far outweighed by the cost of **not** having one. Without a BOQ, a poison message causes an infinite retry loop, leading to CPU spikes, queue depth exhaustion, and application stall.

---

## 9. Monitoring and Alerting

### 9.1 Recommended Monitoring

- Queue depth on `MAIN.QUEUE.BO`
- Queue depth on `SYSTEM.DEAD.LETTER.QUEUE`
- MQ reason codes related to backout or put failures
- Application rollback rates

### 9.2 Alert Thresholds

| Condition | Severity |
|---|---|
| Backout queue depth > 0 | ⚠️ Warning |
| Rapid growth in BOQ | 🔴 Critical |
| DLQ receives messages | 🔴 Critical |

---

## 10. Operational Handling of Backout Messages

### 10.1 Investigation Steps

1. Browse the message in the backout queue to inspect its content and MQMD fields (including `BackoutCount`).
2. Identify the root cause: data format errors, application bugs, or external dependency failures.
3. Determine the appropriate action in consultation with the application team.

### 10.2 Disposition Options

| Action | When to Use |
|---|---|
| **Reprocess** | Root cause resolved; message is valid and can be safely re-queued to `MAIN.QUEUE`. |
| **Correct & Replay** | Message data was malformed or incomplete; fix the data before resubmitting. |
| **Discard (with approval)** | Message is invalid and cannot be corrected; remove with documented approval from application owner. |

---

## 11. Rollback / Disable Procedure

To temporarily disable backout handling on the main queue:

```mqsc
ALTER QLOCAL(MAIN.QUEUE) BOTHRESH(0)
```

> ⚠️ Setting `BOTHRESH(0)` disables new backout routing, but does **not** delete messages already moved to the BOQ. Ensure monitoring is adjusted accordingly. Re-enable BOQ with a non-zero `BOTHRESH` once the issue is resolved.

---

## 12. Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Infinite poison message loop | Configure backout queues with appropriate `BOTHRESH` |
| CPU spikes from retry storms | Set `BOTHRESH` to limit retries before BOQ diversion |
| Message loss | Configure DLQ as safety net at queue manager level |
| Unmonitored BOQ growth | Implement alerting on BOQ depth and DLQ arrivals |

---

## 13. Summary

The IBM MQ backout queue mechanism provides a controlled, low-overhead method to isolate poison messages, prevent processing loops, and protect overall runtime performance. When correctly configured with an appropriate `BOTHRESH`, a dedicated `BOQNAME`, and a queue manager-level DLQ, the mechanism ensures reliable and predictable message processing with minimal operational overhead.

> ✅ **Proper configuration, active monitoring, and operational discipline** are the three pillars of a well-managed backout queue implementation.

---

## 14. Operational Checklist

Use this checklist for implementation validation, readiness reviews, or production go-live checks.

### 14.1 Prerequisites

- [ ] Queue Manager is running
- [ ] Application uses transactional (syncpoint) processing
- [ ] MQ admin privileges available
- [ ] Sufficient disk space for BOQ and DLQ
- [ ] Application team aligned on rollback behavior

### 14.2 Queue Definition

- [ ] Main application queue defined: `DEFINE QLOCAL(MAIN.QUEUE)`
- [ ] Backout queue defined: `DEFINE QLOCAL(MAIN.QUEUE.BO)`
- [ ] Backout queue is dedicated to this application
- [ ] Backout queue has appropriate `MAXDEPTH` and monitoring enabled

### 14.3 Backout Configuration

- [ ] `BOQNAME` and `BOTHRESH` configured on main queue: `ALTER QLOCAL(MAIN.QUEUE) BOQNAME(MAIN.QUEUE.BO) BOTHRESH(5)`
- [ ] `BOTHRESH` value agreed with application team
- [ ] `BOTHRESH` value documented

### 14.4 Dead Letter Queue

- [ ] `SYSTEM.DEAD.LETTER.QUEUE` exists and is accessible
- [ ] DLQ configured on Queue Manager: `ALTER QMGR DEADQ(SYSTEM.DEAD.LETTER.QUEUE)`
- [ ] DLQ is monitored and alerts are configured

### 14.5 Application Behavior

- [ ] Messages are read under syncpoint
- [ ] Application `ROLLBACK`s on processing failure
- [ ] Application `COMMIT`s on success
- [ ] Application does not suppress `MQRC_BACKED_OUT` logic
- [ ] JMS applications use transacted sessions or XA

### 14.6 Validation & Testing

- [ ] Verified queue attributes: `DISPLAY QLOCAL(MAIN.QUEUE) BOTHRESH BOQNAME`
- [ ] Test poison message injected
- [ ] Message rolled back multiple times
- [ ] Message moved to BOQ after `BOTHRESH` exceeded
- [ ] Message removed from main queue

### 14.7 Monitoring & Alerting

- [ ] Alert on backout queue depth > 0
- [ ] Alert on rapid growth of BOQ
- [ ] Alert on messages arriving in DLQ
- [ ] Rollback rate monitored per application
- [ ] BOQ monitored during business hours

### 14.8 Final Readiness Sign-Off

- [ ] Configuration reviewed and approved
- [ ] Monitoring in place and tested
- [ ] Operations team trained
- [ ] Application team informed
- [ ] Solution approved for production

---

## Appendix A — Reference Diagrams

### A1. Backout Queue Flow

See [Section 4 — Design Overview](#4-design-overview) for the full annotated flow diagram.

### A2. Queue Configuration Relationships

```
Queue Manager ──DEADQ──► SYSTEM.DEAD.LETTER.QUEUE
      │
      └── (fallback)
                │
Application ──MQGET SYNCPOINT──► MAIN.QUEUE ──BOQNAME──► MAIN.QUEUE.BO
                                      │
                                   BOTHRESH──► Threshold = 5
```

### A3. Monitoring and Escalation

```
MAIN.QUEUE.BO ──depth monitor──► Warning: BOQ depth > 0 ──┐
              ──growth rate───► Critical: BOQ rapid growth ──► Operations Team ──► Investigate root cause ──► Action
SYSTEM.DEAD.LETTER.QUEUE ──any message──► Critical: DLQ receives message ──┘

Actions:
  ├── Reprocess
  ├── Fix & Replay
  └── Discard with Approval
```

---

## Appendix B — Reference Links

### IBM Official Documentation

| # | Resource | Description |
|---|---|---|
| 1 | [Handling poison messages in IBM MQ classes for JMS](https://www.ibm.com/docs/en/ibm-mq/9.3.x?topic=applications-handling-poison-messages-in-mq-classes-jms) | Primary IBM Docs page covering how JMS detects BackoutCount, compares against BOTHRESH, and moves messages to BOQNAME or the DLQ. Covers both point-to-point and pub/sub. |
| 2 | [Working with Dead-Letter Queues](https://www.ibm.com/docs/en/ibm-mq/9.3.x?topic=objects-working-dead-letter-queues) | IBM MQ 9.3 official reference for DLQ configuration, the runmqdlq handler, and message disposition. |
| 3 | [runmqdlq — Run Dead-Letter Queue Handler](https://www.ibm.com/docs/en/ibm-mq/9.3.x?topic=reference-runmqdlq-run-dead-letter-queue-handler) | Command reference for the built-in DLQ handler utility. |
| 4 | [Dead-Letter Queue Considerations (IBM MQ 9.4)](https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=troubleshooting-dead-letter-queue-considerations) | Troubleshooting guidance for DLQ-related scenarios. |
| 5 | [DLQ vs. Backout Queue (BOQNAME) — IBM Support](https://www.ibm.com/support/pages/what-are-differences-between-mq-dead-letter-queue-dlq-ordeadq-and-backout-queue-boqname) | Concise IBM support article explaining the difference between the two mechanisms and when each applies. |

### IBM Community Blog

| # | Resource | Description |
|---|---|---|
| 6 | [Understanding MQ Backout Queues & Thresholds](https://community.ibm.com/community/user/blogs/stephaniewilkerson1/2021/02/25/understanding-mq-backout-queues-thresholds) | Wilkerson & Smith, IBM Community Blog (Feb 2021). Accessible overview of poison messages and how different MQ clients (native MQI, JMS, XMS) handle them. |

### Practitioner Blog

| # | Resource | Description |
|---|---|---|
| 7 | ["This poisoned message is killing me — what can I do?"](https://colinpaice.blog/2018/10/02/this-poisoned-message-is-killing-me-what-can-i-do/) | Colin Paice (IBM MQ z/OS veteran), Oct 2018. Practical MQI pseudocode for native application backout handling using MQINQ, BOQNAME, and MQPUT1. Useful for application developers. |

---

*IBM MQ Backout Queue Runbook — Distributed Platforms*
