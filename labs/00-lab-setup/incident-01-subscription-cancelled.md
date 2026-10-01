# INC-0001 · Azure subscription cancelled by mistake

| | |
|---|---|
| **Date** | 1 Oct 2026 |
| **Raised by** | Me (self-caused) |
| **Affected** | Azure subscription 1 (the whole Kestrel lab) |
| **Status** | Resolved |

## What happened

I was on the Subscriptions page trying to get to Budgets. I clicked **+ Add**, which on that page means "create a new subscription", not "add a budget". While backing out of that screen I ended up cancelling the existing subscription instead.

## Symptoms

The subscription's Overview page showed **Status: Cancellation in progress**.

## Impact

Nothing was running yet, so there was no actual outage and no data lost. In a real company this would be a big deal. Cancelling a subscription stops billing and disables everything in it: VMs get deallocated, storage goes read-only.

## Investigation

- Checked the subscription status: "Cancellation in progress", so the cancel had definitely gone through.
- Read Microsoft's docs on cancelling. A cancelled subscription isn't deleted straight away. Data is kept and it can be reactivated within 90 days. Pay-as-you-go can be reactivated from the portal, other types (like free trial) may need a support request.
- Decided **not** to create a new subscription as a workaround. My credit, the spending limit and the free-tier hours belong to the original subscription, and in a real job you'd never work around a mistake like this by starting a new subscription anyway.

## Fix

Waited for the cancellation to finish, then reactivated the subscription. Status went back to **Active**. Checked All resources afterwards: nothing missing (it was empty before too).

## Cause

Two things:

1. **Cancel subscription** sits on the same toolbar as Rename, right on the Overview page. Easy to hit.
2. I didn't check which page I was on before clicking.

## Prevention

- Read the page title before clicking anything on a subscription page.
- I'd assumed a delete lock would stop this. It doesn't. Locks protect resources and resource groups, but cancelling is a **billing** action, so a lock won't block it. The real fix in a company is permissions: day-to-day admins get Contributor, and only billing admins can cancel a subscription.

## What I learned

Undo the mistake properly instead of working around it. And locks and permissions protect different things, which I didn't know before today.

Reference: [Cancel and delete your Azure subscription (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/cancel-azure-subscription)
