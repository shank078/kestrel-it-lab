# CHG-0002 · Department file shares

| | |
|---|---|
| **Date raised** | 9 Oct 2026 |
| **Requested by** | Priya Sharma, Finance Manager (fictional), ticket **KSD-7** |
| **Type** | Normal change |
| **Change window** | After 5pm, so nobody is working in the files (see "What actually happened": not kept) |
| **Approval** | Priya Sharma (business owner of the request), then me as the implementer |
| **Status** | Completed 9 Oct 2026, with the deviations listed below |

## Why

Files are scattered: Finance spreadsheets get emailed around, HR records sit on one laptop, and an overwritten pay run file couldn't be recovered. Priya asked for department drives that only that department can open, a company-wide folder everyone can read, drives that appear on their own at login, and a way to get yesterday's version of a file back.

## What changes

**On the domain controller (`vm-kestrel-dc01`)**
- Add a 32 GiB Standard SSD data disk, formatted as `E:` (label `Data`).
- Create four folders on `E:` and share them: `Company`, `Finance`, `HR`, `Operations`.
- Set NTFS permissions using groups (AGDLP), not individual users.
- Turn on shadow copies for `E:`, twice a day.

**In Active Directory**
- Users: `psharma` (Priya Sharma, Finance Manager) and `ewilson` (Emma Wilson, HR Advisor) in Staff › Canberra. `jsmith` already exists.
- Global groups (who people are): `GG-Finance`, `GG-HR`, `GG-Operations`, `GG-Managers`, `GG-AllStaff`.
- Domain local groups (what they can access): `DL-Finance-Modify`, `DL-HR-Modify`, `DL-Ops-Modify`, `DL-Company-Read`, `DL-Company-Modify`.

**Group Policy**
- A user GPO linked to Staff that maps drives by group (Group Policy Preferences): Company `K:`, Finance `F:`, HR `R:`, Operations `O:`.
- A computer GPO linked to Computer_Resources › Workstations that lets staff log on to PCs over RDP (needed in this lab because the "PC" is an Azure VM).

## Access design

| Folder | Drive | Modify | Read |
|---|---|---|---|
| Company | K: | Managers | All staff |
| Finance | F: | Finance | (nobody else) |
| HR | R: | HR | (nobody else) |
| Operations | O: | Operations | (nobody else) |

## Already done before this change was written

- Created `vm-kestrel-cl01` (stand-in staff PC) in `snet-clients` and joined it to `kestrel.local`, moved to Computer_Resources › Workstations.
- **Changed the VNet's DNS servers from Azure-provided to `10.10.1.4` (the DC).** This affected every VM in the network, not just the new PC, and I did it during preparation without a change record. It should have been one. It worked, but the DC is now the only DNS server, so if the DC is off, no VM can resolve any name.

## Impact

- Staff: no existing shares are being replaced, so nobody loses access to anything. New drives appear at their next login.
- Servers: the DC gets a second role (file server). See risks.
- Cost: the data disk, about A$3–4 a month (to confirm on the Azure pricing screen before creating it).

## Risks

| Risk | What I'm doing about it |
|---|---|
| File shares on a domain controller (Microsoft recommends a DC only does DC work) | Accepted because there's no budget for another server. Recorded as a known issue, with a later lab planned to move the shares to their own server |
| Wrong permissions expose Finance or HR files | Test with three real users before telling Priya it's done, including the "access denied" cases |
| A drive-map GPO maps the wrong drives for everyone | Test on the one test PC first |
| Shadow copies fill the data disk | Shadow copies are capped at a set size of the volume |

## Test plan

| # | Test | Expected |
|---|---|---|
| 1 | Priya logs in to the PC | Gets K: and F: |
| 2 | Priya edits a file in K: | Allowed |
| 3 | Priya opens HR or Operations | Access denied |
| 4 | James logs in | Gets K: (read only) and O: |
| 5 | James tries to save in K: | Denied |
| 6 | James opens `\\vm-kestrel-dc01\Finance` directly | Access denied |
| 7 | Emma logs in | Gets K: (read only) and R: |
| 8 | A file in F: is changed, then restored with **Previous Versions** | Older copy comes back |

## Rollback

1. Unlink the drive-map GPO (drives stop appearing at next login).
2. Remove the shares (folders and files stay on `E:`).
3. If needed, detach and delete the data disk.
4. Users and groups can stay; they do nothing without the shares.

---

## What actually happened

*Added after the change, 9 Oct 2026.*

### Timeline

| Time | Step |
|---|---|
| 10:53 | KSD-7 raised by Priya |
| 10:57 | First response, with a question about Company managers |
| ~11:25–11:45 | Preparation: NSG rule updated for my new home IP, VNet DNS changed, PC joined and moved to Workstations |
| ~11:53 | This change record written |
| 11:58–12:03 | Priya approved, confirmed, and the full plan was posted on the ticket |
| 12:11–12:17 | Users created and fixed, groups created and nested |
| 13:09–13:24 | Data disk attached, `E:` formatted, NTFS permissions set, shares created |
| 13:29–13:36 | RDP GPO built and checked on the PC |
| 13:48–13:52 | Drive map GPO built, shadow copies turned on |
| 13:58 | Test 1 **failed**: GPO applied, but no drives |
| ~14:08 | Drive map targeting fixed |
| 14:11–14:46 | Tests 1–8 run |
| 14:48 | Priya confirmed it works. KSD-7 resolved, both SLAs met |

### Deviations from the plan

- **The change window wasn't kept.** The work was done during the day, not after 5pm. Kestrel's users are made up, so nobody could lose work, but at a real company I'd have waited.
- **Approval came before the plan was on the ticket.** Priya said "the plan looks good" before I'd posted it. I asked her to confirm again and then posted the full plan on KSD-7, so the approval and what she approved are recorded together. Next time: plan first, then approval.
- **The users were first created disabled.** The temporary password didn't meet the complexity rules. I reset the passwords and enabled the accounts. No impact.
- **The drive maps failed at first.** Two items had no targeting, and all four targeting groups had been typed instead of picked, so they had no SID. Fixed by re-picking each group with the **…** button. Details in the [F3 README](README.md).
- **Shadow copy cap:** set to 3000 MB, using the default schedule (7am and 12pm on weekdays). These snapshots only happen while the DC is running.
- **Data disk cost:** the pricing screen wasn't screenshotted, so the A$3–4 figure is still an estimate.

### Test results

7 of 8 passed with screenshot evidence. Test 6 passed on screen, but the screenshot was taken before the command ran. Full table in the [F3 README](README.md#testing).

### Rollback

Not needed.
