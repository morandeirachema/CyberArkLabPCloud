# Lab Privilege Cloud - Technical Architecture & Implementation Guide

## Document Control

| Version | Date | Author | Description |
|---------|------|--------|-------------|
| 1.0 | January 2026 | Lab Team | Initial architecture document |
| 2.0 | January 2026 | Lab Team | Added detailed how-to implementation steps |

---

## 📚 External References & Official Documentation

### CyberArk Documentation Portal

| Component | Documentation Link |
|-----------|-------------------|
| **Privilege Cloud Overview** | [docs.cyberark.com/privilege-cloud](https://docs.cyberark.com/privilege-cloud/latest/en/content/landing-pages/lprivcloud.htm) |
| **CyberArk Identity** | [docs.cyberark.com/identity](https://docs.cyberark.com/identity/latest/en/content/landing-pages/lpidentity.htm) |
| **System Requirements** | [Privilege Cloud Requirements](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/system-requirements.htm) |
| **Network Requirements** | [Network Configuration](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/network-requirements.htm) |
| **AD Integration** | [Identity AD Integration](https://docs.cyberark.com/identity/latest/en/content/integrations/ad/ad-integration.htm) |
| **MFA Configuration** | [Multi-Factor Authentication](https://docs.cyberark.com/identity/latest/en/content/mfa/mfa.htm) |
| **PSM Overview** | [Privileged Session Manager](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/psm-overview.htm) |
| **PSM Installation** | [Install Connector](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/install-connector.htm) |
| **PSM High Availability** | [PSM HA Configuration](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/psm-ha.htm) |
| **SIA Overview** | [Secure Infrastructure Access](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/sia-overview.htm) |
| **Safe Management** | [Managing Safes](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/safes.htm) |
| **Platform Management** | [Platforms Overview](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/platforms-overview.htm) |
| **SAML Federation** | [External IdP Integration](https://docs.cyberark.com/identity/latest/en/content/integrations/saml/saml-external-idp.htm) |

### Microsoft Documentation

| Topic | Documentation Link |
|-------|-------------------|
| **Active Directory DS** | [AD DS Overview](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview) |
| **TIER Model / PAW** | [Privileged Access Model](https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-access-model) |
| **Securing Privileged Access** | [Security Best Practices](https://learn.microsoft.com/en-us/security/privileged-access-workstations/overview) |
| **LDAP/LDAPS** | [LDAP Overview](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ldap/lightweight-directory-access-protocol-ldap-api) |
| **DNS Server** | [DNS Documentation](https://learn.microsoft.com/en-us/windows-server/networking/dns/dns-top) |
| **Remote Desktop Services** | [RDS Overview](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/welcome-to-rds) |
| **PowerShell AD Module** | [ActiveDirectory Cmdlets](https://learn.microsoft.com/en-us/powershell/module/activedirectory/) |

### Third-Party Integration

| Platform | Documentation Link |
|----------|-------------------|
| **Okta SAML** | [Okta SAML App Integration](https://help.okta.com/en-us/content/topics/apps/apps_app_integration_wizard_saml.htm) |
| **Okta Developer** | [SAML 2.0 Overview](https://developer.okta.com/docs/concepts/saml/) |
| **SAML Specification** | [OASIS SAML 2.0](http://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html) |

### CyberArk Community & Support

| Resource | Link |
|----------|------|
| **CyberArk Community** | [cyberark-customers.force.com](https://cyberark-customers.force.com/s/) |
| **CyberArk Marketplace** | [Platform & Integration Packs](https://cyberark-customers.force.com/mplace/s/#--Background) |
| **CyberArk Support** | [support.cyberark.com](https://support.cyberark.com) |
| **psPAS PowerShell Module** | [github.com/pspete/psPAS](https://github.com/pspete/psPAS) |
| **CyberArk GitHub** | [github.com/cyberark](https://github.com/cyberark) |

---

## 1. Executive Summary

This document defines the technical architecture for implementing the CyberArk Privilege Cloud lab environment. The architecture addresses eight key areas:

1. **Active Directory Integration** - AD as Identity Provider
2. **Directory Structure** - TIER model implementation for users and groups
3. **User Management** - Account cleanup and homogenization
4. **Multi-Factor Authentication** - PolicySets for 2FA (email/phone)
5. **Mobile App Configuration** - CyberArk Identity mobile enrollment
6. **High Availability Infrastructure** - PSM HA + SIA HA
7. **Naming Conventions** - Standardized Safe and Platform nomenclature
8. **OKTA Integration** - SAML federation for select user groups

---

## 2. Requirements Mapping

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      LAB REQUIREMENTS MAPPING                            │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  REQ  REQUIREMENT                        COMPONENT          PRIORITY     │
│  ───  ───────────                        ─────────          ────────     │
│                                                                          │
│  #1   AD Integration as IdP              CyberArk Identity  HIGH         │
│  #2   TIER Model Directory Structure     Active Directory   HIGH         │
│  #3   User Account Cleanup               Identity Admin     MEDIUM       │
│  #4   2FA PolicySets (Email/Phone)       CyberArk Identity  HIGH         │
│  #5   Mobile App Configuration           CyberArk Identity  MEDIUM       │
│  #6   PSM HA + SIA HA Infrastructure     Privilege Cloud    HIGH         │
│  #7   Safe/Platform Naming Convention    Vault Admin        HIGH         │
│  #8   OKTA Integration                   Identity/SAML      LOW          │
│                                                                          │
│  Legend:                                                                 │
│  HIGH   = Phase 1-2 (Immediate)                                          │
│  MEDIUM = Phase 3 (Post-deployment)                                      │
│  LOW    = Phase 4 (Post-holidays, requires Admin access recovery)        │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Identity Architecture

### 3.1 Active Directory as Identity Provider

```
┌──────────────────────────────────────────────────────────────────────────┐
│                   IDENTITY PROVIDER ARCHITECTURE                         │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                     CYBERARK IDENTITY                              │  │
│  │                   (company.id.cyberark.cloud)                      │  │
│  │                                                                    │  │
│  │  ┌──────────────────────────────────────────────────────────────┐  │  │
│  │  │                    IDENTITY SOURCES                          │  │  │
│  │  │                                                              │  │  │
│  │  │  ┌──────────────────┐ ┌────────────────┐ ┌────────────────┐  │  │  │
│  │  │  │     PRIMARY      │ │   SECONDARY    │ │     FUTURE     │  │  │  │
│  │  │  │                  │ │                │ │                │  │  │  │
│  │  │  │  ┌────────────┐  │ │  ┌──────────┐  │ │  ┌──────────┐  │  │  │  │
│  │  │  │  │     AD     │  │ │  │  LOCAL   │  │ │  │   OKTA   │  │  │  │  │
│  │  │  │  │ indramind  │  │ │  │  USERS   │  │ │  │   SAML   │  │  │  │  │
│  │  │  │  │  .cyblab   │  │ │  │          │  │ │  │          │  │  │  │  │
│  │  │  │  └────────────┘  │ │  └──────────┘  │ │  └──────────┘  │  │  │  │
│  │  │  │                  │ │                │ │                │  │  │  │
│  │  │  │    LDAP/LDAPS    │ │   Cloud Dir    │ │   Federation   │  │  │  │
│  │  │  └──────────────────┘ └────────────────┘ └────────────────┘  │  │  │
│  │  │                                                              │  │  │
│  │  └──────────────────────────────────────────────────────────────┘  │  │
│  │                                                                    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                    │                                     │
│                                    │ LDAP/LDAPS (389/636)                │
│                                    ▼                                     │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                      ACTIVE DIRECTORY                              │  │
│  │                   DC01 (indramind.cyblab)                          │  │
│  │                                                                    │  │
│  │  ┌──────────────────────────────────────────────────────────────┐  │  │
│  │  │                   AD CONNECTOR CONFIG                        │  │  │
│  │  │                                                              │  │  │
│  │  │  Server:        DC01.indramind.cyblab (10.10.3.151)          │  │  │
│  │  │  Port:          636 (LDAPS) / 389 (LDAP)                     │  │  │
│  │  │  Base DN:       DC=indramind,DC=cyblab                       │  │  │
│  │  │  User DN:       OU=PAM,DC=indramind,DC=cyblab                │  │  │
│  │  │  Bind Account:  svc-CyberArkIdentity@indramind.cyblab        │  │  │
│  │  │  Sync Interval: 15 minutes                                   │  │  │
│  │  │                                                              │  │  │
│  │  └──────────────────────────────────────────────────────────────┘  │  │
│  │                                                                    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2 AD Connector Service Account

| Attribute | Value |
|-----------|-------|
| **Account Name** | svc-CyberArkIdentity |
| **Location** | OU=ServiceAccounts,OU=PAM,DC=indramind,DC=cyblab |
| **Purpose** | CyberArk Identity AD connector |
| **Permissions** | Read-only access to user/group objects |
| **Password Policy** | 128-character, non-expiring, managed by CyberArk |

### 3.3 HOW-TO: Configure AD as Identity Provider

#### Step 1: Create the AD Connector Service Account

Run on DC01 (Domain Controller).

> **Script:** [lab_pcloud_scripts.md - AD Connector Service Account](lab_pcloud_scripts.md#1-ad-connector-service-account)

#### Step 2: Configure AD Connector in CyberArk Identity

1. **Login to CyberArk Identity Admin Portal**
   ```
   URL: https://[tenant].id.cyberark.cloud/admin
   ```

2. **Navigate to Directory Services**
   ```
   Settings → Core Services → Directory Services → Add Directory
   ```

3. **Configure AD Connection**
   ```
   Directory Type:     Active Directory
   Directory Name:     LAB-AD

   Connection Settings:
   ├── Server Address:   DC01.indramind.cyblab
   ├── Port:             636 (LDAPS) or 389 (LDAP)
   ├── Use SSL:          Yes (recommended)
   ├── Base DN:          DC=indramind,DC=cyblab
   └── Bind DN:          CN=svc-CyberArkIdentity,OU=ServiceAccounts,OU=PAM,DC=indramind,DC=cyblab

   Sync Settings:
   ├── User Container:   OU=PAM,DC=indramind,DC=cyblab
   ├── Group Container:  OU=PAM,DC=indramind,DC=cyblab
   ├── Sync Interval:    15 minutes
   └── Sync Nested:      Yes
   ```

4. **Test Connection**
   ```
   Click "Test Connection" → Verify "Connection Successful"
   ```

5. **Configure User Sync Filter (Optional)**
   ```
   LDAP Filter: (&(objectClass=user)(objectCategory=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
   ```
   This filter syncs only enabled user accounts.

6. **Save and Start Initial Sync**
   ```
   Click "Save" → Click "Sync Now"
   ```

#### Step 3: Verify AD Integration

1. **Check Sync Status**
   ```
   Settings → Core Services → Directory Services → LAB-AD → Sync Status
   Verify: "Last Sync: Successful" with user/group counts
   ```

2. **Verify Users Imported**
   ```
   Users → All Users → Filter by Directory: LAB-AD
   Confirm lab users appear in the list
   ```

3. **Test AD Authentication**
   ```
   Open incognito browser → Navigate to https://[tenant].id.cyberark.cloud
   Login with: username@indramind.cyblab + AD password
   Verify: Successful authentication
   ```

---

## 4. Directory Structure (TIER Model)

### 4.1 Complete OU Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                  TIER MODEL - DIRECTORY STRUCTURE                        │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  DC=indramind,DC=cyblab                                                         │
│  │                                                                       │
│  └─ OU=PAM (CyberArk Privileged Access Management)                       │
│     │                                                                    │
│     ├─ OU=Tier0 (Domain Controllers, Forest Admins)                      │
│     │  ├─ OU=Groups                                                      │
│     │  │  ├─ PAM-T0-Admins       Full DC access                          │
│     │  │  ├─ PAM-T0-Operators    Limited DC operations                   │
│     │  │  └─ PAM-T0-Auditors     Read-only DC access                     │
│     │  └─ OU=Accounts                                                    │
│     │                                                                    │
│     ├─ OU=Tier1 (Enterprise Servers)                                     │
│     │  ├─ OU=Groups                                                      │
│     │  │  ├─ PAM-T1-Admins          General server admins                │
│     │  │  ├─ PAM-T1-WindowsAdmins   Windows server specialists           │
│     │  │  ├─ PAM-T1-LinuxAdmins     Linux/Unix administrators            │
│     │  │  ├─ PAM-T1-DatabaseAdmins  DBA team                             │
│     │  │  └─ PAM-T1-Operators       Limited server access                │
│     │  └─ OU=Accounts                                                    │
│     │                                                                    │
│     ├─ OU=Tier2 (Workstations)                                           │
│     │  ├─ OU=Groups                                                      │
│     │  │  ├─ PAM-T2-Admins      Workstation administrators               │
│     │  │  └─ PAM-T2-HelpDesk    Help desk limited access                 │
│     │  └─ OU=Accounts                                                    │
│     │                                                                    │
│     ├─ OU=ServiceAccounts                                                │
│     │  ├─ svc-CyberArkCPM        CPM password management                 │
│     │  ├─ svc-CyberArkPSM        PSM session management                  │
│     │  ├─ svc-CyberArkScan       Account discovery/scanning              │
│     │  ├─ svc-CyberArkIdentity   Identity AD connector                   │
│     │  └─ svc-CyberArkReconcile  Reconciliation account                  │
│     │                                                                    │
│     ├─ OU=AdminGroups                                                    │
│     │  ├─ PAM-Vault-Admins    Vault administrators                       │
│     │  ├─ PAM-Safe-Managers   Safe creation and management               │
│     │  ├─ PAM-Auditors        Compliance and audit team                  │
│     │  ├─ PAM-Users           Standard privileged users                  │
│     │  └─ PAM-Approvers       Access request approvers                   │
│     │                                                                    │
│     └─ OU=Users                                                          │
│        ├─ OU=Active     Current team members                             │
│        └─ OU=Disabled   Former team members                              │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.2 HOW-TO: Create TIER Model Structure

> **Script:** [lab_pcloud_scripts.md - TIER Model Structure](lab_pcloud_scripts.md#2-tier-model-structure)

1. Save as `Create-TierStructure.ps1` on DC01
2. Open PowerShell as Administrator
3. Run: `.\Create-TierStructure.ps1 -DomainDN "DC=indramind,DC=cyblab"`

---

## 5. User Management Architecture

### 5.1 Account Cleanup Process

```
┌──────────────────────────────────────────────────────────────────────────┐
│                   USER ACCOUNT CLEANUP WORKFLOW                          │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  PHASE 1: INVENTORY                                                      │
│  ══════════════════                                                      │
│                                                                          │
│  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐      │
│  │   LOCAL USERS    │   │   DOMAIN USERS   │   │    COMPARISON    │      │
│  │   (Identity)     │──►│   (AD)           │──►│    ANALYSIS      │      │
│  └──────────────────┘   └──────────────────┘   └──────────────────┘      │
│                                                         │                │
│                                                         ▼                │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │  CATEGORIZATION                                                  │    │
│  │                                                                  │    │
│  │  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐  │    │
│  │  │   ACTIVE USERS   │ │  INACTIVE USERS  │ │   DELETE USERS   │  │    │
│  │  │                  │ │                  │ │                  │  │    │
│  │  │  Current team    │ │  Require         │ │  Yago            │  │    │
│  │  │  members with    │ │  validation      │ │  Olmedo          │  │    │
│  │  │  valid license   │ │  before deletion │ │  Ana             │  │    │
│  │  │                  │ │                  │ │  [Others]        │  │    │
│  │  └──────────────────┘ └──────────────────┘ └──────────────────┘  │    │
│  │                                                                  │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 5.2 HOW-TO: User Account Cleanup

#### Step 1: Export Current Users from CyberArk Identity

1. **Login to CyberArk Identity Admin Portal**
   ```
   URL: https://[tenant].id.cyberark.cloud/admin
   ```

2. **Export User List**
   ```
   Users → All Users → Export → CSV
   ```

3. **Save as** `identity_users_export.csv`

#### Step 2: Audit and Categorize Users

Create a spreadsheet with columns:
```
| Username | Email | Status | Action | Notes |
|----------|-------|--------|--------|-------|
| user1    | old@  | Active | UPDATE | Update email to new domain |
| yago     | yago@ | Inactive | DELETE | No longer on team |
| olmedo   | olmed@| Inactive | DELETE | No longer on team |
| ana      | ana@  | Inactive | DELETE | No longer on team |
```

#### Step 3: Update Email Addresses in CyberArk Identity

For each active user:

1. **Navigate to User**
   ```
   Users → All Users → [Click Username]
   ```

2. **Update Email**
   ```
   Profile → Email → Change to: firstname.lastname@newdomain.com
   ```

3. **Save Changes**

**Bulk Update via API (Optional):**

> **Script:** [lab_pcloud_scripts.md - Bulk Update User Emails](lab_pcloud_scripts.md#31-bulk-update-user-emails-via-api)

#### Step 4: Create Domain Accounts for Active Users

> **Script:** [lab_pcloud_scripts.md - Create Domain Accounts](lab_pcloud_scripts.md#32-create-domain-accounts)

#### Step 5: Disable/Delete Departed Users

> **Script:** [lab_pcloud_scripts.md - Disable Departed Users](lab_pcloud_scripts.md#33-disable-departed-users)

#### Step 6: Document Changes

Create a change log entry:
```
Date: [Date]
Action: User Account Cleanup
Changes:
- Updated X users with new email domain
- Created X new domain accounts
- Disabled accounts: yago, olmedo, ana
- Accounts moved to OU=Disabled,OU=Users,OU=PAM
Performed by: [Admin Name]
```

---

## 6. Multi-Factor Authentication Architecture

### 6.1 PolicySet Structure

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     MFA POLICYSET ARCHITECTURE                           │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                      CYBERARK IDENTITY                             │  │
│  │                       MFA POLICIES                                 │  │
│  │                                                                    │  │
│  │  ┌──────────────────────────────────────────────────────────────┐  │  │
│  │  │           POLICYSET: LAB-2FA-Standard                        │  │  │
│  │  │                                                              │  │  │
│  │  │  Target:     All users in PAM-Users group                    │  │  │
│  │  │  Priority:   100 (Default)                                   │  │  │
│  │  │  MFA:        Email OTP / Phone OTP / Mobile App              │  │  │
│  │  │                                                              │  │  │
│  │  └──────────────────────────────────────────────────────────────┘  │  │
│  │                                                                    │  │
│  │  ┌──────────────────────────────────────────────────────────────┐  │  │
│  │  │           POLICYSET: LAB-2FA-HighSecurity                    │  │  │
│  │  │                                                              │  │  │
│  │  │  Target:     PAM-Vault-Admins, PAM-T0-Admins                 │  │  │
│  │  │  Priority:   50 (Higher priority)                            │  │  │
│  │  │  MFA:        Mobile App ONLY (No fallback)                   │  │  │
│  │  │                                                              │  │  │
│  │  └──────────────────────────────────────────────────────────────┘  │  │
│  │                                                                    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 6.2 HOW-TO: Create MFA PolicySets

#### Step 1: Access Policy Administration

1. **Login to CyberArk Identity Admin Portal**
   ```
   URL: https://[tenant].id.cyberark.cloud/admin
   ```

2. **Navigate to Policies**
   ```
   Access → Policies → Authentication Policies
   ```

#### Step 2: Create Standard 2FA Policy (LAB-2FA-Standard)

1. **Create New Policy**
   ```
   Click "Add Policy Set" → Enter Name: "LAB-2FA-Standard"
   ```

2. **Configure Policy Settings**
   ```
   General:
   ├── Name:        LAB-2FA-Standard
   ├── Description: Standard 2FA policy for all lab users
   ├── Status:      Active
   └── Priority:    100

   Policy Scope:
   ├── Apply to:    Specific Groups
   └── Groups:      PAM-Users

   Authentication Rules:
   ├── Primary Authentication:
   │   └── Method: Password (AD or Identity)
   │
   └── Secondary Authentication (MFA):
       ├── Required: Yes
       ├── Allowed Methods:
       │   ├── [x] Email OTP
       │   ├── [x] SMS OTP
       │   └── [x] Mobile Authenticator
       │
       └── Settings:
           ├── OTP Length:     6 digits
           ├── OTP Validity:   5 minutes
           ├── Max Attempts:   3
           └── Lockout Time:   15 minutes
   ```

3. **Configure Conditions (Optional)**
   ```
   Conditions:
   ├── Network:     Any
   ├── Device:      Any
   ├── Time:        Any
   └── Risk Level:  Any
   ```

4. **Save Policy**
   ```
   Click "Save"
   ```

#### Step 3: Create High Security Policy (LAB-2FA-HighSecurity)

1. **Create New Policy**
   ```
   Click "Add Policy Set" → Enter Name: "LAB-2FA-HighSecurity"
   ```

2. **Configure Policy Settings**
   ```
   General:
   ├── Name:        LAB-2FA-HighSecurity
   ├── Description: High security policy for vault and tier 0 admins
   ├── Status:      Active
   └── Priority:    50 (Higher priority than Standard)

   Policy Scope:
   ├── Apply to:    Specific Groups
   └── Groups:
       ├── PAM-Vault-Admins
       └── PAM-T0-Admins

   Authentication Rules:
   ├── Primary Authentication:
   │   └── Method: Password (AD or Identity)
   │
   └── Secondary Authentication (MFA):
       ├── Required: Yes
       ├── Allowed Methods:
       │   └── [x] Mobile Authenticator ONLY
       │
       └── Settings:
           ├── Push Notification:   Required
           ├── Biometric Unlock:    Required
           ├── Device Trust:        Enrolled devices only
           ├── Session Duration:    4 hours maximum
           └── Re-auth on Sensitive: Yes
   ```

3. **Save Policy**

#### Step 4: Configure Email OTP Settings

1. **Navigate to MFA Settings**
   ```
   Settings → Authentication → Email OTP
   ```

2. **Configure Email OTP**
   ```
   Email OTP Settings:
   ├── Enabled:         Yes
   ├── OTP Length:      6 digits
   ├── OTP Validity:    5 minutes
   ├── Sender Address:  noreply@cyberark.cloud
   ├── Subject Line:    Your CyberArk verification code
   └── Email Template:  Default
   ```

#### Step 5: Configure Phone OTP (SMS) Settings

1. **Navigate to SMS Settings**
   ```
   Settings → Authentication → SMS OTP
   ```

2. **Configure SMS OTP**
   ```
   SMS OTP Settings:
   ├── Enabled:         Yes
   ├── OTP Length:      6 digits
   ├── OTP Validity:    5 minutes
   ├── Message Format:  "Your CyberArk code is: {OTP}"
   └── International:   Enabled
   ```

#### Step 6: Verify Policy Order

1. **Check Policy Priority**
   ```
   Access → Policies → Authentication Policies

   Verify order:
   1. LAB-2FA-HighSecurity (Priority: 50) ← Higher priority
   2. LAB-2FA-Standard (Priority: 100)
   ```

2. **Test Policy Application**
   - Login as PAM-Users member → Should get Email/SMS/App options
   - Login as PAM-Vault-Admins member → Should ONLY get Mobile App option

---

## 7. Mobile App Configuration

### 7.1 HOW-TO: Configure Mobile App Enrollment

#### Step 1: Enable Mobile Authenticator

1. **Navigate to Authentication Settings**
   ```
   Settings → Authentication → Mobile Authenticator
   ```

2. **Configure Mobile App**
   ```
   Mobile Authenticator Settings:
   ├── Enabled:              Yes
   ├── App Name:             CyberArk Identity
   ├── Allow Push:           Yes
   ├── Allow Offline OTP:    Yes
   ├── Biometric Option:     Enabled
   └── Device Trust:         Optional (Required for High Security)
   ```

#### Step 2: Configure Self-Service Enrollment

1. **Navigate to Self-Service**
   ```
   Settings → Self-Service → MFA Enrollment
   ```

2. **Enable Self-Enrollment**
   ```
   Self-Service Enrollment:
   ├── Allow Mobile App Enrollment:    Yes
   ├── Require Admin Approval:         No (for Standard users)
   ├── Enrollment Link Validity:       24 hours
   └── Max Devices per User:           2
   ```

#### Step 3: User Enrollment Process

**For End Users:**

1. **Access User Portal**
   ```
   URL: https://[tenant].id.cyberark.cloud
   Login with username and password
   ```

2. **Navigate to Security Settings**
   ```
   User Menu (top right) → Security Settings → MFA Devices
   ```

3. **Add Mobile Device**
   ```
   Click "Add Device" → Select "Mobile Authenticator"
   ```

4. **Scan QR Code**
   ```
   1. Download CyberArk Identity app from App Store / Play Store
   2. Open app → Add Account → Scan QR Code
   3. Scan the QR code displayed on screen
   4. Verify with test push notification
   ```

5. **Complete Setup**
   ```
   Click "Verify" → Approve push notification on phone
   Device is now enrolled
   ```

#### Step 4: Admin-Initiated Enrollment

For users who need assistance:

1. **Navigate to User Management**
   ```
   Users → All Users → [Select User]
   ```

2. **Manage MFA Devices**
   ```
   Security → MFA Devices → Send Enrollment Link
   ```

3. **User Receives Email**
   ```
   User clicks link → Follows enrollment wizard
   ```

---

## 8. High Availability Infrastructure

### 8.1 PSM + SIA HA Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     PSM + SIA HIGH AVAILABILITY                          │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                       PRIVILEGE CLOUD                              │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                    │                                     │
│                                    │ HTTPS/443                           │
│                                    ▼                                     │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                  CONNECTOR ZONE (indramind.cyblab)                        │  │
│  │                                                                    │  │
│  │            ┌───────────────────────────────────┐                   │  │
│  │            │   DNS ROUND ROBIN (psm.indramind.cyblab) │                   │  │
│  │            └─────────────────┬─────────────────┘                   │  │
│  │                    ┌─────────┴─────────┐                           │  │
│  │                    ▼                   ▼                           │  │
│  │  ┌────────────────────────┐    ┌────────────────────────┐          │  │
│  │  │      Connector01       │    │      Connector02       │          │  │
│  │  │                        │    │                        │          │  │
│  │  │  ┌──────────────────┐  │    │  ┌──────────────────┐  │          │  │
│  │  │  │  PSM Component   │  │    │  │  PSM Component   │  │          │  │
│  │  │  └──────────────────┘  │    │  └──────────────────┘  │          │  │
│  │  │  ┌──────────────────┐  │    │  ┌──────────────────┐  │          │  │
│  │  │  │  SIA Connector   │  │    │  │  SIA Connector   │  │          │  │
│  │  │  └──────────────────┘  │    │  └──────────────────┘  │          │  │
│  │  │                        │    │                        │          │  │
│  │  └────────────────────────┘    └────────────────────────┘          │  │
│  │                                                                    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 8.2 HOW-TO: Deploy PSM + SIA in High Availability

#### Prerequisites

| Server | Requirements |
|--------|--------------|
| Connector01 | Windows Server 2022, 8 vCPU, 16GB RAM, 500GB disk, domain-joined |
| Connector02 | Windows Server 2022, 8 vCPU, 16GB RAM, 500GB disk, domain-joined |

#### Step 1: Prepare Servers

**On both Connector01 and Connector02:**

> **Script:** [lab_pcloud_scripts.md - Prepare Connector Servers](lab_pcloud_scripts.md#41-prepare-connector-servers)

#### Step 2: Install PSM on Connector01 (Primary)

1. **Download Connector Package**
   ```
   Login to Privilege Cloud Portal
   Administration → Connectors → Download Connector Package
   Save: PrivilegeCloudConnector.exe
   ```

2. **Run Installer**
   - Run `PrivilegeCloudConnector.exe` as Administrator

3. **Installation Wizard**
   ```
   Welcome → Next

   Select Components:
   ├── [x] Privileged Session Manager (PSM)
   └── [x] Secure Infrastructure Access (SIA)

   Connection Settings:
   ├── Tenant URL:     [your-tenant].privilegecloud.cyberark.cloud
   ├── Installer User: [admin username]
   └── Password:       [admin password]

   PSM Settings:
   ├── PSM Server Name:    Connector01
   ├── Service Account:    svc-CyberArkPSM@indramind.cyblab
   └── Recording Path:     D:\PSMRecordings (or appropriate path)

   SIA Settings:
   ├── Enable SIA:         Yes
   └── Connector Name:     SIA-Connector01

   Proceed with installation...
   ```

4. **Verify Installation**
   > **Script:** [lab_pcloud_scripts.md - Verify Installation](lab_pcloud_scripts.md#43-verify-installation)

#### Step 3: Install PSM on Connector02 (Secondary)

1. **Repeat Installation Steps**
   - Same process as Connector01
   - Use different server name: `Connector02`
   - Use different SIA connector name: `SIA-Connector02`

2. **Installation Wizard Settings**
   ```
   PSM Settings:
   ├── PSM Server Name:    Connector02
   ├── Service Account:    svc-CyberArkPSM@indramind.cyblab (same account)
   └── Recording Path:     D:\PSMRecordings

   SIA Settings:
   ├── Enable SIA:         Yes
   └── Connector Name:     SIA-Connector02
   ```

#### Step 4: Configure HA in Privilege Cloud

1. **Login to Privilege Cloud Portal**
   ```
   URL: https://[tenant].privilegecloud.cyberark.cloud
   ```

2. **Navigate to Connectors**
   ```
   Administration → System Health → Connectors
   ```

3. **Verify Both PSM Servers Appear**
   ```
   PSM Servers:
   ├── Connector01 - Status: Connected
   └── Connector02 - Status: Connected
   ```

4. **Configure DNS Round Robin for HA**
   ```
   Create DNS A records for load balancing:

   psm.indramind.cyblab  →  10.10.3.8   (Connector01, DHCP)
   psm.indramind.cyblab  →  10.10.3.70  (Connector02, DHCP)

   DNS round robin distributes connections between both PSM servers.
   ```

#### Step 5: Test HA Failover

1. **Test Normal Operation**
   ```
   1. Connect to a target system via PSM
   2. Verify session works on either PSM server
   3. Check which PSM server handled the session
   ```

2. **Test Failover**
   > **Script:** [lab_pcloud_scripts.md - Test HA Failover](lab_pcloud_scripts.md#44-test-ha-failover)

3. **Verify Automatic Recovery**
   ```
   After starting PSM01, verify it rejoins the pool:
   Administration → System Health → Connectors
   Both servers should show "Connected"
   ```

---

## 9. Naming Convention Standards

### 9.1 Safe Naming Convention

```
FORMAT: {ENV}-{TIER}-{TYPE}-{OWNER}

Examples:
├── LAB-T0-WIN-DomainAdmins     Tier 0 Domain Admin accounts
├── LAB-T1-WIN-Servers          Tier 1 Windows server local admins
├── LAB-T1-LNX-Servers          Tier 1 Linux root accounts
├── LAB-T1-DB-Oracle            Tier 1 Oracle DBA accounts
├── LAB-SVC-Automation          Service accounts for automation
└── PRD-T0-WIN-DomainAdmins     Production Domain Admin accounts
```

### 9.2 HOW-TO: Implement Naming Conventions

#### Step 1: Create Initial Safes

**In Privilege Cloud Portal:**

1. **Navigate to Safes**
   ```
   Policies → Safes → Add Safe
   ```

2. **Create Tier 0 Safes**
   ```
   Safe Name:        LAB-T0-WIN-DomainAdmins
   Description:      Tier 0 Domain Administrator accounts for lab environment
   Managing CPM:     LabCPM

   Members:
   ├── PAM-Vault-Admins     (Full permissions)
   ├── PAM-T0-Admins        (Use, Retrieve)
   ├── PAM-Auditors         (List, View Audit)
   └── PAM-Approvers        (Approve requests)

   Dual Control:     Required (2 approvers, 60-min timeout)
   ```

3. **Create Tier 1 Safes**
   ```
   Safe Name:        LAB-T1-WIN-Servers
   Description:      Tier 1 Windows Server local admin accounts
   Managing CPM:     LabCPM

   Members:
   ├── PAM-Safe-Managers    (Full permissions)
   ├── PAM-T1-WindowsAdmins (Use, Retrieve)
   ├── PAM-T1-Operators     (Use only)
   └── PAM-Auditors         (List, View Audit)

   Dual Control:     Optional
   ```

#### Step 2: Create Safe Naming Policy Document

Create and share with team:

```markdown
# Safe Naming Convention Policy

## Format
`{ENV}-{TIER}-{TYPE}-{OWNER}`

## Environment Codes
| Code | Environment |
|------|-------------|
| LAB  | Lab/Test    |
| DEV  | Development |
| PRD  | Production  |

## Tier Codes
| Code | Tier        | Description              |
|------|-------------|--------------------------|
| T0   | Tier 0      | Domain Controllers       |
| T1   | Tier 1      | Enterprise Servers       |
| T2   | Tier 2      | Workstations             |
| SVC  | Service     | Service Accounts         |

## Type Codes
| Code | Type        |
|------|-------------|
| WIN  | Windows     |
| LNX  | Linux       |
| DB   | Database    |
| NET  | Network     |
| CLD  | Cloud       |

## Examples
- `LAB-T0-WIN-DomainAdmins`
- `PRD-T1-DB-Oracle`
- `DEV-SVC-Automation`

## Enforcement
All new safes MUST follow this convention.
Non-compliant safes will be renamed.
```

#### Step 3: Platform Naming Convention

**Standard Platforms (Use as-is):**
```
Windows Domain Account
Windows Local Account
Unix SSH Account
Oracle Database
MS SQL Server
MySQL Database
```

**Custom Platforms (when needed):**
```
Format: {BasePlatform}-{CustomSuffix}

Examples:
├── Windows Domain Account-LabServers
├── Unix SSH Account-RHEL8
└── Oracle Database-ERP
```

#### Step 4: Account Naming Convention

```
FORMAT: {PLATFORM}_{TYPE}_{TARGET}_{USERNAME}

Examples:
├── WIN_DA_LAB_Administrator     Windows Domain Admin on lab domain
├── WIN_LA_SRV01_Administrator   Windows Local Admin on SRV01
├── LNX_ROOT_WEB01_root          Linux root on WEB01
├── ORA_DBA_PROD_sys             Oracle DBA sys on PROD
└── SQL_SA_SQLPRD_sa             SQL Server SA on SQLPRD
```

---

## 10. OKTA Integration

### 10.1 SAML Federation Design

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      OKTA SAML INTEGRATION                               │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  User → Privilege Cloud → CyberArk Identity (SP) ←SAML→ OKTA (IdP)       │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 10.2 HOW-TO: Configure OKTA Integration

> **NOTE:** This requires OKTA Admin access. Recover admin credentials before proceeding.

#### Prerequisites

- OKTA Admin access
- CyberArk Identity Admin access
- List of pilot users for testing

#### Step 1: Create SAML Application in OKTA

1. **Login to OKTA Admin Console**
   ```
   URL: https://[org].okta.com/admin
   ```

2. **Create New Application**
   ```
   Applications → Create App Integration
   ├── Sign-in method: SAML 2.0
   └── Click "Next"
   ```

3. **General Settings**
   ```
   App name:           CyberArk Privilege Cloud
   App logo:           [Upload CyberArk logo]
   App visibility:     Show in Okta dashboard
   ```

4. **SAML Settings**
   ```
   Single sign-on URL:     https://[tenant].id.cyberark.cloud/saml/consume
   Audience URI (SP):      https://[tenant].id.cyberark.cloud
   Name ID format:         EmailAddress
   Application username:   Email

   Attribute Statements:
   ├── firstName    → user.firstName
   ├── lastName     → user.lastName
   ├── email        → user.email
   └── groups       → appuser.groups
   ```

5. **Download Metadata**
   ```
   Sign On tab → Download SAML Signing Certificate
   Copy: Identity Provider Issuer URL
   Copy: Identity Provider Single Sign-On URL
   ```

#### Step 2: Configure OKTA as IdP in CyberArk Identity

1. **Login to CyberArk Identity Admin**
   ```
   URL: https://[tenant].id.cyberark.cloud/admin
   ```

2. **Navigate to External IdP**
   ```
   Settings → Authentication → External IdP → Add
   ```

3. **Configure OKTA IdP**
   ```
   IdP Name:            OKTA
   IdP Type:            SAML 2.0

   SAML Configuration:
   ├── IdP Issuer:          [From OKTA metadata]
   ├── IdP SSO URL:         [From OKTA metadata]
   ├── IdP Certificate:     [Upload OKTA certificate]
   └── SP Entity ID:        https://[tenant].id.cyberark.cloud

   User Mapping:
   ├── Match by:            Email
   └── Create users:        No (users must exist)

   Group Mapping:
   ├── CyberArk-Admins   → PAM-Vault-Admins
   ├── CyberArk-Users    → PAM-Users
   └── CyberArk-Auditors → PAM-Auditors
   ```

4. **Save Configuration**

#### Step 3: Assign Users in OKTA

1. **Navigate to Application**
   ```
   Applications → CyberArk Privilege Cloud → Assignments
   ```

2. **Assign Pilot Users**
   ```
   Assign → Assign to People
   Select pilot users → Assign → Done
   ```

3. **Assign Groups (Optional)**
   ```
   Assign → Assign to Groups
   Select: CyberArk-Users, CyberArk-Admins, etc.
   ```

#### Step 4: Test OKTA Integration

1. **Test IdP-Initiated Login**
   ```
   Login to OKTA → Click "CyberArk Privilege Cloud" tile
   Should redirect to CyberArk Identity and create session
   ```

2. **Test SP-Initiated Login**
   ```
   Navigate to: https://[tenant].id.cyberark.cloud
   Click "Login with OKTA"
   Redirects to OKTA → Authenticate → Returns to CyberArk
   ```

3. **Verify Group Mapping**
   ```
   After OKTA login, check user's CyberArk Identity groups
   Users → [Username] → Groups
   Verify OKTA groups mapped correctly
   ```

---

## 11. Implementation Roadmap

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      IMPLEMENTATION PHASES                               │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  PHASE 1: FOUNDATION (Week 1-2)                                          │
│  ══════════════════════════════                                          │
│  □ Validate AD (DC01)                                                    │
│  □ Run Create-TierStructure.ps1                                          │
│  □ Create service accounts                                               │
│  □ Configure AD as IdP in CyberArk Identity                              │
│                                                                          │
│  PHASE 2: USER MANAGEMENT (Week 2-3)                                     │
│  ═══════════════════════════════════                                     │
│  □ Export and audit current users                                        │
│  □ Update email addresses                                                │
│  □ Create domain accounts                                                │
│  □ Disable departed users                                                │
│                                                                          │
│  PHASE 3: MFA & INFRASTRUCTURE (Week 3-4)                                │
│  ═════════════════════════════════════════                               │
│  □ Create LAB-2FA-Standard policy                                        │
│  □ Create LAB-2FA-HighSecurity policy                                    │
│  □ Configure Email/Phone OTP                                             │
│  □ Configure Mobile App                                                  │
│  □ Deploy PSM + SIA on Connector01                                       │
│  □ Deploy PSM + SIA on Connector02                                       │
│  □ Configure DNS round robin (psm.indramind.cyblab)                      │
│  □ Test HA failover                                                      │
│                                                                          │
│  PHASE 4: STANDARDS & OKTA (January+)                                    │
│  ═════════════════════════════════════                                   │
│  □ Implement naming conventions                                          │
│  □ Document standards for team                                           │
│  □ Recover OKTA Admin access                                             │
│  □ Configure OKTA SAML                                                   │
│  □ Test with pilot users                                                 │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 12. Quick Reference Commands

> **Script:** [lab_pcloud_scripts.md - Quick Reference Commands](lab_pcloud_scripts.md#5-quick-reference-commands)

### Key URLs

| Service | URL |
|---------|-----|
| Privilege Cloud Portal | https://[tenant].privilegecloud.cyberark.cloud |
| CyberArk Identity Admin | https://[tenant].id.cyberark.cloud/admin |
| CyberArk Identity User Portal | https://[tenant].id.cyberark.cloud |

---

*Document Version: 2.0*
*Last Updated: January 2026*
