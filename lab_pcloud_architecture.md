# Lab Privilege Cloud - Technical Architecture & Implementation Guide

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                    CYBERARK PRIVILEGE CLOUD LAB ENVIRONMENT                      ║
║                      Technical Architecture Document                              ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

## Document Control

| Property | Value |
|----------|-------|
| **Document ID** | LAB-ARCH-001 |
| **Version** | 3.0 |
| **Status** | Production Ready |
| **Classification** | Internal Use |
| **Last Updated** | January 2026 |
| **Owner** | Lab Team |

### Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Jan 2026 | Lab Team | Initial architecture document |
| 2.0 | Jan 2026 | Lab Team | Added HOW-TO implementation steps |
| 3.0 | Jan 2026 | Lab Team | A++ refactor with validation checklists and references |

---

## 📚 Quick Reference

### Key URLs

| Service | URL | Purpose |
|---------|-----|---------|
| Privilege Cloud Portal | `https://[tenant].privilegecloud.cyberark.cloud` | Account management, sessions |
| Identity Admin | `https://[tenant].id.cyberark.cloud/admin` | User/MFA administration |
| Identity User Portal | `https://[tenant].id.cyberark.cloud` | End-user access |

### Infrastructure Summary

| Component | Server | IP Address | Role |
|-----------|--------|------------|------|
| Domain Controller | DC01 | 10.10.3.151 | AD DS, DNS, LDAPS |
| PSM Primary | Connector01 | 10.10.3.8 | PSM + SIA |
| PSM Secondary | Connector02 | 10.10.3.70 | PSM + SIA |
| PSM HA DNS | psm.indramind.cyblab | Round Robin | Load balancing |

### Service Accounts

| Account | Purpose | Location |
|---------|---------|----------|
| svc-CyberArkIdentity | AD Connector | OU=ServiceAccounts,OU=PAM |
| svc-CyberArkPSM | PSM Service | OU=ServiceAccounts,OU=PAM |
| svc-CyberArkScan | Discovery | OU=ServiceAccounts,OU=PAM |
| svc-CyberArkReconcile | Reconciliation | OU=ServiceAccounts,OU=PAM |

---

## 📖 External References

### CyberArk Documentation

