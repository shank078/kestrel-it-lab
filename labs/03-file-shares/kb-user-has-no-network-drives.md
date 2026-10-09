# KB · User has no network drives

For the Kestrel IT Service Desk. Tested in F3 (Priya's first login) and INC-0005.

**What normally happens:** when a staff member logs in, the GPO **USR - Department drive maps** gives them K: (Company) and their department's drive: F: (Finance), R: (HR) or O: (Operations).

## 1. One person or everyone?

- **Only this person:** keep going.
- **Everyone:** it's not the user. Check the DC is running and that `\\vm-kestrel-dc01` opens at all.

## 2. Have they logged off and on since the change?

Groups and drive maps are read at logon. A new starter added to a group a minute ago won't have the drive until they log in again. If you're not sure, ask them to log off and on, then check again.

## 3. Can they open the share by its path?

On their PC, in File Explorer or PowerShell:

```
\\vm-kestrel-dc01\Finance
```

| Result | Meaning |
|---|---|
| Opens | Permissions are fine. Only the drive mapping is missing. Go to step 4 |
| Access denied | They're missing the group. Go to step 5 |
| Can't find it | Network or DNS problem, not a drive problem. Check `nslookup vm-kestrel-dc01` |

## 4. Is the GPO reaching them?

As the user, run:

```
gpresult /r /scope user
```

| What you see | Meaning |
|---|---|
| *Applied Group Policy Objects:* **N/A** | The GPO isn't reaching them. Check the first line of USER SETTINGS. If it says `CN=Users`, their account is in the wrong place. Move it to Staff › Canberra (INC-0005) |
| **USR - Department drive maps** is listed, but no drives | The GPO arrived, but the drive map items didn't match them. Check the targeting (step 6) |

## 5. Do they have the right group?

On the DC:

```powershell
hostname
Get-ADUser <username> -Properties MemberOf | Select-Object DistinguishedName, MemberOf
```

They should be in their team's **GG-** group (for example GG-Finance) and GG-AllStaff. Add them to the GG- group, not to a DL- group, then get them to log off and on.

## 6. Check the drive map targeting

In the GPO, each drive map has item-level targeting on a DL- group. The group must be picked with the **…** button, which saves its SID. A typed name saves no SID, and the drive won't map for anyone.

To check without opening the editor, look at the GPO's settings file:

```
\\kestrel.local\SYSVOL\kestrel.local\Policies\{B0813F5E-19E1-485A-BD4C-32A4CB20DB6D}\User\Preferences\Drives\Drives.xml
```

Every `FilterGroup` should have a `sid="S-1-5-21-..."`. If one says `sid=""`, re-pick that group with **…** and Check Names.

## After fixing

Get the user to log off and on, then confirm with:

```
net use
```

Their drives should show as **OK**.
