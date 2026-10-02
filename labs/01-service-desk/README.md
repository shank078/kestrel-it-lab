# F1 · Service desk (Jira Service Management)

**Started:** 2 Oct 2026
**Status:** In progress. The service desk is built and tested end to end. Still to do: a planned break/fix, a KB article, and the follow-ups listed at the bottom.

## The request

> **REQ-0003** · From: Megan Doyle, Operations Manager, Kestrel Freight (fictional)
>
> IT problems come in by email, phone and people stopping you in the corridor, and things get forgotten. I want one place where staff log problems and requests, where we can see what's open, who's on it and how long it's been waiting. Free if possible.

So Kestrel needs a proper service desk: a portal for staff, queues for IT, and time targets so nothing sits forgotten.

## Why Jira and not ServiceNow

I wanted ServiceNow, but the free developer instances had a waitlist of a few weeks. Jira Service Management has a free plan (up to 3 agents) that a small company like Kestrel could actually use, and the ideas are the same: request types, queues, SLAs, workflows, automation.

## What I built

| What | Details |
|------|---------|
| Site and space | `kestrel-it-lab.atlassian.net`, space **Kestrel IT Service Desk**, key **KSD** (tickets are KSD-1, KSD-2…) |
| Work types | **Incident** (something broke) and **Service request** (asking for something), kept separate |
| Portal | Two groups staff understand: *Something's not working* and *Request something*. 5 request types |
| Forms | Each request type asks the question an agent actually needs answered |
| Workflow | Waiting for support → In progress → Waiting for customer / Pending → Resolved → Closed |
| SLAs | First response and resolution, on Canberra business hours with a public holiday |
| Queues | All open, Assigned to me, Unassigned, Incidents, Service requests, About to breach |
| Automation | Customer reply moves the ticket back to IT. Resolved tickets close themselves after 7 days |
| Access | Portal restricted to customers I add. Test customer in organisation *Kestrel Freight - Canberra* |

## How I did it

### 1. The space

Renamed the template's "IT Service Project" and changed the key to **KSD** before any tickets existed. Changing a key later breaks links and re-indexes everything.

![Space name and key](evidence/01-space-name-and-key-ksd.png)

### 2. Incidents and service requests

The template only had one combined type ("Submit a request or incident"). A real desk keeps them apart: an incident is "it used to work and now it doesn't", a service request is "I need something new". They get different targets and you want to report on them separately.

![Incident work type](evidence/02-work-type-incident.png)

New work types aren't usable until they're added to the space's **work type scheme**. Jira first added them to the default scheme, which no space uses, so I had to add them to Kestrel's scheme myself.

![Work type scheme](evidence/03-work-type-scheme-incident-service-request.png)

They also landed on Jira's plain built-in workflow, so I moved them onto the service desk one.

![Workflow scheme](evidence/04-workflow-scheme-assign-to-esm-workflow.png)

### 3. Request types and the portal

Staff never see the words "incident" or "service request". They pick from plain options, and each one maps to the right work type behind the scenes.

![Request types](evidence/05-request-types-and-portal-groups.png)
![Portal home](evidence/06-portal-home-two-groups.png)

Each form asks something specific, so an agent can start straight away instead of replying "can you give me more detail?". Report a problem asks *What's happening?* (required) and allows a screenshot. I also replaced the template's welcome text, which still said "IT Service Project".

![Report a problem form](evidence/07-report-a-problem-form-fields.png)
![Portal name and intro](evidence/08-portal-name-and-intro-text.png)

### 4. SLAs on business hours

