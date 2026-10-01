# IP plan

## Kestrel head office (Azure, Australia Southeast)

**Network:** `vnet-kestrel-ause` · 10.10.0.0/16 (10.10.0.0 to 10.10.255.255)

| Subnet | Range | Used for |
|--------|-------|----------|
| `snet-servers` | 10.10.1.0/24 | Domain controller, file and print server |
| `snet-clients` | 10.10.2.0/24 | Staff PCs |

A /16 means the first two numbers (10.10) are fixed and the last two can change. A /24 fixes the first three, so each subnet has 256 addresses.

Azure keeps 5 addresses in every subnet for itself (the first four and the last one). So in `snet-servers` the first one I can actually use is **10.10.1.4**. That's where the domain controller will go in F2.

## Planned (not built yet)

| Site | Range | When |
|------|-------|------|
| Sydney warehouse | 10.20.0.0/16 | Second site, later |
| Home lab (Cisco gear) | 10.30.0.0/16 | Networking labs |

These are kept separate on purpose. If two sites use overlapping ranges, you can't connect them later without re-addressing one of them.
