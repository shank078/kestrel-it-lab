# F3 · Shared folders and permissions

**Started:** 9 Oct 2026
**Finished:** 9 Oct 2026
**Status:** Complete. Built, tested with three users, restore tested, break/fix done and KBs written. Open items are under "Known issues".

## The request

> **KSD-7** · From: Priya Sharma, Finance Manager, Kestrel Freight (fictional)
>
> Our files are all over the place. Finance spreadsheets get emailed around, HR keeps staff records on one person's laptop, and last month someone saved over the pay run file and we had no older copy.
>
> - Each department gets its own drive that only that department can open. Finance and HR especially, as those files are confidential.
> - One company-wide folder everyone can read, for policies and forms. Only managers should be able to change it.
> - The drives should just appear when people log in.
> - If someone deletes or saves over a file, we need to be able to get yesterday's version back.
>
> No budget for new hardware this quarter. Please make changes after 5pm so nobody loses work.

![KSD-7 raised](evidence/01-ksd7-raised-by-priya.png)

In short, Priya wants file shares with group-based permissions, drives mapped by Group Policy, and shadow copies so people can restore older versions themselves.

## What I built

| What | Details |
|------|---------|
| Staff PC | `vm-kestrel-cl01`, Windows 11, in `snet-clients` (10.10.2.4), joined to `kestrel.local`, in Computer_Resources › Workstations |
| Data disk | 32 GiB Standard SSD on `vm-kestrel-dc01`, formatted as `E:` (label `Data`) |
| Shares | `Company`, `Finance`, `HR`, `Operations` under `E:\Shares`, all with access-based enumeration |
| Users | Priya Sharma (`psharma`, Finance Manager) and Emma Wilson (`ewilson`, HR Advisor). James Smith (`jsmith`) updated to Dispatch Officer, Operations |
| Groups | 5 global groups (who people are) and 5 domain local groups (what they can open). See below |
| GPOs | **USR - Department drive maps** (linked to Staff) and **WS - Staff RDP to lab PCs** (linked to Workstations) |
| Shadow copies | On for `E:`, default schedule (7am and 12pm on weekdays), 3000 MB cap |
| Change record | [CHG-0002](change-02-department-file-shares.md) |

### Who can open what

| Folder | Drive | Can change | Can read |
|---|---|---|---|
| Company | K: | Managers (Priya; Megan Doyle once she has an account) | All staff |
| Finance | F: | Finance | Nobody else |
| HR | R: | HR | Nobody else |
| Operations | O: | Operations | Nobody else |

## How I did it

### 1. Talking to Priya first

I replied within 4 minutes, said what would happen next, and asked who should be able to change the Company folder.

![First response](evidence/02-ksd7-first-response-1057.png)

Priya replied that the plan "looks good" and named the managers: herself and Megan Doyle. But I hadn't actually posted the plan yet, only said I would. So I asked her to confirm once more, and then posted the full plan on the ticket so the approval and what she approved are in the same place.

