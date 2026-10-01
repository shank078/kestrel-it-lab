# Naming and tagging

## Naming

Pattern: `<type>-kestrel-<what>-<region>`

| Type | Prefix | Example |
|------|--------|---------|
| Resource group | `rg-` | `rg-kestrel-core-aue` |
| Virtual network | `vnet-` | `vnet-kestrel-ause` |
| Subnet | `snet-` | `snet-servers` |
| Network security group | `nsg-` | `nsg-kestrel-ause` |
| Budget | `budget-` | `budget-kestrel-lab-monthly` |
| Lock | `lock-` | `lock-kestrel-core-nodelete` |

Region suffix: `ause` = Australia Southeast.

**Exception:** `rg-kestrel-core-aue` ends in `aue` (Australia East) because that was the original plan. Resource groups can't be renamed, so it stays.

The prefixes follow Microsoft's suggested abbreviations, so anyone used to Azure should recognise them straight away.

## Tags

The standard (lowercase, exactly as written):

| Tag | Value |
|-----|-------|
| `project` | `kestrel-lab` |
| `env` | `lab` |
| `owner` | `shankar` |

Why it matters: tag **values are case-sensitive**, so `Lab` and `lab` show up as two different values when you filter costs. Consistent tags = you can see exactly what Kestrel costs.

**Known issue:** the resource group and VNet were tagged with `projec`, `Lab` and `Owner: ShankarBaral` (typos from the first build). Left as they are for now.
