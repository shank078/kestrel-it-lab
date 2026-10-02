# KB · Customer replied but the ticket didn't move

For agents on the Kestrel IT Service Desk. Tested in INC-0004.

**What normally happens:** you set a ticket to *Waiting for customer*. When the customer replies, the rule *Customer reply moves ticket to Waiting for support* moves it back to you, and the resolution clock starts again.

## 1. Reload the page

Press ⌘R (Mac) or F5. The ticket view doesn't always update by itself, and a stale page can show the old status.

## 2. Was it really the customer who replied?

Open the ticket → **Activity → Comments**.

| What you see | Meaning |
|---|---|
| The reply came from the **reporter** | Should have moved. Go to step 3 |
| The reply came from someone else (an agent, or a participant who isn't the reporter) | Working as designed. The rule only reacts to the reporter |
| The reply is on a **Resolved** ticket | Known gap. The rule only watches Waiting for customer. Reopen it by hand |

## 3. Did the rule run?

On the ticket, open the **Automation** panel (right side, near the bottom).

- **No run at the time of the reply:** the rule may be disabled. Check the **Enabled** toggle on the rule.
- **A run with a green tick:** it ran, but a green tick doesn't mean it did anything. Go to step 4.

## 4. Where did it stop?

Click the run to open its **Audit log** entry, and expand it with **>**.

| Last step shown | Meaning |
|---|---|
| *Work item fields condition*, "did not match the condition" | The status check said no. Open the rule and check box 2 says **Waiting for customer** |
| *Compare two values* | The reporter check said no. See step 2 |
| *Transition the work item*, with an error | The workflow didn't allow the move. Check the **Customer replied** transition still exists |

## 5. What changed?

In the rule's **Audit log**, look for **Config change** entries. A change just before the problem started is the most likely cause. Find out who made it and why before you change it back.

## Fixing it

1. Fix the rule, click **Save**, and wait for Save to go grey.
2. **Fixing the rule won't move tickets that were already missed.** Find them:
   ```
   project = KSD AND status = "Waiting for customer" ORDER BY updated DESC
   ```
   Open each one. If the last comment is from the customer, use **Customer replied** to move it back, and add an internal note saying why.
3. Test it: on a test ticket, Ask customer, reply as the customer, reload, and check History shows **Automation for Jira** changed the status.
