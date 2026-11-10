# Azure Small Business IT Infrastructure Lab

## Client Brief
Trinity Dental Group is a fictional 15-person dental practice with three
departments: Front Office, Clinical, and Admin. They currently share
passwords, store files on individual PCs, have no tested backups, and have
no way to detect suspicious activity. This project designs and builds a
secure, recoverable, monitored cloud IT environment for them in Microsoft
Azure, on a small-business budget.

## Architecture
(diagram coming)

## Phases
### Phase 1: Foundation
**Status:** Complete

- Created a dedicated Microsoft Entra tenant for the client, renamed to
  "Trinity Dental Group"
- Created resource group `rg-trinity-prod` (Central US)
- Naming convention: `<type>-<company>-<environment>`
  (e.g., `rg-trinity-prod`, `vnet-trinity-prod`)
- Infrastructure (Phases 3–5) will run on a separate Azure for Students
  subscription (see Lessons Learned)
  
### Phase 2: Identity & Access
**Status:** Complete

- Created employee accounts with job title, department, and company attributes
- Created department security groups: `grp-front-office`, `grp-clinical`, `grp-admin`
- Enabled MFA for all users via Entra ID security defaults; tested by enrolling
  a user in Microsoft Authenticator
- Restricted non-admin access to the Microsoft Entra admin center
- Applied least-privilege RBAC on `rg-trinity-prod`:

| Group | Role | Reason |
|---|---|---|
| grp-admin | Contributor | Office admin manages IT resources |
| grp-front-office | Reader | Can view resources, can't change them |
| grp-clinical | No access | No business need for cloud resource access |
  
### Phase 3: Network & Compute

**Status:** Complete

All Phase 3 resources run on the Azure for Students subscription in resource
group `rg-trinity-infra`, region **North Central US** (see Lessons Learned).
Every resource is tagged `project=trinity-dental` and `env=lab`.

**Cost guardrail (built first):** budget `budget-trinity-lab`, $15/month, with
email alerts at 50%, 80%, and 100%.

**Network design:** one VNet segmented by role, so servers and staff
workstations are isolated from each other.

| Resource | Address range | Purpose |
|---|---|---|
| `vnet-trinity-prod` | 10.10.0.0/16 | Practice network |
| `snet-servers` | 10.10.1.0/24 | File server and future servers |
| `snet-workstations` | 10.10.2.0/24 | Reserved for staff workstations |

Both subnets are private subnets (no default outbound internet access).

**Network security groups:** attached at the subnet level so every resource in
a subnet inherits the same rules.

| NSG | Subnet | Custom inbound rules |
|---|---|---|
| `nsg-trinity-servers` | `snet-servers` | `Allow-RDP-MyIP`: TCP 3389 from one admin IP only (`<my-home-IP>/32`), priority 100 |
| `nsg-trinity-workstations` | `snet-workstations` | None. Default `DenyAllInBound` blocks all internet traffic |

**File server VM:** `vm-trinity-fs01`

| Setting | Value | Reason |
|---|---|---|
| Image | Windows Server 2022 Datacenter: Azure Edition (smalldisk, Gen2) | Windows security events feed Sentinel in Phase 5 |
| Size | Standard_B2als_v2 (2 vCPU, 4 GiB) | Burstable, lowest-cost size that runs Windows Server well |
| OS disk | 30 GiB Standard SSD | Small disk keeps the always-on storage cost low |
| Security type | Trusted launch (Secure Boot + vTPM) | Protects against boot-level malware |
| Public inbound ports | None on the VM | RDP is controlled only by the subnet NSG |
| Managed identity | System-assigned | Used by the Azure Monitor Agent in Phase 5 |
| Auto-shutdown | 11:00 PM Central | Prevents paying for idle compute |

**Validation: RDP access control**

| Test | Source | Expected | Result |
|---|---|---|---|
| A | Home network (allowed IP) | Connects | ✅ Logged in to Windows Server desktop |
| B | Phone hotspot (any other IP) | Blocked | ✅ Connection timed out (error 0x704) |

Screenshots are in [`docs/screenshots/`](docs/screenshots/). Public IPs and
subscription IDs are redacted.

**Infrastructure as code:** the ARM templates exported from each deployment are
in [`templates/`](templates/): `vnet-trinity-prod.json`,
`nsg-trinity-servers.json`, `nsg-trinity-workstations.json`,
`vm-trinity-fs01.json`.

### Phase 4: Storage & Backup

**Status:** Not started

### Phase 5: Security Monitoring (Microsoft Sentinel)

**Status:** Not started

## Lessons Learned

**Student subscription landed in the university's directory.**
Signing up for Azure for Students with my university account placed the
subscription in the university's Entra tenant, where I had only User-level
access, so I couldn't create users or manage identity settings.

**Evaluated and rejected the "Governed Workforce" tenant option.**
Creating a new tenant from the university account would have established a
governance relationship giving the university control over the new tenant,
and it requires a paid subscription. I chose a different approach.

**University policy blocked subscription transfer.**
I created a separate tenant under a personal account and attempted to move
the student subscription into it. The university's subscription policy
blocked the transfer.

**Resolution: split architecture.** Identity lives in the dedicated
tenant; infrastructure runs on the student subscription. Security monitoring will use VM Windows security events instead of Entra sign-in logs.

**A regional policy blocked the first VNet deployment.**
The VNet failed validation with a policy violation, and the error pane showed
no details. Under Policy → Assignments I found the subscription's only policy,
*Allowed resource deployment regions*, which permits only `westus`,
`northcentralus`, `canadacentral`, `denmarkeast`, and `belgiumcentral`. Central
US was not allowed. I redeployed in North Central US, the closest allowed
region. The resource group stayed in Central US, since a resource group's
region only stores metadata and doesn't restrict where its resources run.

**The default VM size wasn't available.**
The portal defaulted to Standard_D2s_v3, which returned
`NotAvailableForSubscription` in North Central US. I compared the B-series v2
sizes and chose Standard_B2als_v2. It has enough memory for Windows Server
(the 1 GiB B2ats_v2 does not) at about half the cost of B2as_v2.

## Estimated Monthly Cost

Identity (Entra ID Free tier), resource groups, the VNet, and NSGs have no charge.

| Item | Approx. monthly cost | Notes |
|---|---|---|
| VM compute (B2als_v2) | ~$0.05/hr while running | Stopped (deallocated) between sessions, plus auto-shutdown |
| OS disk (30 GiB Standard SSD) | ~$2–3 | Billed even when the VM is stopped |
| Public IP (Standard, static) | ~$3–4 | Billed even when the VM is stopped |
| **Typical lab month** | **~$8–12** | Within the $15 budget |

Running the VM 24/7 would cost about $34/month for compute alone, which is why
deallocating it and auto-shutdown matter.