Two clocks: **time to first response** (someone has picked it up) and **time to resolution** (it's actually fixed).

| | First response | Resolution |
|---|---|---|
| Incident | 1 business hour | 8 business hours |
| Service request | 4 business hours | 24 business hours |

![First response SLA (first version)](evidence/09-sla-first-response-v1.png)

The only calendar Jira offered was 24/7, which would breach a ticket logged at 4:55pm on a Friday by 5:55pm with nobody in the office. So I made **Kestrel Canberra business hours** (Mon–Fri, 8:30–5:00, Sydney time zone).

Then I noticed **Monday 5 October 2026 is Labour Day** in the ACT. Before adding the holiday, KSD-1 was due Monday 10:41am. After adding it, every due time moved by one business day. Jira recalculated the open tickets on its own.

![Before the holiday](evidence/13-queue-sla-due-before-holiday.png)
![Calendar with Labour Day](evidence/14-calendar-business-hours-labour-day.png)
![After the holiday](evidence/15-queue-sla-due-after-holiday.png)

### 5. Queues and access

Queues are saved filters, like inbox views. *About to breach* shows anything with under an hour left on its resolution clock.

![Incidents queue](evidence/10-queue-incidents-jql.png)

By default anyone with the link could raise a request. I set it to **Restricted**, so only customers I've added can log tickets.

![Customer permissions](evidence/11-customer-permissions-restricted.png)

### 6. First tickets and a gap I found

I raised three tickets as a test customer (name blacked out in screenshots).

![KSD-1 customer view](evidence/12-ksd1-customer-view.png)

Resolving KSD-1 failed in a quiet way: the **Resolution** dropdown was empty. The site had no resolution values at all. Without one, a ticket shows "Done" but still counts as unresolved, so it stays in the queues and its SLA keeps running. I added Done, Won't do, Duplicate and Cannot reproduce.

![Resolutions added](evidence/17-resolutions-added.png)
![KSD-1 resolved, both SLAs met](evidence/18-ksd1-resolved-both-slas-met.png)

I also built my first automation (customer replies → move the ticket back to IT). KSD-1's history shows **Automation for Jira** moving it, not me.

![Automation v1](evidence/16-automation-v1-customer-reply.png)
![KSD-1 history](evidence/19-ksd1-history-automation-transition.png)

### 7. Redesigning the workflow

The template's workflow didn't match how a desk actually works. Its buttons were confusing ("In review" led to Pending, "Approved" led back to In Progress) and there was no way to say *"I'm waiting on the customer"* separately from *"I'm waiting on a vendor"*. That difference matters for SLAs.

Full change record: [change-01-workflow-redesign.md](change-01-workflow-redesign.md)

![Before](evidence/20-workflow-before-with-labels.png)
![After](evidence/21-workflow-after.png)

| Status | Meaning | Resolution clock |
|---|---|---|
| Waiting for support | New, or the customer has replied | Running |
| In progress | Someone's working on it | Running |
| Waiting for customer | We asked the customer something | **Paused** (the delay is theirs) |
| Pending | Waiting on a vendor, parts or another team | **Running** (still IT's job to chase) |
| Resolved | Fix delivered, customer can still reply | Stopped |
| Closed | Final | Stopped |

Open tickets were moved to the new statuses when I published it. Then I updated the SLAs and the automation to use the new statuses.

![Queue after migration](evidence/22-queue-after-status-migration.png)
![First response SLA v2](evidence/23-sla-first-response-v2.png)
![Automation v2](evidence/24-automation-v2-waiting-for-support.png)

## Testing (end to end on KSD-4)

One fresh ticket, "Printer on level 1 not printing", taken through every path.

| Step | What I did | Expected | Result |
|---|---|---|---|
| 1 | Customer raised it | Lands in Waiting for support | ✅ |
| 2 | Agent: *Ask customer* | First response met, resolution paused | ✅ (met 1:09pm, ⏸) |
| 3 | Customer replied | Automation moves it to Waiting for support, clock resumes | ✅ (deadline moved by the minutes it was paused) |
| 4 | Agent: *Wait on third party* | Pending does **not** pause the clock | ✅ (deadline unchanged) |
| 5 | Agent: resolved with Done | Both SLAs met | ✅ |
| 6 | Customer: "still streaky" on the resolved ticket | Should get noticed | ❌ Nothing happened (see follow-ups) |
| 7 | Agent: *Reopen* | Resolution cleared, new resolution clock | ⚠️ Resolution cleared, but **no new clock**. Fixed, then ✅ |

![New ticket](evidence/25-ksd4-new-ticket-waiting-for-support.png)
![Waiting for customer, paused](evidence/26-ksd4-waiting-for-customer-sla-paused.png)
![Customer reply, auto transition](evidence/27-ksd4-customer-reply-auto-transition.png)
![Clock resumed](evidence/28-ksd4-sla-resumed.png)
![Pending keeps running](evidence/29-ksd4-pending-sla-keeps-running.png)
![Resolved, met](evidence/30-ksd4-resolved-slas-met.png)

**The reopen bug:** the resolution SLA only started on "Issue Created". A reopened ticket wasn't just created, so it ran with no SLA at all. Adding **Resolution: Cleared** as a second start condition fixed it. The next reopen got a fresh 8-hour clock.

![No new clock](evidence/31-ksd4-reopened-no-new-clock.png)
![The fix](evidence/32-sla-resolution-start-on-resolution-cleared.png)
![New clock](evidence/33-ksd4-reopen-starts-new-clock.png)

KSD-3 (folder access) checked the service request targets (4h/24h). It also showed that a customer's own comment doesn't stop the first-response clock; only an agent's public reply does.

![KSD-3](evidence/34-ksd3-service-request-targets.png)

## Things that went wrong 😅

- **I kept mixing up "Waiting for support" and "Waiting for customer"** when building transitions and the automation. Three times. Reading the From/To fields before saving every time caught it.
- **A reply I posted went on the wrong ticket** (KSD-3 instead of KSD-1). Left an internal note on KSD-3 explaining it, the way you would at work.
- **I assumed the automation was broken** when the SLA panel still showed ⏸. It wasn't: the page was stale. A full page reload showed the right state. Lesson: reload before deciding something's broken.
- **I deleted the auto-close rule by accident** halfway through and rebuilt it.

## Known issues / follow-ups

- **Customer replies on Resolved tickets go unnoticed** (KSD-4, step 6). Next rule to build: reopen when the customer replies within 7 days of resolution.
- **Labour Day is saved as "every 5 October"**, but Labour Day is the first Monday of October, so the date moves. It needs updating each year, or ACT holidays imported from an ICS file.
- Only one public holiday is in the calendar so far.
- **Emailed requests** still use the old simple workflow. Email isn't set up yet.
- **Priority** isn't used by the SLAs, and I didn't set it consistently (KSD-4 affected the whole floor and should have been High).
- Folder access approval was "confirmed by phone" in a note, not with Jira's Approvals feature.
- **KSD-2** (locked out) is still open on purpose. I'll work it in a DC session using the [F2 runbook](../02-active-directory/runbook.md).
- **Auto-close rule:** first real run expected Monday 12 October (KSD-1). Not verified yet.
- Evidence gaps: no screenshot of the auto-close rule, the full resolution SLA settings, or the status-mapping screen when publishing the workflow.

## Cost

A$0. Jira Service Management free plan.
