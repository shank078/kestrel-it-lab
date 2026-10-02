# CHG-0001 · Service desk workflow redesign

| | |
|---|---|
| **Date** | 2 Oct 2026 |
| **Requested by** | Me (found during testing) |
| **Type** | Normal change |
| **Affects** | Kestrel IT Service Desk: Incident and Service request work types, both SLAs, the customer-reply automation |
| **Status** | Done and tested |

## Why

The template workflow (To Do → In Progress / Pending → Done) didn't fit a service desk:

- Its transition names didn't match what they did. "In review" went to Pending, "Approved" went back to In Progress.
- There was one "waiting" status for two different situations. Waiting on the **customer** should pause the resolution SLA, because the delay is theirs. Waiting on a **vendor** shouldn't, because chasing the vendor is still IT's job.
- There was no "Resolved but customer can still come back" state before the ticket was final.

## What changed

**Statuses:** removed To Do and Done from this workflow (they're still used by the Task workflow). Added Waiting for support, Waiting for customer, Resolved and Closed.

**Transitions:**

| Transition | From → To | Notes |
|---|---|---|
| Create | → Waiting for support | |
| Start | Waiting for support → In progress | |
| Ask customer | Waiting for support, In progress → Waiting for customer | New |
| Customer replied | Waiting for customer → Waiting for support | New, used by automation |
| Wait on third party | In progress → Pending | Was "In review" |
| Resume work | Pending → In progress | Was "Approved" |
| Resolved | Waiting for support, In progress, Waiting for customer, Pending → Resolved | Kept the existing transitions so their rule (asks for a resolution) stayed |
| Reopen | Resolved → Waiting for support | Was "Reopened". Kept its rule that clears the resolution |
| Close | Resolved → Closed | New, used by auto-close |

**SLAs:**
- First response stops on: public reply, entering Waiting for customer, or resolution set.
- Resolution pauses during Waiting for customer only. Starts on Issue Created **or Resolution: Cleared** (the second one added after testing, see below).

**Automation:**
- Customer reply: if the status is Waiting for customer and the commenter is the reporter, move to Waiting for support.
- New rule, auto-close: weekdays 7:00am, `project = KSD AND status = Resolved AND resolved <= -7d` → Closed.

## Impact on open tickets

When publishing, Jira mapped the existing tickets to the new statuses: KSD-2 and KSD-3 To Do → Waiting for support, KSD-1 Done → Resolved. Their SLA due times didn't change.

## Test plan and results

End-to-end test on a new ticket (KSD-4). Full table in the [F1 README](README.md#testing-end-to-end-on-ksd-4). Every path passed except the reopen, which found a bug.

## Problem found during testing

Reopening a resolved ticket cleared the resolution correctly, but the resolution SLA didn't restart. It only started on "Issue Created". Fixed by adding **Resolution: Cleared** as a second start condition, then re-tested: the reopen started a new 8-hour clock.

## Rollback

Not needed. If it had been: switch Incident and Service request back to the original workflow in the workflow scheme, and put the old SLA conditions and automation back (screenshots of the originals are in `evidence/`).

## Not done in this change

- Rule to react when a customer replies on a Resolved ticket.
- Emailed request moved onto this workflow.
