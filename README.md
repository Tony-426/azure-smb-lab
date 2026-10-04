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
**Status:** In progress

- Created 3 employee accounts with job title, department, and company
  attributes (Front Office Manager, Dental Hygienist, Office Administrator)
- Created security group `grp-front-office` with assigned membership
- Next: remaining department groups, MFA, and role-based access control
  
### Phase 3: Network & Compute
### Phase 4: Storage & Backup
### Phase 5: Security Monitoring (Microsoft Sentinel)

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
tenant; infrastructure runs on the student subscription. Security monitoring
was adapted to use VM Windows security events instead of Entra sign-in logs.

## Estimated Monthly Cost
