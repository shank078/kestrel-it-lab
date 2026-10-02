# KB · Can't RDP to the Kestrel DC (Azure)

For whoever looks after `vm-kestrel-dc01`. Tested in INC-0003.

## 1. Read the error

| Error | Where the problem is | Go to |
|---|---|---|
| Comes back instantly: wrong password, account locked (0xd07), must change password (0x1207) | **The account.** The server answered | [F2 runbook](../02-active-directory/runbook.md) §2, §3, §5 |
| Hangs, then "couldn't connect" (0x204) | **The network.** Nothing answered | Steps 2–5 below |

## 2. Is the VM running?

Portal → **vm-kestrel-dc01** → Overview → **Status**. It has to say **Running**. Stopped (deallocated) VMs don't answer.

## 3. Am I connecting to the right address?

The public IP on the Overview page has to match the address in your RDP connection.

## 4. What's my IP right now?

Search "what is my IP" and compare it with the **Source** of the `Allow-RDP-MyIP` rule in **nsg-kestrel-ause → Inbound security rules**.

## 5. Ask Azure which rule decides

**Network Watcher → IP flow verify**: VM `vm-kestrel-dc01`, TCP, Inbound, local `10.10.1.4:3389`, remote = your IP, remote port 50000. It says allowed or denied, and names the rule.

## Fixing it

- **Rule has the wrong source:** set it to your current home IP. Never use a VPN or mobile IP; they're shared with other people.
- **After any NSG change:** test with a **new** RDP connection. Connections that are already open can keep working and hide the problem.
- **Still nothing?** Use **Serial console** (vm-kestrel-dc01 → Help → Serial console). It works even when the network is broken. Run `ipconfig`: a 169.254.x.x address means Windows has no valid IP (see F2, "Things that went wrong").