| Topic | Link |
|-------|------|
| Privilege Cloud Overview | [docs.cyberark.com/privilege-cloud](https://docs.cyberark.com/privilege-cloud/latest/en/content/landing-pages/lprivcloud.htm) |
| CyberArk Identity | [docs.cyberark.com/identity](https://docs.cyberark.com/identity/latest/en/content/landing-pages/lpidentity.htm) |
| System Requirements | [Privilege Cloud Requirements](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/system-requirements.htm) |
| Network Requirements | [Network Configuration](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/network-requirements.htm) |
| AD Integration | [Identity AD Integration](https://docs.cyberark.com/identity/latest/en/content/integrations/ad/ad-integration.htm) |
| MFA Configuration | [Multi-Factor Authentication](https://docs.cyberark.com/identity/latest/en/content/mfa/mfa.htm) |
| PSM Installation | [Install Connector](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/install-connector.htm) |
| PSM High Availability | [PSM HA Configuration](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/psm-ha.htm) |
| SIA Overview | [Secure Infrastructure Access](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/sia-overview.htm) |
| Safe Management | [Managing Safes](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/safes.htm) |
| SAML Federation | [External IdP](https://docs.cyberark.com/identity/latest/en/content/integrations/saml/saml-external-idp.htm) |

### Microsoft Documentation

| Topic | Link |
|-------|------|
| Active Directory DS | [AD DS Overview](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview) |
| TIER Model | [Privileged Access Model](https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-access-model) |
| Securing Privileged Access | [Security Best Practices](https://learn.microsoft.com/en-us/security/privileged-access-workstations/overview) |
| PowerShell AD Module | [ActiveDirectory Cmdlets](https://learn.microsoft.com/en-us/powershell/module/activedirectory/) |
| Remote Desktop Services | [RDS Overview](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/welcome-to-rds) |

### Third-Party & Community

| Resource | Link |
|----------|------|
| Okta SAML | [Okta Documentation](https://help.okta.com/en-us/content/topics/apps/apps_app_integration_wizard_saml.htm) |
| psPAS Module | [github.com/pspete/psPAS](https://github.com/pspete/psPAS) |
| CyberArk Community | [Community Portal](https://cyberark-customers.force.com/s/) |
| CyberArk Marketplace | [Marketplace](https://cyberark-customers.force.com/mplace/s/#--Background) |
| CyberArk Support | [support.cyberark.com](https://support.cyberark.com) |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Requirements Mapping](#2-requirements-mapping)
3. [Identity Architecture](#3-identity-architecture)
4. [Directory Structure (TIER Model)](#4-directory-structure-tier-model)
5. [User Management](#5-user-management)
6. [Multi-Factor Authentication](#6-multi-factor-authentication)
7. [Mobile App Configuration](#7-mobile-app-configuration)
8. [High Availability Infrastructure](#8-high-availability-infrastructure)
9. [Naming Conventions](#9-naming-conventions)
10. [OKTA Integration](#10-okta-integration)
11. [Implementation Roadmap](#11-implementation-roadmap)
12. [Validation Checklists](#12-validation-checklists)
13. [Glossary](#13-glossary)

---

## 1. Executive Summary

### 1.1 Purpose

This document defines the complete technical architecture for the CyberArk Privilege Cloud lab environment. It serves as the authoritative reference for:

- Infrastructure design and component relationships
- Step-by-step implementation procedures (HOW-TO guides)
- Configuration standards and naming conventions
- Validation criteria and acceptance testing

### 1.2 Scope

The architecture addresses eight key implementation areas:

| # | Area | Component | Priority |
|---|------|-----------|----------|
| 1 | Active Directory Integration | CyberArk Identity | HIGH |
| 2 | TIER Model Directory Structure | Active Directory | HIGH |
| 3 | User Account Management | Identity Admin | MEDIUM |
| 4 | Multi-Factor Authentication | CyberArk Identity | HIGH |
| 5 | Mobile App Configuration | CyberArk Identity | MEDIUM |
| 6 | High Availability (PSM + SIA) | Privilege Cloud | HIGH |
| 7 | Naming Conventions | Vault Admin | HIGH |
| 8 | OKTA Integration | SAML Federation | LOW |

### 1.3 Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                              ARCHITECTURE OVERVIEW                               ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║                                                                                  ║
║  ┌────────────────────────────────────────────────────────────────────────────┐  ║
║  │                        CYBERARK PRIVILEGE CLOUD                            │  ║
║  │                   (*.privilegecloud.cyberark.cloud)                        │  ║
║  │                                                                            │  ║
║  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │  ║
║  │   │    VAULT     │  │     CPM      │  │   PVWA       │  │   IDENTITY   │   │  ║
║  │   │   (Cloud)    │  │   (Cloud)    │  │   (Cloud)    │  │   (Cloud)    │   │  ║
║  │   └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │  ║
║  └────────────────────────────────────────────────────────────────────────────┘  ║
║                                       │                                          ║
║                                       │ HTTPS/443 (Outbound)                     ║
║                                       ▼                                          ║
║  ┌────────────────────────────────────────────────────────────────────────────┐  ║
║  │                    ON-PREMISES CONNECTOR ZONE                              │  ║
║  │                      (indramind.cyblab)                                    │  ║
║  │                                                                            │  ║
║  │   ┌────────────────────────────────────────────────────────────────────┐   │  ║
║  │   │              DNS ROUND ROBIN: psm.indramind.cyblab                 │   │  ║
║  │   └─────────────────────────────┬──────────────────────────────────────┘   │  ║
║  │                    ┌────────────┴────────────┐                             │  ║
║  │                    ▼                         ▼                             │  ║
║  │   ┌─────────────────────────┐   ┌─────────────────────────┐                │  ║
║  │   │      CONNECTOR01        │   │      CONNECTOR02        │                │  ║
║  │   │      10.10.3.8          │   │      10.10.3.70         │                │  ║
║  │   │                         │   │                         │                │  ║
║  │   │  ┌───────────────────┐  │   │  ┌───────────────────┐  │                │  ║
║  │   │  │   PSM Component   │  │   │  │   PSM Component   │  │                │  ║
║  │   │  └───────────────────┘  │   │  └───────────────────┘  │                │  ║
║  │   │  ┌───────────────────┐  │   │  ┌───────────────────┐  │                │  ║
║  │   │  │   SIA Connector   │  │   │  │   SIA Connector   │  │                │  ║
║  │   │  └───────────────────┘  │   │  └───────────────────┘  │                │  ║
║  │   └─────────────────────────┘   └─────────────────────────┘                │  ║
║  │                    │                         │                             │  ║
║  │                    └────────────┬────────────┘                             │  ║
║  │                                 │                                          │  ║
║  │                                 ▼                                          │  ║
║  │   ┌─────────────────────────────────────────────────────────────────────┐  │  ║
║  │   │                    DOMAIN CONTROLLER (DC01)                         │  │  ║
║  │   │                       10.10.3.151                                   │  │  ║
║  │   │                                                                     │  │  ║
║  │   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │  │  ║
║  │   │   │   AD DS     │  │    DNS      │  │   LDAPS     │                 │  │  ║
║  │   │   └─────────────┘  └─────────────┘  └─────────────┘                 │  │  ║
║  │   └─────────────────────────────────────────────────────────────────────┘  │  ║
║  └────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                  ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

---

## 2. Requirements Mapping

### 2.1 Requirements Matrix

| REQ | Requirement | Component | Priority | Phase | Status |
|-----|-------------|-----------|----------|-------|--------|
| R01 | AD Integration as IdP | CyberArk Identity | HIGH | 1 | ☐ |
| R02 | TIER Model Directory | Active Directory | HIGH | 1 | ☐ |
| R03 | User Account Cleanup | Identity Admin | MEDIUM | 2 | ☐ |
| R04 | 2FA PolicySets | CyberArk Identity | HIGH | 3 | ☐ |
| R05 | Mobile App Config | CyberArk Identity | MEDIUM | 3 | ☐ |
| R06 | PSM + SIA HA | Privilege Cloud | HIGH | 3 | ☐ |
| R07 | Naming Conventions | Vault Admin | HIGH | 4 | ☐ |
| R08 | OKTA Integration | Identity/SAML | LOW | 4 | ☐ |

### 2.2 Priority Legend

| Priority | Definition | Timeline |
|----------|------------|----------|
| **HIGH** | Critical for core functionality | Phase 1-2 |
| **MEDIUM** | Important for operations | Phase 3 |
| **LOW** | Enhancement/Optional | Phase 4+ |

---

## 3. Identity Architecture

> **📚 References:**
> - [CyberArk Identity AD Integration](https://docs.cyberark.com/identity/latest/en/content/integrations/ad/ad-integration.htm)
> - [Microsoft LDAP Overview](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ldap/lightweight-directory-access-protocol-ldap-api)

### 3.1 Identity Provider Design

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                          IDENTITY ARCHITECTURE                                  │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                      CYBERARK IDENTITY TENANT                            │  │
│  │                  ([tenant].id.cyberark.cloud)                            │  │
│  │                                                                          │  │
│  │  IDENTITY SOURCES:                                                       │  │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐        │  │
│  │  │    PRIMARY       │  │   SECONDARY      │  │    FUTURE        │        │  │
│  │  │                  │  │                  │  │                  │        │  │
│  │  │  Active          │  │  Cloud           │  │  OKTA            │        │  │
│  │  │  Directory       │  │  Directory       │  │  SAML            │        │  │
│  │  │  (indramind      │  │  (Local Users)   │  │  Federation      │        │  │
│  │  │   .cyblab)       │  │                  │  │                  │        │  │
│  │  │                  │  │                  │  │                  │        │  │
│  │  │  Protocol:       │  │  Protocol:       │  │  Protocol:       │        │  │
│  │  │  LDAPS (636)     │  │  Internal        │  │  SAML 2.0        │        │  │
│  │  └──────────────────┘  └──────────────────┘  └──────────────────┘        │  │
│  │                                                                          │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 AD Connector Configuration

| Parameter | Value |
|-----------|-------|
| **Directory Name** | LAB-AD |
| **Server Address** | DC01.indramind.cyblab |
| **IP Address** | 10.10.3.151 |
| **Port** | 636 (LDAPS) / 389 (LDAP) |
| **Use SSL** | Yes (recommended) |
| **Base DN** | DC=indramind,DC=cyblab |
| **User Container** | OU=PAM,DC=indramind,DC=cyblab |
| **Group Container** | OU=PAM,DC=indramind,DC=cyblab |
| **Sync Interval** | 15 minutes |
| **Sync Nested Groups** | Yes |

### 3.3 AD Connector Service Account

| Attribute | Value |
|-----------|-------|
| **Account Name** | svc-CyberArkIdentity |
| **UPN** | svc-CyberArkIdentity@indramind.cyblab |
| **Location** | OU=ServiceAccounts,OU=PAM,DC=indramind,DC=cyblab |
| **Bind DN** | CN=svc-CyberArkIdentity,OU=ServiceAccounts,OU=PAM,DC=indramind,DC=cyblab |
| **Permissions** | Read-only access to OU=PAM |
| **Password Policy** | Non-expiring, managed by CyberArk |

### 3.4 HOW-TO: Configure AD as Identity Provider

#### Step 1: Create Service Account

> **Script:** [lab_pcloud_scripts.md - Section 1](lab_pcloud_scripts.md#1-ad-connector-service-account)

Run on DC01 as Domain Administrator.

#### Step 2: Configure AD Connector in CyberArk Identity

1. **Access Admin Portal**
   ```
   URL: https://[tenant].id.cyberark.cloud/admin
   ```

2. **Navigate to Directory Services**
   ```
   Settings → Core Services → Directory Services → Add Directory
   ```

3. **Configure Connection**
   ```
   Directory Type:     Active Directory
   Directory Name:     LAB-AD

   Connection:
   ├── Server:         DC01.indramind.cyblab
   ├── Port:           636 (LDAPS)
   ├── Use SSL:        Yes
   ├── Base DN:        DC=indramind,DC=cyblab
   └── Bind DN:        CN=svc-CyberArkIdentity,OU=ServiceAccounts,OU=PAM,DC=indramind,DC=cyblab

   Sync Settings:
   ├── User Container:   OU=PAM,DC=indramind,DC=cyblab
   ├── Group Container:  OU=PAM,DC=indramind,DC=cyblab
   ├── Sync Interval:    15 minutes
   └── Sync Nested:      Yes
   ```

4. **Configure LDAP Filter (Optional)**
   ```ldap
   (&(objectClass=user)(objectCategory=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
   ```

5. **Test and Save**
   - Click "Test Connection" → Verify success
   - Click "Save" → Click "Sync Now"

#### Step 3: Verify Integration

| Check | Expected Result | Status |
|-------|-----------------|--------|
| Sync Status | "Last Sync: Successful" | ☐ |
| User Count | > 0 users synced | ☐ |
| Group Count | PAM groups visible | ☐ |
| Test Login | AD user authenticates | ☐ |

---

## 4. Directory Structure (TIER Model)

> **📚 References:**
> - [Microsoft Privileged Access Model](https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-access-model)
> - [Microsoft Securing Privileged Access](https://learn.microsoft.com/en-us/security/privileged-access-workstations/overview)

### 4.1 OU Architecture

```
DC=indramind,DC=cyblab
│
└─ OU=PAM ─────────────────────────────────────────────────────────────────────
   │
   ├─ OU=Tier0 ─────────────────── Domain Controllers, Forest Admins
   │  ├─ OU=Groups
   │  │  ├─ PAM-T0-Admins ──────── Full DC access
   │  │  ├─ PAM-T0-Operators ───── Limited DC operations
   │  │  └─ PAM-T0-Auditors ────── Read-only DC access
   │  └─ OU=Accounts
   │
   ├─ OU=Tier1 ─────────────────── Enterprise Servers
   │  ├─ OU=Groups
   │  │  ├─ PAM-T1-Admins ──────── General server admins
   │  │  ├─ PAM-T1-WindowsAdmins ─ Windows server specialists
   │  │  ├─ PAM-T1-LinuxAdmins ─── Linux/Unix administrators
   │  │  ├─ PAM-T1-DatabaseAdmins  DBA team
   │  │  └─ PAM-T1-Operators ───── Limited server access
   │  └─ OU=Accounts
   │
   ├─ OU=Tier2 ─────────────────── Workstations
   │  ├─ OU=Groups
   │  │  ├─ PAM-T2-Admins ──────── Workstation administrators
   │  │  └─ PAM-T2-HelpDesk ────── Help desk limited access
   │  └─ OU=Accounts
   │
   ├─ OU=ServiceAccounts ───────── CyberArk Service Accounts
   │  ├─ svc-CyberArkIdentity ──── AD connector
   │  ├─ svc-CyberArkPSM ───────── PSM service
   │  ├─ svc-CyberArkScan ──────── Discovery
   │  └─ svc-CyberArkReconcile ─── Reconciliation
   │
   ├─ OU=AdminGroups ───────────── CyberArk Administrative Groups
   │  ├─ PAM-Vault-Admins ──────── Vault administrators
   │  ├─ PAM-Safe-Managers ─────── Safe management
   │  ├─ PAM-Auditors ──────────── Compliance team
   │  ├─ PAM-Users ─────────────── Standard users
   │  └─ PAM-Approvers ─────────── Access approvers
   │
   └─ OU=Users ─────────────────── Lab Team Accounts
      ├─ OU=Active ─────────────── Current members
      └─ OU=Disabled ───────────── Departed members
```

### 4.2 Security Groups Reference

| Group Name | Tier | Purpose | Safe Access |
|------------|------|---------|-------------|
| PAM-T0-Admins | 0 | Full DC access | T0 Safes: Full |
| PAM-T0-Operators | 0 | Limited DC operations | T0 Safes: Use |
| PAM-T0-Auditors | 0 | Read-only DC access | T0 Safes: List |
| PAM-T1-Admins | 1 | Server administration | T1 Safes: Full |
| PAM-T1-WindowsAdmins | 1 | Windows servers | T1-WIN: Use/Retrieve |
| PAM-T1-LinuxAdmins | 1 | Linux servers | T1-LNX: Use/Retrieve |
| PAM-T1-DatabaseAdmins | 1 | Database access | T1-DB: Use/Retrieve |
| PAM-T1-Operators | 1 | Limited server access | T1 Safes: Use |
| PAM-T2-Admins | 2 | Workstation admin | T2 Safes: Full |
| PAM-T2-HelpDesk | 2 | Help desk access | T2 Safes: Use |
| PAM-Vault-Admins | Admin | Vault administration | All: Full |
| PAM-Safe-Managers | Admin | Safe management | Assigned: Full |
| PAM-Auditors | Admin | Compliance/Audit | All: List/Audit |
| PAM-Users | Admin | Standard access | Assigned: Use |
| PAM-Approvers | Admin | Request approval | Assigned: Approve |

### 4.3 HOW-TO: Create TIER Model Structure

> **Script:** [lab_pcloud_scripts.md - Section 2](lab_pcloud_scripts.md#2-tier-model-structure)

```powershell
# Run on DC01 as Domain Administrator
.\Create-TierStructure.ps1 -DomainDN "DC=indramind,DC=cyblab"
```

### 4.4 Verification Checklist

| Item | Verification Command | Expected | Status |
|------|---------------------|----------|--------|
| PAM OU exists | `Get-ADOrganizationalUnit -Filter "Name -eq 'PAM'"` | Returns OU | ☐ |
| All Tier OUs | `Get-ADOrganizationalUnit -SearchBase "OU=PAM,DC=indramind,DC=cyblab"` | 12+ OUs | ☐ |
| Security groups | `Get-ADGroup -SearchBase "OU=PAM,DC=indramind,DC=cyblab"` | 15+ groups | ☐ |
| Service accounts | `Get-ADUser -SearchBase "OU=ServiceAccounts,OU=PAM,DC=indramind,DC=cyblab"` | 4+ accounts | ☐ |

---

## 5. User Management

> **📚 References:**
> - [CyberArk Identity User Management](https://docs.cyberark.com/identity/latest/en/content/admin/user-management.htm)

### 5.1 Account Lifecycle Workflow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          USER ACCOUNT LIFECYCLE                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐       │
│  │   EXPORT    │───►│   AUDIT     │───►│ CATEGORIZE  │───►│   ACTION    │       │
│  │   Users     │    │   Users     │    │   Users     │    │   Execute   │       │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘       │
│                                                                                 │
│  Categories:                                                                    │
│  ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐              │
│  │ ACTIVE            │ │ INACTIVE          │ │ DEPARTED          │              │
│  │                   │ │                   │ │                   │              │
│  │ • Update email    │ │ • Verify status   │ │ • Disable account │              │
│  │ • Verify groups   │ │ • Contact owner   │ │ • Move to Disabled│              │
│  │ • Create AD acct  │ │ • Decision: Keep/ │ │ • Document change │              │
│  │                   │ │   Disable         │ │                   │              │
│  └───────────────────┘ └───────────────────┘ └───────────────────┘              │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 HOW-TO: User Account Cleanup

#### Step 1: Export Users
```
CyberArk Identity Admin → Users → All Users → Export → CSV
Save as: identity_users_export.csv
```

#### Step 2: Categorize Users

| Username | Email | Status | Action | Notes |
|----------|-------|--------|--------|-------|
| user1 | old@email | Active | UPDATE | Update email |
| yago | yago@old | Departed | DISABLE | No longer on team |
| olmedo | olmedo@old | Departed | DISABLE | No longer on team |
| ana | ana@old | Departed | DISABLE | No longer on team |

#### Step 3: Execute Changes

> **Scripts:** [lab_pcloud_scripts.md - Section 3](lab_pcloud_scripts.md#3-user-management)

- Create domain accounts: Section 3.2
- Disable departed users: Section 3.3

---

## 6. Multi-Factor Authentication

> **📚 References:**
> - [CyberArk Identity MFA](https://docs.cyberark.com/identity/latest/en/content/mfa/mfa.htm)
> - [Authentication Policies](https://docs.cyberark.com/identity/latest/en/content/policies/policies.htm)

### 6.1 MFA Policy Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           MFA POLICY HIERARCHY                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  PRIORITY 50 (HIGHER)                                                           │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │  POLICY: LAB-2FA-HighSecurity                                             │  │
│  │  ─────────────────────────────                                            │  │
│  │  Target:   PAM-Vault-Admins, PAM-T0-Admins                                │  │
│  │  Methods:  Mobile Authenticator ONLY                                      │  │
│  │  Options:  Push Required, Biometric Required                              │  │
│  │  Session:  4 hours maximum                                                │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  PRIORITY 100 (DEFAULT)                                                         │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │  POLICY: LAB-2FA-Standard                                                 │  │
│  │  ────────────────────────                                                 │  │
│  │  Target:   PAM-Users                                                      │  │
│  │  Methods:  Email OTP, SMS OTP, Mobile Authenticator                       │  │
│  │  Options:  OTP: 6 digits, 5 min validity, 3 attempts                      │  │
│  │  Session:  Standard duration                                              │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Policy Configuration Reference

| Setting | LAB-2FA-Standard | LAB-2FA-HighSecurity |
|---------|------------------|----------------------|
| **Priority** | 100 | 50 (higher) |
| **Target Groups** | PAM-Users | PAM-Vault-Admins, PAM-T0-Admins |
| **Email OTP** | ✅ Enabled | ❌ Disabled |
| **SMS OTP** | ✅ Enabled | ❌ Disabled |
| **Mobile App** | ✅ Enabled | ✅ Required |
| **OTP Length** | 6 digits | N/A |
| **OTP Validity** | 5 minutes | N/A |
| **Max Attempts** | 3 | N/A |
| **Lockout Time** | 15 minutes | N/A |
| **Push Notification** | Optional | Required |
| **Biometric** | Optional | Required |
| **Session Duration** | Standard | 4 hours max |

### 6.3 HOW-TO: Create MFA Policies

#### Navigate to Policies
```
CyberArk Identity Admin → Access → Policies → Authentication Policies
```

#### Create LAB-2FA-Standard
1. Click "Add Policy Set"
2. Configure per table above
3. Assign to PAM-Users group
4. Save

#### Create LAB-2FA-HighSecurity
1. Click "Add Policy Set"
2. Configure per table above
3. Assign to PAM-Vault-Admins, PAM-T0-Admins
4. Save

### 6.4 Verification Checklist

| Test | Expected Result | Status |
|------|-----------------|--------|
| Login as PAM-Users member | Email/SMS/App options shown | ☐ |
| Login as PAM-Vault-Admins member | ONLY Mobile App option | ☐ |
| Email OTP received | 6-digit code, expires 5 min | ☐ |
| Mobile push works | Notification received, approve | ☐ |

---

## 7. Mobile App Configuration

> **📚 References:**
> - [CyberArk Identity Mobile Authenticator](https://docs.cyberark.com/identity/latest/en/content/mfa/mobile-authenticator.htm)
> - [iOS App](https://apps.apple.com/app/cyberark-identity/id1527456686)
> - [Android App](https://play.google.com/store/apps/details?id=com.cyberark.identity)

### 7.1 Mobile Authenticator Settings

| Setting | Value |
|---------|-------|
| Enabled | Yes |
| Allow Push | Yes |
| Allow Offline OTP | Yes |
| Biometric Option | Enabled |
| Max Devices per User | 2 |
| Enrollment Link Validity | 24 hours |

### 7.2 User Enrollment Process

1. Download CyberArk Identity app (iOS/Android)
2. Login to portal: `https://[tenant].id.cyberark.cloud`
3. Navigate: User Menu → Security Settings → MFA Devices
4. Click "Add Device" → "Mobile Authenticator"
5. Scan QR code with app
6. Approve test push notification

---

## 8. High Availability Infrastructure

> **📚 References:**
> - [CyberArk PSM Overview](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/psm-overview.htm)
> - [PSM High Availability](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/psm-ha.htm)
> - [SIA Overview](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/sia-overview.htm)

### 8.1 HA Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         PSM + SIA HIGH AVAILABILITY                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│                        ┌─────────────────────────────┐                          │
│                        │    PRIVILEGE CLOUD SAAS     │                          │
│                        └──────────────┬──────────────┘                          │
│                                       │                                         │
│                                       │ HTTPS/443                               │
│                                       ▼                                         │
│                        ┌─────────────────────────────┐                          │
│                        │  DNS: psm.indramind.cyblab  │                          │
│                        │      (Round Robin)          │                          │
│                        └──────────────┬──────────────┘                          │
│                           ┌───────────┴───────────┐                             │
│                           ▼                       ▼                             │
│            ┌──────────────────────┐   ┌──────────────────────┐                  │
│            │     CONNECTOR01      │   │     CONNECTOR02      │                  │
│            │     10.10.3.8        │   │     10.10.3.70       │                  │
│            │                      │   │                      │                  │
│            │  ┌────────────────┐  │   │  ┌────────────────┐  │                  │
│            │  │ PSM Component  │  │   │  │ PSM Component  │  │                  │
│            │  │ (Connector01-  │  │   │  │ (Connector02-  │  │                  │
│            │  │  PSM)          │  │   │  │  PSM)          │  │                  │
│            │  └────────────────┘  │   │  └────────────────┘  │                  │
│            │  ┌────────────────┐  │   │  ┌────────────────┐  │                  │
│            │  │ SIA Connector  │  │   │  │ SIA Connector  │  │                  │
│            │  │ (SIA-          │  │   │  │ (SIA-          │  │                  │
│            │  │  Connector01)  │  │   │  │  Connector02)  │  │                  │
│            │  └────────────────┘  │   │  └────────────────┘  │                  │
│            └──────────────────────┘   └──────────────────────┘                  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Server Specifications

| Specification | Connector01 | Connector02 |
|---------------|-------------|-------------|
| **Hostname** | Connector01 | Connector02 |
| **IP Address** | 10.10.3.8 | 10.10.3.70 |
| **OS** | Windows Server 2022 | Windows Server 2022 |
| **vCPU** | 8 | 8 |
| **RAM** | 16 GB | 16 GB |
| **Disk (OS)** | 120 GB | 120 GB |
| **Disk (Recordings)** | 500 GB | 500 GB |
| **Domain** | indramind.cyblab | indramind.cyblab |
| **PSM ID** | Connector01-PSM | Connector02-PSM |
| **SIA ID** | SIA-Connector01 | SIA-Connector02 |

### 8.3 DNS Configuration

| Record Type | Name | Value | TTL |
|-------------|------|-------|-----|
| A | psm.indramind.cyblab | 10.10.3.8 | 300 |
| A | psm.indramind.cyblab | 10.10.3.70 | 300 |
| A | connector01.indramind.cyblab | 10.10.3.8 | 3600 |
| A | connector02.indramind.cyblab | 10.10.3.70 | 3600 |

### 8.4 HOW-TO: Deploy PSM + SIA

> **Scripts:** [lab_pcloud_scripts.md - Section 4](lab_pcloud_scripts.md#4-psm-server-preparation)

#### Prerequisites
- Windows Features installed (RDS-RD-Server)
- Domain joined
- Cloud connectivity verified
- Service account created (svc-CyberArkPSM)

#### Installation Steps
1. Download connector from Privilege Cloud portal
2. Run `PrivilegeCloudConnector.exe` as Administrator
3. Select: PSM + SIA components
4. Configure PSM Server ID and service account
5. Configure SIA Connector Name
6. Complete installation
7. Repeat on second server

### 8.5 Verification Checklist

| Check | Connector01 | Connector02 |
|-------|-------------|-------------|
| PSM Service Running | ☐ | ☐ |
| SIA Service Running | ☐ | ☐ |
| Portal Status: Connected | ☐ | ☐ |
| DNS Round Robin Works | ☐ | ☐ |
| Failover Test Passed | ☐ | ☐ |

---

## 9. Naming Conventions

> **📚 References:**
> - [CyberArk Safe Management](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/safes.htm)
> - [Platform Management](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/platforms-overview.htm)

### 9.1 Safe Naming Convention

**Format:** `{ENV}-{TIER}-{TYPE}-{OWNER}`

#### Environment Codes

| Code | Environment |
|------|-------------|
| LAB | Lab/Test |
| DEV | Development |
| STG | Staging |
| PRD | Production |

#### Tier Codes

| Code | Tier | Description |
|------|------|-------------|
| T0 | Tier 0 | Domain Controllers |
| T1 | Tier 1 | Enterprise Servers |
| T2 | Tier 2 | Workstations |
| SVC | Service | Service Accounts |
| APP | Application | Application-specific |

#### Type Codes

| Code | Type |
|------|------|
| WIN | Windows |
| LNX | Linux |
| DB | Database |
| NET | Network |
| CLD | Cloud |
| SSH | SSH Keys |

#### Examples

| Safe Name | Description |
|-----------|-------------|
| LAB-T0-WIN-DomainAdmins | Lab Tier 0 Domain Admin accounts |
| LAB-T1-WIN-Servers | Lab Tier 1 Windows server admins |
| LAB-T1-LNX-Servers | Lab Tier 1 Linux root accounts |
| LAB-T1-DB-Oracle | Lab Tier 1 Oracle DBA accounts |
| LAB-SVC-Automation | Lab Service accounts |
| PRD-T0-WIN-DomainAdmins | Production Domain Admin accounts |

### 9.2 Account Naming Convention

**Format:** `{PLATFORM}_{TYPE}_{TARGET}_{USERNAME}`

| Account Name | Description |
|--------------|-------------|
| WIN_DA_LAB_Administrator | Windows Domain Admin on LAB |
| WIN_LA_SRV01_Administrator | Windows Local Admin on SRV01 |
| LNX_ROOT_WEB01_root | Linux root on WEB01 |
| ORA_DBA_PROD_sys | Oracle DBA sys on PROD |

---

## 10. OKTA Integration

> **📚 References:**
> - [CyberArk SAML External IdP](https://docs.cyberark.com/identity/latest/en/content/integrations/saml/saml-external-idp.htm)
> - [Okta SAML Configuration](https://help.okta.com/en-us/content/topics/apps/apps_app_integration_wizard_saml.htm)

### 10.1 SAML Federation Design

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           OKTA SAML INTEGRATION                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   User                                                                          │
│    │                                                                            │
│    │ (1) Access Request                                                         │
│    ▼                                                                            │
│   CyberArk Identity (SP) ◄────────────────────────► OKTA (IdP)                  │
│    │                        (2) SAML AuthnRequest                               │
│    │                        (3) SAML Response                                   │
│    │                                                                            │
│    │ (4) Session Created                                                        │
│    ▼                                                                            │
│   Privilege Cloud                                                               │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 10.2 SAML Configuration

| Setting | Value |
|---------|-------|
| **IdP Name** | OKTA |
| **Single Sign-on URL** | https://[tenant].id.cyberark.cloud/saml/consume |
| **Audience URI (SP Entity ID)** | https://[tenant].id.cyberark.cloud |
| **Name ID Format** | EmailAddress |
| **Application Username** | Email |

### 10.3 Attribute Mapping

| Attribute | OKTA Value |
|-----------|------------|
| firstName | user.firstName |
| lastName | user.lastName |
| email | user.email |
| groups | appuser.groups |

### 10.4 Group Mapping

| OKTA Group | CyberArk Group |
|------------|----------------|
| CyberArk-Admins | PAM-Vault-Admins |
| CyberArk-Users | PAM-Users |
| CyberArk-Auditors | PAM-Auditors |

---

## 11. Implementation Roadmap

### 11.1 Phase Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         IMPLEMENTATION ROADMAP                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  PHASE 1: FOUNDATION                                                            │
│  ════════════════════                                                           │
│  ☐ Validate AD (DC01)                                                           │
│  ☐ Run Create-TierStructure.ps1                                                 │
│  ☐ Create service accounts                                                      │
│  ☐ Configure AD as IdP in CyberArk Identity                                     │
│                                                                                 │
│  PHASE 2: USER MANAGEMENT                                                       │
│  ════════════════════════                                                       │
│  ☐ Export and audit current users                                               │
│  ☐ Update email addresses                                                       │
│  ☐ Create domain accounts                                                       │
│  ☐ Disable departed users                                                       │
│                                                                                 │
│  PHASE 3: MFA & INFRASTRUCTURE                                                  │
│  ══════════════════════════════                                                 │
│  ☐ Create LAB-2FA-Standard policy                                               │
│  ☐ Create LAB-2FA-HighSecurity policy                                           │
│  ☐ Configure Email/Phone OTP                                                    │
│  ☐ Configure Mobile App                                                         │
│  ☐ Deploy PSM + SIA on Connector01                                              │
│  ☐ Deploy PSM + SIA on Connector02                                              │
│  ☐ Configure DNS round robin                                                    │
│  ☐ Test HA failover                                                             │
│                                                                                 │
│  PHASE 4: STANDARDS & OKTA                                                      │
│  ══════════════════════════                                                     │
│  ☐ Implement naming conventions                                                 │
│  ☐ Document standards for team                                                  │
│  ☐ Recover OKTA Admin access                                                    │
│  ☐ Configure OKTA SAML                                                          │
│  ☐ Test with pilot users                                                        │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 12. Validation Checklists

### 12.1 Complete System Validation

| Category | Item | Status |
|----------|------|--------|
| **AD Integration** | | |
| | AD connector service account exists | ☐ |
| | TIER model OU structure created | ☐ |
| | All security groups created | ☐ |
| | Service accounts created | ☐ |
| | AD directory connected in Identity | ☐ |
| | User sync working | ☐ |
| | AD authentication tested | ☐ |
| **User Management** | | |
| | Team accounts created in AD | ☐ |
| | Users in correct OUs | ☐ |
| | Group memberships correct | ☐ |
| | Departed users disabled | ☐ |
| **MFA** | | |
| | Email OTP configured | ☐ |
| | SMS OTP configured | ☐ |
| | Mobile App configured | ☐ |
| | Standard policy created | ☐ |
| | High security policy created | ☐ |
| | Policy priority correct | ☐ |
| **PSM HA** | | |
| | Connector01 PSM running | ☐ |
| | Connector01 SIA running | ☐ |
| | Connector02 PSM running | ☐ |
| | Connector02 SIA running | ☐ |
| | DNS round robin configured | ☐ |
| | Failover tested | ☐ |
| **Naming** | | |
| | Safe naming documented | ☐ |
| | Initial safes created | ☐ |
| | Permissions configured | ☐ |

---

## 13. Glossary

| Term | Definition |
|------|------------|
| **AD DS** | Active Directory Domain Services |
| **CPM** | Central Policy Manager (cloud-managed in Privilege Cloud) |
| **IdP** | Identity Provider |
| **LDAP/LDAPS** | Lightweight Directory Access Protocol (Secure) |
| **MFA** | Multi-Factor Authentication |
| **OTP** | One-Time Password |
| **PAM** | Privileged Access Management |
| **PSM** | Privileged Session Manager |
| **PVWA** | Password Vault Web Access |
| **SAML** | Security Assertion Markup Language |
| **SIA** | Secure Infrastructure Access |
| **SP** | Service Provider |
| **TIER Model** | Microsoft's privileged access security model |

---

## Document Footer

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║  Document: LAB-ARCH-001 | Version: 3.0 | Status: Production Ready               ║
║  Classification: Internal Use | Owner: Lab Team | Updated: January 2026         ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

**Related Documents:**
- [INSTALLATION.md](INSTALLATION.md) - Complete installation guide
- [lab_pcloud_scripts.md](lab_pcloud_scripts.md) - PowerShell scripts reference
- [README.md](README.md) - Repository overview