![Priya's reply](evidence/03-ksd7-priya-replies-managers-named.png)
![Approval confirmed](evidence/04-ksd7-approval-confirmed-1201.png)
![Full plan posted](evidence/05-ksd7-full-plan-posted-after-approval.png)

The plan is written up properly in the change record: [CHG-0002](change-02-department-file-shares.md).

### 2. A staff PC to test from

Until now there was only the DC, and James's account had never logged in to a PC. I created a Windows 11 VM in `snet-clients`. No extra NSG work was needed, because the NSG from F0 is attached to both subnets.

Before joining it to the domain I checked DNS. The PC was using Azure's DNS (168.63.129.16), which knows nothing about `kestrel.local`:

![Can't find the domain](evidence/06-client-dns-azure-provided-cant-find-domain.png)

On a normal office network, DHCP option 006 tells PCs which DNS server to use. In Azure the same job is done by the **VNet's DNS setting**, so I changed it from Azure-provided to the DC (10.10.1.4).

![VNet DNS before](evidence/07-vnet-dns-before-azure-provided.png)
![VNet DNS after](evidence/08-vnet-dns-after-custom-dc-10-10-1-4.png)

After that the PC was using 10.10.1.4. My first lookup failed because I typed `kestrel.locals`. The second one worked:

![nslookup](evidence/09-client-nslookup-typo-then-domain-resolves.png)

Then the record a PC actually uses to find a domain controller (the SRV record), and an internet name to make sure that still worked through the DC's forwarder:

![SRV record](evidence/10-client-srv-record-and-internet-names.png)

Joined it to the domain:

![Joined](evidence/11-client-joined-kestrel-local.png)

New computers land in the built-in **Computers** container, which can't have GPOs linked to it. I moved it into Computer_Resources › Workstations, where workstation GPOs will apply.

![In Computers](evidence/12-client-in-default-computers-container.png)
![In Workstations](evidence/13-client-moved-to-workstations-ou.png)

### 3. Users

Two new users with PowerShell. I used splatting (putting the settings in a `@{ }` list first), which is easier to read and check than one long line. The temporary password was typed with `Read-Host -AsSecureString`, so it isn't on screen or in the history.

The first attempt failed: the password didn't meet the domain's complexity rules. The accounts were still created, just **disabled** with no password. That's worth knowing, because a failed `New-ADUser` can leave a half-made account behind.

![Users disabled](evidence/15-new-users-created-disabled-weak-password.png)

Running `New-ADUser` again would have failed with "already exists". The fix was to set a proper password on the existing accounts and enable them:

![Fixed](evidence/16-users-fixed-password-reset-and-enabled.png)

Each account has a lab email alias (`kestrel.jsmith.lab+priya@gmail.com` and `+emma@`). Gmail delivers these to the same inbox, so each person can be a separate Jira customer without me creating more mailboxes. Same idea as linking James in F2.

### 4. Groups (AGDLP)

The rule I followed: **people go into groups, groups go onto folders. Never people straight onto folders.**

- **Global groups (GG-)** describe *who someone is*: GG-Finance, GG-HR, GG-Operations, GG-Managers, GG-AllStaff.
- **Domain local groups (DL-)** are *keys to a folder*: DL-Finance-Modify, DL-Company-Read, and so on. Only these appear in the folder permissions.
- Each team is put onto the keys it needs.

When someone joins Finance, I add them to GG-Finance and nothing else. They get every Finance key automatically. Nobody has to touch the folder permissions.

![Groups](evidence/17-groups-created-global-and-domain-local.png)
![Nesting](evidence/18-group-nesting-who-holds-each-key.png)

The check at the end lists who really ends up holding each key (`Get-ADGroupMember -Recursive`). This is how you'd check access at scale too. With 1,500 users you don't test logins one by one; you report on the groups.

### 5. Data disk

Kept the files off the C: drive. Added a 32 GiB Standard SSD data disk to the DC, with host caching set to None.

![Data disk](evidence/20-dc-data-disk-attached-32gib.png)

Initialised it as GPT, created one partition and formatted it as `E:` with the label `Data`:

![E: formatted](evidence/21-e-drive-initialised-formatted-data.png)

I'd wondered whether each department should get its own partition. It doesn't need one. One volume with folders is normal, and a mapped drive like F: is just a shortcut to a share, not a real disk.

### 6. Folder permissions (NTFS)

Before changing anything I looked at what a new folder inherits. Every member of BUILTIN\Users could read Finance and create files in it, which is the opposite of what Priya asked for:

![Default permissions](evidence/22-finance-folder-default-permissions-before.png)

`icacls` with `/inheritance:r` removed the inherited entries, and `/grant:r` set exactly what each folder should have: SYSTEM and Administrators full control, plus the department's DL group. Company gets both the Read and the Modify group. `(OI)(CI)` means the permission flows down to every subfolder and file.

![NTFS set](evidence/23-ntfs-permissions-set-by-group.png)

### 7. Shares

Shared all four with **access-based enumeration**, which hides folders from people who can't open them.

![Shares](evidence/24-smb-shares-access-based-enumeration.png)

Share permissions are wide on purpose: Administrators Full, Authenticated Users Change. When someone opens a file over the network, Windows checks the share permission *and* the NTFS permission and uses whichever is stricter. So the NTFS permissions from step 6 do the real work, and there's only one place to manage access.

![Share permissions](evidence/25-share-permissions.png)

### 8. Letting staff log in to the PC

Domain users can't RDP to a PC unless they're in its local **Remote Desktop Users** group. Normally staff sit at the PC, so this isn't needed. Here the "PC" is an Azure VM, so I added GG-AllStaff to that group with a computer GPO linked to Workstations. It's a lab-only setting.

![RDP GPO](evidence/26-gpo-staff-rdp-to-lab-pcs.png)
![On the PC](evidence/27-client-rdp-group-has-gg-allstaff.png)
![Computer GPO applied](evidence/28-client-computer-gpo-applied.png)

### 9. Mapping the drives

A user GPO, **USR - Department drive maps**, linked to the Staff OU. It uses Group Policy Preferences, with one drive map per share and **item-level targeting**: the F: drive only maps for members of DL-Finance-Modify, and so on.

![Drive maps](evidence/29-drive-maps-gpo-four-drives-targeted.png)

This didn't work first time. See "Things that went wrong".

### 10. Shadow copies

Turned on for `E:` with the default schedule and a 3000 MB cap, so old versions can't fill the disk. The first snapshot was taken straight away (empty folders at that point).

![Shadow copies on](evidence/30-shadow-copies-enabled-first-snapshot.png)

## Testing

Each person logged in to the PC as themselves. I checked their drives, then tried things they should and shouldn't be able to do.

**Priya (Finance, manager)**

![Drives mapped](evidence/34-priya-drives-mapped-after-sid-fix.png)
![This PC](evidence/35-priya-this-pc-f-and-k.png)
![Writes and denials](evidence/36-priya-writes-own-areas-hr-ops-denied.png)

**Restoring an overwritten file.** Priya created the pay run file, a snapshot was taken at 2:14 PM, then she saved over it. She restored it from **Properties → Previous Versions**, and the original text came back.

![Snapshot](evidence/37-shadow-copy-taken-with-pay-run-file.png)
![Previous Versions](evidence/38-previous-versions-tab.png)
![Overwritten then restored](evidence/39-pay-run-overwritten-then-restored.png)

**James (Operations)**

![James](evidence/40-james-k-read-only-o-works.png)

**Emma (HR)**

![Emma](evidence/41-emma-hr-works-finance-ops-denied.png)

| # | Test | Result |
|---|------|--------|
| 1 | Priya logs in and gets K: and F: | ✅ Pass (34, 35), after the targeting fix. Failed the first time (31) |
| 2 | Priya can edit Company (K:) | ✅ Pass (36) |
| 3 | Priya can't open HR or Operations | ✅ Pass, access denied (36) |
| 4 | James gets K: and O: | ✅ Pass (40) |
| 5 | James can't save in K: | ✅ Pass, access denied (40) |
| 6 | James can't open `\\vm-kestrel-dc01\Finance` | ✅ Pass on screen, but my screenshot was taken before I pressed Enter, so the "denied" message isn't in evidence 40. Emma's identical test is (41) |
| 7 | Emma gets K: (read only) and R: | ✅ Pass (41) |
| 8 | Overwritten file restored with Previous Versions | ✅ Pass (37–39) |
| 9 | PC finds the DC through DNS and joins the domain | ✅ Pass (09–11) |
| 10 | Access-based enumeration hides folders a user can't open | Not tested. It's switched on (24), but I didn't browse `\\vm-kestrel-dc01` as a user to see it |

Priya confirmed on KSD-7 that she could see Finance and Company and had restored an old version herself. Resolved at 2:48 PM, both SLAs met.

![Priya confirms](evidence/42-ksd7-priya-confirms-working.png)
![KSD-7 resolved](evidence/43-ksd7-resolved-both-slas-met.png)

## Break/fix 🔧

**INC-0005: new starter has no network drives.** I created a new Finance user, Daniel Lee, the way a rushed admin might: right groups, but no `-Path`, so the account landed in the default **Users** container instead of Staff › Canberra. He could log in, but had no drives. Priya raised KSD-8. I compared Daniel with Priya, used gpresult to show the drive map GPO wasn't applying at all, proved his permissions were fine by opening the share directly, then moved the account into the right OU. Fixed and verified in 16 minutes from the ticket being raised.

Full write-up: [incident-05-new-starter-no-network-drives.md](incident-05-new-starter-no-network-drives.md)

## KB 📘

- [kb-user-has-no-network-drives.md](kb-user-has-no-network-drives.md): what to check, in order, when someone's mapped drives are missing.
- [kb-restore-previous-version-of-a-file.md](kb-restore-previous-version-of-a-file.md): getting back a file someone deleted or saved over.

## Things that went wrong 😅

**1. Drive maps applied but no drives appeared.** Priya's first login showed the GPO in "Applied Group Policy Objects" and all the right DL groups, but `net use` was empty.

![GPO applied, no drives](evidence/31-priya-gpo-applied-but-no-drives.png)
![Groups present](evidence/32-priya-groups-include-dl-keys.png)

So the GPO was reaching her, and she had the groups. That pointed at the drive map items themselves. Two things were wrong:

- F: and K: had been saved **without** item-level targeting. Fixed on all four items.
- In the targeting, I'd **typed** the group names instead of picking them with the **…** button. Typing only saves the name. Picking also saves the group's **SID**, which is what Windows actually checks. With an empty SID, the "is this user in the group?" check never matched. The GPO's settings file (`Drives.xml` in SYSVOL) showed `sid=""` on each item.

I re-picked each group with **…** and Check Names. `Drives.xml` now has a SID on every item, and the drives mapped straight away.

![SIDs filled in](evidence/33-drives-xml-targeting-sids-filled-in.png)

My first suspect was Windows' fast logon (user GPOs sometimes need a second login to apply). That was wrong. gpresult already showed the GPO had applied. I also didn't screenshot the `sid=""` version, only the fixed one.

**2. I changed the VNet's DNS without a change record.** It looked like a setup step for the new PC, but it changed DNS for every VM in the network. It's now recorded in CHG-0002 as done beforehand. It also means the DC is the only DNS server: if the DC is off, no VM can look up any name, even on the internet.

**3. RDP to the new PC failed.** My ISP had given me a new home IP, so the NSG rule that only allows my IP no longer matched. The first guess was a problem on the Mac. Updating the rule to the new IP fixed it. It's the same symptom as INC-0003 in F0 (the NSG rule not matching my IP), except this time my IP changed, not the rule. I didn't recognise it straight away.

![NSG rule updated](evidence/14-nsg-rdp-rule-updated-after-isp-ip-change.png)

**4. I ran the disk check on the wrong server.** I looked for the new data disk on the client PC instead of the DC. Only one disk showed up, so it looked like the disk hadn't attached. The window title and prompt looked almost the same. I now run `hostname` on its own first and read the answer before anything else.

![Wrong machine](evidence/19-disk-check-run-on-client-by-mistake.png)

**5. The weak temporary password** left two disabled, half-made accounts (step 3 above).

**6. James had to change his password at first login.** That was the "must change at next logon" flag left over from F2. It closes one of F2's known issues.

**7. Priya approved before she'd seen the plan** (step 1 above). In real life I'd post the plan first and ask for approval on that.

**8. The change window.** Priya asked for changes after 5pm, and CHG-0002 says so. I did the work during the day because Kestrel's users are made up and nobody could lose work. At a real company I'd stick to the window.

**9. Small stuff:** `nslookup kestrel.locals` (typo), and twice I waited on a command I hadn't pressed Enter on.

## Known issues / still to do

- **File shares on the DC.** Microsoft recommends a DC only does DC work. Accepted because there's no budget for another server. Moving the shares to their own server is a later lab.
- **The DC is the only DNS server** for every VM. If it's off or broken, nothing resolves. A second DC would fix this.
- **Shadow copies aren't a backup.** They live on the same disk, so if the disk is lost, so are the old versions. Real backups come in a later lab.
- **Shadow copies only run while the DC is on.** The schedule is 7am and 12pm on weekdays, but the DC is usually shut down then to save money, so in this lab I take snapshots by hand.
- **Files saved on the PC itself** (Desktop, Documents) aren't on the server and aren't covered by any of this. Folder redirection is planned for the Group Policy lab.
- **Megan Doyle doesn't have an account yet**, so she isn't in GG-Managers and can't edit the Company folder. Add her when her account is created.
- **New user accounts land in the default Users container** if no path is given (INC-0005). `redirusr` can change that default to a "new starters" OU. Planned for the Group Policy lab.
- **Access-based enumeration hasn't been checked** from a user's point of view.
- **The client PC shows US date format and time zone.** Fix: `Set-TimeZone -Id "AUS Eastern Standard Time"` and the region set to Australia.
- **The staff RDP GPO is lab-only.** Remove it if the PCs are ever real desks.

## Cost

- Data disk: about A$3–4 a month (Azure's estimate).
- Client VM: only runs during lab sessions and is shut down after.
- Actual cost to be added from Cost Management.

## Handover

For whoever looks after the file shares next:

- **Where:** `\\vm-kestrel-dc01\Company`, `\Finance`, `\HR`, `\Operations`, stored on `E:\Shares` on the DC.
- **Giving someone access:** add them to their team's **GG-** group (GG-Finance, GG-HR, GG-Operations, plus GG-Managers for managers and GG-AllStaff for everyone). Don't add people to DL- groups or straight onto folders.
- **Their account must be in Staff › Canberra (or Sydney)**, otherwise the drive map GPO won't reach them. See INC-0005.
- **Drives:** set by the GPO **USR - Department drive maps**. If you add a share, add a drive map item, and pick the targeting group with the **…** button, never by typing.
- **Changes show up at the next logon**, because group membership is read when someone logs in.
- **Old versions:** right-click the file → Properties → Previous Versions. Steps in the [KB](kb-restore-previous-version-of-a-file.md).
- **Test PC:** `vm-kestrel-cl01`, 10.10.2.4. Shut down between sessions, client first, then the DC.
- **Test users:** `psharma` (Finance, manager), `jsmith` (Operations), `ewilson` (HR), `dlee` (Finance, from INC-0005).
- **Open items:** see "Known issues" above.

## Retro

**What went well**
- Checking before changing: DNS before the domain join, and the default permissions before setting new ones. The second one showed every user could read Finance by default.
- Testing with three real accounts, including the things that **should** fail. Most of the value was in the access-denied results.
- Comparing a broken user with a working one in the break/fix. It found the cause in two commands.

**What was hard**
- **Group scopes.** Global vs domain local didn't make sense until I thought of it as "who you are" vs "which key you hold".
- **The drive map targeting.** Everything looked right in the GPO editor. The answer was in `Drives.xml`, which is what Windows actually reads.

**What I'd do differently**
- Run `hostname` on its own before every command that changes something.
- Use a strong temporary password from the start, and check `Enabled` straight after creating an account.
- Always pick groups with the browse button in GPOs.
- Post the plan before asking for approval, and stick to the agreed change window.
- Screenshot the broken state before fixing it.

**What I learned**
- AGDLP: people into teams, teams onto keys, keys onto folders.
- The stricter of share and NTFS permissions wins, so keep the share wide and control access with NTFS.
- User GPOs follow the user's OU and computer GPOs follow the PC's OU. Containers like Users and Computers can't have GPOs.
- If gpresult shows "Applied Group Policy Objects: N/A", check where the account sits before anything else.
- Shadow copies make restores self-service, but they aren't a backup.
