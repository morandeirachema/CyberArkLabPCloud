# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a documentation repository for implementing a CyberArk Privilege Cloud lab environment. It contains:

- **Architecture documentation** (`lab_pcloud_architecture.md`) - Technical architecture and step-by-step implementation guides
- **PowerShell scripts** (`lab_pcloud_scripts.md`) - Scripts for AD configuration, TIER model setup, and PSM deployment
- **Requirements** (`PeticionesBardera.md`) - Original project requirements (Spanish)

## Domain and Environment

- **Domain**: `indramind.cyblab`
- **Domain Controller**: DC01 (10.10.3.151)
- **PSM Servers**: Connector01 (10.10.3.8), Connector02 (10.10.3.70)
- **Cloud Tenant**: `[tenant].privilegecloud.cyberark.cloud` and `[tenant].id.cyberark.cloud`

## Key Implementation Areas

1. **AD Integration** - Active Directory as Identity Provider for CyberArk Identity
2. **TIER Model** - OU structure with Tier0/Tier1/Tier2 for privileged access management
3. **MFA PolicySets** - Email OTP, Phone OTP, and Mobile App authentication
4. **PSM + SIA HA** - High availability setup with DNS round robin (`psm.indramind.cyblab`)
5. **Naming Conventions** - Safe format: `{ENV}-{TIER}-{TYPE}-{OWNER}` (e.g., `LAB-T0-WIN-DomainAdmins`)

## PowerShell Scripts Location

All scripts are embedded in `lab_pcloud_scripts.md`. Key scripts:
- `Create-TierStructure.ps1` - Creates complete PAM OU structure and security groups
- AD Connector service account creation
- User management (bulk email updates, account creation/disabling)
- PSM server preparation and verification

## Service Accounts

Located in `OU=ServiceAccounts,OU=PAM,DC=indramind,DC=cyblab`:
- `svc-CyberArkCPM` - Password management
- `svc-CyberArkPSM` - Session management
- `svc-CyberArkScan` - Account discovery
- `svc-CyberArkIdentity` - AD connector
- `svc-CyberArkReconcile` - Reconciliation
