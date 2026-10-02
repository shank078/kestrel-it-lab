# Runbook · Kestrel domain controller (DC01)

For the service desk and whoever looks after `vm-kestrel-dc01`. Everything here is run in **PowerShell as Administrator** on the DC, signed in as `KESTREL\kadmin`, unless it says otherwise.

| | |
|---|---|
| **Server** | `vm-kestrel-dc01` · 10.10.1.4 (Azure, Australia Southeast) |
| **Domain** | `kestrel.local` |
| **Admin account** | `KESTREL\kadmin` |
| **User OUs** | `Staff › Canberra`, `Staff › Sydney` |
| **Last updated** | 2 Oct 2026 |

Each procedure says whether I've actually run it in the lab. If it says "not tested yet", treat it with care.

---

## 1. New starter: create a user

**Tested in lab:** Yes (`jsmith`), but with the password typed into the command. The version below is the safer way.

```powershell
New-ADUser -Name "Jane Citizen" -GivenName "Jane" -Surname "Citizen" `
  -SamAccountName "jcitizen" -UserPrincipalName "jcitizen@kestrel.local" `
  -Path "OU=Canberra,OU=Staff,DC=kestrel,DC=local" `
  -AccountPassword (Read-Host -AsSecureString "Temporary password") `
  -ChangePasswordAtLogon $true -Enabled $true
```

- Change the names, and `Canberra` to `Sydney` if needed.
- `Read-Host -AsSecureString` asks for the password without showing it, so it doesn't end up in the PowerShell history.
- `-ChangePasswordAtLogon $true` means only the user ever knows their real password.

**Check it worked:**

```powershell
Get-ADUser jcitizen -Properties Enabled, DistinguishedName | Select-Object Name, Enabled, DistinguishedName
```

It should say `Enabled: True` and show the right OU.

---

## 2. Forgotten password: reset it

**Tested in lab:** Not tested yet.

First, check you're talking to the real person (follow the identity check process). Then:

```powershell
Set-ADAccountPassword -Identity jcitizen -Reset -NewPassword (Read-Host -AsSecureString "Temporary password")
Set-ADUser -Identity jcitizen -ChangePasswordAtLogon $true
```

Give them the temporary password by phone or in person, never in the ticket.

---

## 3. "I'm locked out": unlock the account

**Tested in lab:** Not tested yet. See the note below.

See who's locked out:

```powershell
Search-ADAccount -LockedOut | Select-Object Name, SamAccountName
```

Unlock one person:

```powershell
Unlock-ADAccount -Identity jcitizen
Get-ADUser jcitizen -Properties LockedOut | Select-Object Name, LockedOut
```

`LockedOut` should now be `False`.

> **Note:** a brand new domain might not lock anyone out at all, because the lockout threshold can be 0 (never lock). Check it with `Get-ADDefaultDomainPasswordPolicy` and look at `LockoutThreshold`. Setting a proper lockout policy is part of the Group Policy lab.

If the same person keeps getting locked out, the cause is usually an old password saved somewhere: a phone's email app, a mapped drive, or a second PC. Unlocking alone won't fix that.

---

## 4. Leaver: disable the account

**Tested in lab:** Not tested yet.

Disable it, don't delete it. A disabled account can be turned back on if someone left by mistake, and their group memberships are still there to check.

```powershell
Disable-ADAccount -Identity jcitizen
Get-ADUser jcitizen | Select-Object Name, Enabled
```

`Enabled` should be `False`. To see which groups they were in (worth recording on the ticket before anything is removed):

```powershell
Get-ADPrincipalGroupMembership -Identity jcitizen | Select-Object Name
```

---

## 5. Look up a user's account status

**Tested in lab:** Not tested yet.

The first thing to check on most "can't log in" tickets:

```powershell
Get-ADUser jcitizen -Properties Enabled, LockedOut, PasswordLastSet, PasswordExpired, AccountExpirationDate |
  Select-Object Name, Enabled, LockedOut, PasswordLastSet, PasswordExpired, AccountExpirationDate
```

| If you see | Then |
|---|---|
| `Enabled: False` | Account is disabled. Check it's not a leaver before turning it back on |
| `LockedOut: True` | Section 3 |
| `PasswordExpired: True` | Section 2 |
| `AccountExpirationDate` in the past | Contractor end date passed. Needs a manager's OK to extend |

---

## 6. DNS: internal or internet names not working

**Tested in lab:** Yes, see [INC-0002](incident-02-dc-cant-resolve-internet-names.md).

The DC is also Kestrel's DNS server. Work through this in order. Each step rules something out.

| Step | Command | If it works | If it fails |
|---|---|---|---|
| 1. Internal name | `nslookup vm-kestrel-dc01.kestrel.local` | DNS service is up | DNS service or the kestrel.local zone. Check `Get-Service DNS` |
| 2. Internet name | `nslookup microsoft.com` | All fine | Go to step 3 |
| 3. Network to a public DNS server | `Test-NetConnection 8.8.8.8 -Port 53` | Network is fine, problem is the DC's DNS setup | Network, NSG or firewall problem, not DNS |
| 4. Skip the DC | `nslookup microsoft.com 8.8.8.8` | Internet DNS is fine, so the DC is the problem | Wider internet issue |
| 5. Check forwarders | `Get-DnsServerForwarder` | Compare against the known-good below | |

**Known-good forwarder settings:** forwarder `8.8.8.8`, `UseRootHint: True`.

To put it back:

```powershell
Set-DnsServerForwarder -IPAddress 8.8.8.8 -UseRootHint $true
Clear-DnsServerCache -Force
Clear-DnsClientCache
```

Then run steps 1 and 2 again to confirm.

> `Server: UnKnown` at the top of every nslookup is normal for now. There's no reverse lookup zone yet. It's not a fault.

---

## 7. Quick DC health check

**Tested in lab:** Not tested yet.

Worth doing after any change or restart:

```powershell
Get-Service NTDS, DNS, Netlogon, Kdc | Select-Object Name, Status
dcdiag /q
```

- All four services should be `Running`. NTDS is AD itself, Kdc handles Kerberos logins, and Netlogon is what PCs talk to when they log in.
- `dcdiag /q` only prints errors. No output means no errors found.

---

## 8. Can't RDP to the DC

**Tested in lab:** Yes (the lockout in the F2 README).

1. Azure portal: is the VM **running**? Deallocated VMs don't answer.
2. Has your home IP changed? The NSG only allows RDP from one IP (see F0). Compare your current IP with the `Allow-RDP-MyIP` rule.
3. Still nothing? Use **Serial Console** in the portal (vm-kestrel-dc01 › Help › Serial console). It works even when the network settings are broken. Run `ipconfig` there. A 169.254.x.x address means Windows has no valid IP.

The private IP should be **10.10.1.4**. It's set on the Azure network card and in Windows, and the two must match.
