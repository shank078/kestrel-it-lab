# F2 · Domain controller and first accounts

**Started:** 1 Oct 2026
**Status:** In progress. The domain is up and working. Still to do: move the IP setting to the Azure side (see "Known issues"), the break/fix exercise and the KB article.

## The request

> **REQ-0002** · From: Megan Doyle, Operations Manager, Kestrel Freight (fictional)
>
> Everyone at head office logs into their PC with their own local account, and when someone leaves we have no idea which PCs they could get into. I want one login per person that works on any office PC, and one place to switch it off when they leave. Sydney staff will need the same thing next year.

That's Active Directory. One server (the domain controller) holds every user account, and every PC checks logins against it. Disable someone there and they're locked out of everything at once.

## What I built

| What | Details |
|------|---------|
| Server | `vm-kestrel-dc01`, Windows Server 2022 Datacenter, size Standard_D2als_v6, Australia Southeast |
| Network | `snet-servers` (10.10.1.0/24), private IP **10.10.1.4** fixed in Azure |
| Domain | `kestrel.local`, new forest, functional level Windows Server 2016 |
| Roles | AD DS, DNS, Global Catalog |
| Admin account | `KESTREL\kadmin` |
| OUs | Staff (Canberra, Sydney), Computer_Resources (Workstations, Servers), Groups |
| Test user | James Smith (`jsmith`) in Staff › Canberra |

## How I did it

### 1. Creating the VM

![VM basics and cost estimate](evidence/01-vm-create-basics-cost-estimate.png)

Azure estimated about $160 a month for this VM if it runs 24/7. That's way over my A$50 budget, so it only runs while I'm working on it and gets shut down after every session.

![Networking](evidence/02-vm-networking-snet-servers.png)

It goes in `snet-servers`. No NSG on the network card itself, because the NSG from F0 is already attached to the whole subnet.

### 2. Fixing the server's IP

A domain controller is also the DNS server for the office, so every PC needs to find it at the same address forever. By default Azure gives out IPs dynamically, so I changed the network card's private IP from Dynamic to **Static, 10.10.1.4** (the first usable address in the subnet, as planned in the [IP plan](../../docs/standards/ip-plan.md)).

![Setting static](evidence/03-nic-private-ip-set-static.png)
![Static confirmed](evidence/04-nic-private-ip-static-confirmed.png)

### 3. First login and installing AD DS

