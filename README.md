# CyberArk Privilege Cloud Lab Environment

Technical documentation and implementation scripts for deploying a CyberArk Privilege Cloud lab environment with high availability, Active Directory integration, and multi-factor authentication.

---

## 📚 Official Documentation & References

| Resource | URL |
|----------|-----|
| **CyberArk Privilege Cloud Docs** | [docs.cyberark.com/privilege-cloud](https://docs.cyberark.com/privilege-cloud/latest/en/content/landing-pages/lprivcloud.htm) |
| **CyberArk Identity Docs** | [docs.cyberark.com/identity](https://docs.cyberark.com/identity/latest/en/content/landing-pages/lpidentity.htm) |
| **CyberArk Marketplace** | [cyberark-customers.force.com/mplace](https://cyberark-customers.force.com/mplace/s/#--Background) |
| **CyberArk Community** | [cyberark-customers.force.com](https://cyberark-customers.force.com/s/) |
| **CyberArk Support Portal** | [support.cyberark.com](https://support.cyberark.com) |
| **Microsoft TIER Model** | [learn.microsoft.com - Privileged Access Model](https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-access-model) |
| **Microsoft AD DS Documentation** | [learn.microsoft.com - AD DS](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview) |

---

## Overview

This repository contains the complete architecture and implementation guide for an enterprise-grade CyberArk Privilege Cloud lab environment, implementing:

- **Active Directory as Identity Provider** - LDAP/LDAPS integration with CyberArk Identity
- **TIER Model Directory Structure** - Privileged access management following Microsoft's TIER model
- **Multi-Factor Authentication** - Email OTP, SMS OTP, and Mobile App policies
- **High Availability Infrastructure** - PSM + SIA in HA configuration with DNS round robin
- **OKTA Federation** - SAML integration for external identity provider support

## Environment

| Component | Details |
|-----------|---------|
| Domain | `indramind.cyblab` |
| Domain Controller | DC01 (10.10.3.151) |
| PSM Server 1 | Connector01 (10.10.3.8) |
| PSM Server 2 | Connector02 (10.10.3.70) |
| PSM HA DNS | `psm.indramind.cyblab` |

## Documentation

| Document | Description |
|----------|-------------|
| [lab_pcloud_architecture.md](lab_pcloud_architecture.md) | Complete technical architecture with step-by-step HOW-TO guides for all components |
| [lab_pcloud_scripts.md](lab_pcloud_scripts.md) | PowerShell scripts for AD configuration, TIER model creation, and PSM deployment |

## Implementation Components

### 1. Identity Architecture

- AD Connector configuration for CyberArk Identity
- Service account: `svc-CyberArkIdentity@indramind.cyblab`
- LDAP sync interval: 15 minutes
- User/Group sync from `OU=PAM,DC=indramind,DC=cyblab`

### 2. TIER Model Structure

```
DC=indramind,DC=cyblab
└─ OU=PAM
   ├─ OU=Tier0 (Domain Controllers, Forest Admins)
   │  ├─ OU=Groups (PAM-T0-Admins, PAM-T0-Operators, PAM-T0-Auditors)
   │  └─ OU=Accounts
   ├─ OU=Tier1 (Enterprise Servers)
   │  ├─ OU=Groups (PAM-T1-Admins, PAM-T1-WindowsAdmins, PAM-T1-LinuxAdmins, etc.)
   │  └─ OU=Accounts
   ├─ OU=Tier2 (Workstations)
   │  ├─ OU=Groups (PAM-T2-Admins, PAM-T2-HelpDesk)
   │  └─ OU=Accounts
   ├─ OU=ServiceAccounts
   ├─ OU=AdminGroups
   └─ OU=Users
      ├─ OU=Active
      └─ OU=Disabled
```

### 3. MFA PolicySets

| Policy | Target | Methods |
|--------|--------|---------|
| LAB-2FA-Standard | PAM-Users | Email OTP, SMS OTP, Mobile App |
| LAB-2FA-HighSecurity | PAM-Vault-Admins, PAM-T0-Admins | Mobile App only (with biometric) |

### 4. Naming Conventions

**Safe Naming Format:** `{ENV}-{TIER}-{TYPE}-{OWNER}`

| Code | Environment | | Code | Tier | | Code | Type |
|------|-------------|-|------|------|-|------|------|
| LAB | Lab/Test | | T0 | Domain Controllers | | WIN | Windows |
| DEV | Development | | T1 | Enterprise Servers | | LNX | Linux |
| PRD | Production | | T2 | Workstations | | DB | Database |
| | | | SVC | Service Accounts | | NET | Network |

**Examples:**
- `LAB-T0-WIN-DomainAdmins`
- `LAB-T1-LNX-Servers`
- `PRD-T1-DB-Oracle`

### 5. Service Accounts

| Account | Purpose |
|---------|---------|
| `svc-CyberArkCPM` | CPM password management |
| `svc-CyberArkPSM` | PSM session management |
| `svc-CyberArkScan` | Account discovery/scanning |
| `svc-CyberArkIdentity` | Identity AD connector |
| `svc-CyberArkReconcile` | Reconciliation account |

## Quick Start

### Create TIER Model Structure

```powershell
# Run on DC01 as Administrator
.\Create-TierStructure.ps1 -DomainDN "DC=indramind,DC=cyblab"
```

### Verify AD Structure

```powershell
# List PAM OUs
Get-ADOrganizationalUnit -Filter * -SearchBase "OU=PAM,DC=indramind,DC=cyblab" |
    Select-Object Name, DistinguishedName

# List PAM Groups
Get-ADGroup -Filter * -SearchBase "OU=PAM,DC=indramind,DC=cyblab" |
    Select-Object Name, GroupScope

# Check PSM Services
Get-Service -Name "CyberArk*"
```

### Test Cloud Connectivity

```powershell
Test-NetConnection -ComputerName "connector.privilegecloud.cyberark.cloud" -Port 443
```

## Key URLs

| Service | URL |
|---------|-----|
| Privilege Cloud Portal | `https://[tenant].privilegecloud.cyberark.cloud` |
| CyberArk Identity Admin | `https://[tenant].id.cyberark.cloud/admin` |
| CyberArk Identity User Portal | `https://[tenant].id.cyberark.cloud` |

## Implementation Phases

1. **Phase 1: Foundation** - AD validation, TIER structure, service accounts, AD as IdP
2. **Phase 2: User Management** - User audit, email updates, domain accounts, disable departed users
3. **Phase 3: MFA & Infrastructure** - PolicySets, Email/Phone OTP, Mobile App, PSM HA deployment
4. **Phase 4: Standards & OKTA** - Naming conventions, OKTA SAML integration

## Requirements

- Windows Server 2022 for PSM/SIA servers
- 8 vCPU, 16GB RAM, 500GB disk per connector
- Domain-joined servers
- HTTPS/443 connectivity to CyberArk Cloud
- PowerShell with ActiveDirectory module on DC

---

## 📖 Additional Resources

### CyberArk Documentation

| Topic | Link |
|-------|------|
| System Requirements | [Privilege Cloud Requirements](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/system-requirements.htm) |
| Network Requirements | [Network Configuration](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/network-requirements.htm) |
| PSM Installation | [Install PSM Connector](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/install-connector.htm) |
| PSM High Availability | [PSM HA Configuration](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/psm-ha.htm) |
| SIA Overview | [Secure Infrastructure Access](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/sia-overview.htm) |
| MFA Configuration | [Multi-Factor Authentication](https://docs.cyberark.com/identity/latest/en/content/mfa/mfa.htm) |
| Safe Management | [Managing Safes](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/safes.htm) |
| REST API | [Web Services SDK](https://docs.cyberark.com/privilege-cloud/latest/en/content/webservices/webservices.htm) |

### Microsoft Documentation

| Topic | Link |
|-------|------|
| Active Directory | [AD DS Overview](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview) |
| TIER Model | [Privileged Access Security](https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-access-model) |
| PowerShell AD Module | [ActiveDirectory Module](https://learn.microsoft.com/en-us/powershell/module/activedirectory/) |
| Windows Server | [Windows Server Documentation](https://learn.microsoft.com/en-us/windows-server/) |
| Remote Desktop Services | [RDS Overview](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/welcome-to-rds) |

### Third-Party Integration

| Topic | Link |
|-------|------|
| Okta SAML | [Okta SAML Configuration](https://help.okta.com/en-us/content/topics/apps/apps_app_integration_wizard_saml.htm) |
| SAML 2.0 Specification | [OASIS SAML](http://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html) |

### Community & Tools

| Resource | Link |
|----------|------|
| psPAS PowerShell Module | [github.com/pspete/psPAS](https://github.com/pspete/psPAS) |
| CyberArk GitHub | [github.com/cyberark](https://github.com/cyberark) |
| CyberArk Community Forums | [CyberArk Community](https://cyberark-customers.force.com/s/) |

---

## 📋 License & Disclaimer

This documentation is provided for educational and implementation purposes within authorized lab environments. Always refer to official CyberArk documentation for production deployments.

**CyberArk** is a registered trademark of CyberArk Software Ltd. **Microsoft**, **Windows Server**, and **Active Directory** are trademarks of Microsoft Corporation. **Okta** is a trademark of Okta, Inc.