RDP in from home (only my IP is allowed, from F0's NSG rule).

![First sign-in](evidence/05-rdp-first-sign-in.png)
![Server Manager](evidence/06-server-manager-first-look.png)

Then **Add Roles and Features → Active Directory Domain Services**. Installing the role only puts the software on. The server isn't a domain controller yet, which is why Server Manager shows the warning flag asking to promote it.

![Installing AD DS](evidence/07-adds-role-installing.png)
![Promote prompt](evidence/08-adds-installed-promote-prompt.png)

### 4. Promoting it to a domain controller

New forest, domain name `kestrel.local`. Kept DNS server and Global Catalog ticked. On a first DC you want both, because PCs find the domain through DNS and the Global Catalog is what logins search.

![DC options](evidence/09-dc-options-functional-level-dns-gc.png)

The DSRM password is a separate emergency password for repairing AD if it ever breaks. It's not the admin password.

![Promotion running](evidence/11-dc-promotion-installing.png)

The warning about "Windows NT 4.0 cryptography" is normal. It's telling you the DC blocks old weak encryption by default.

### 5. Checking it worked

After the restart, I log in as a domain account now, not a local one:

![Signed in as KESTREL\kadmin](evidence/12-signed-in-as-kestrel-kadmin.png)
![AD DS and DNS roles](evidence/13-server-manager-adds-dns-roles.png)
![Domain kestrel.local](evidence/14-local-server-joined-kestrel-local.png)

DNS showed one warning, event 4013. That's normal on a brand new DC's first boot: DNS starts before AD has finished loading and waits for it.

![DNS event 4013](evidence/15-dns-role-event-4013.png)
![nslookup and dsquery](evidence/17-nslookup-and-dsquery-check.png)

`nslookup` finds the DC at 10.10.1.4. `dsquery domain` failed because "domain" isn't a type dsquery knows. `dsquery ou` worked and showed the only OU so far: Domain Controllers.

### 6. OUs

A new domain only has the built-in folders. I made OUs for Kestrel's layout: staff by site, computers by type, and one place for groups.

![Empty domain](evidence/18-aduc-new-domain.png)
![OU structure](evidence/19-aduc-ou-structure.png)

I couldn't call the OU "Computers" because there's already a built-in **container** with that name. The built-in Users and Computers folders are containers, not OUs, so you can't link Group Policy to them. That's the reason for making my own.

### 7. First user, with PowerShell

![New-ADUser](evidence/20-new-aduser-jsmith.png)
![jsmith in Canberra](evidence/21-aduc-jsmith-in-canberra-ou.png)

(Password blacked out.) Typing a password in plain text in a command isn't great, since it stays in the PowerShell history. For real accounts I'd use `Read-Host -AsSecureString` and tick "change password at next logon".

## Testing

| Check | Result |
|-------|--------|
| Server is a DC for `kestrel.local` | ✅ Pass (screenshots 12–14) |
| AD DS and DNS roles running | ✅ Pass (13) |
| `nslookup` resolves the DC to 10.10.1.4 | ✅ Pass (17, 24) |
| OUs created in the planned layout | ✅ Pass (19) |
| `jsmith` created in Staff › Canberra | ✅ Pass (21) |
| `jsmith` can actually log in from a domain PC | Not tested yet. There's no client PC until a later lab |
| RDP blocked from anywhere except my IP | Still not tested (carried over from F0) |

## Things that went wrong 😅

**1. I locked myself out of the server.** I set the static IP inside Windows as well as in Azure, and lost RDP. I got back in through Azure's **Serial Console**, a text console in the portal that works even when the network is broken, and fixed the IP settings from there.

![After recovery](evidence/10-serial-console-ip-after-recovery.png)

I didn't screenshot the broken state, only the result afterwards. Lesson: in Azure the IP is fixed **on the Azure network card only** and Windows stays on DHCP. Azure's DHCP always hands it the same reserved address. Setting it inside Windows is how you'd do it on a physical server, but in Azure it can break the next time Azure changes anything about the VM. (Fixing this properly is still to do, see below.)

**2. DNS pointed at itself in a confusing way.** After promotion, the DNS servers were `::1` and `127.0.0.1`, which are both "this computer" (IPv6 and IPv4). The promotion wizard sets that.

![Before](evidence/16-dns-servers-loopback-before-fix.png)

I changed IPv4 to 10.10.1.4, but `::1` was still listed first:

![::1 still there](evidence/22-ipv6-loopback-still-listed.png)
![netsh attempts](evidence/23-netsh-ipv4-attempts.png)

That's because IPv4 and IPv6 have **separate** DNS settings, so the IPv4 command can't touch the IPv6 entry. The fix was the IPv6 command:

![Fixed](evidence/24-ipv6-dns-cleared-nslookup.png)

`nslookup` still says `Server: UnKnown`. That's a different thing. nslookup tries to look up a name for the DNS server's own IP, and there's no reverse lookup zone yet. That gets built in the DNS lab (F5).

## Known issues / still to do

- **Move the IP config to Azure.** Put Windows back on DHCP, and set DNS (10.10.1.4) on the Azure network card instead.
- **`.local` domain name.** Microsoft recommends a subdomain of a real domain (like `ad.kestrelfreight.com.au`). `.local` also clashes with how Macs find devices on the network, which will matter in the Mac lab. Keeping it for now and noting it here.
- **Firewall profile shows "Private".** On a DC it should be "Domain". Need to check this.
- **The DC has a public IP.** OK for a lab because RDP only accepts my IP, but a real DC would never face the internet.
- Break/fix exercise and KB article.

## Cost

Only runs during lab sessions and gets shut down after. Actual cost to be added from Cost Management.

## Next

Finish the IP fix, then break/fix and the KB article.
