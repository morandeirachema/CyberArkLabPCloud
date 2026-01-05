# Lab Privilege Cloud - PowerShell Scripts Reference

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                    CYBERARK PRIVILEGE CLOUD LAB ENVIRONMENT                      ║
║                         PowerShell Scripts Library                               ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

## Document Control

| Property | Value |
|----------|-------|
| **Document ID** | LAB-SCRIPTS-001 |
| **Version** | 3.0 |
| **Status** | Production Ready |
| **Classification** | Internal Use |
| **Last Updated** | January 2026 |
| **Owner** | Lab Team |

> **Reference:** See [lab_pcloud_architecture.md](lab_pcloud_architecture.md) for full technical architecture.

---

## 📚 External References

### PowerShell Cmdlet Documentation

| Cmdlet | Purpose | Documentation |
|--------|---------|---------------|
| `New-ADUser` | Create AD users | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/new-aduser) |
| `New-ADGroup` | Create AD groups | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/new-adgroup) |
| `New-ADOrganizationalUnit` | Create OUs | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/new-adorganizationalunit) |
| `Get-ADUser` | Query AD users | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-aduser) |
| `Get-ADGroup` | Query AD groups | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-adgroup) |
| `Add-ADGroupMember` | Add group members | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/add-adgroupmember) |
| `Disable-ADAccount` | Disable accounts | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/disable-adaccount) |
| `Move-ADObject` | Move AD objects | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/move-adobject) |
| `Get-Service` | Query Windows services | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-service) |
| `Test-NetConnection` | Test connectivity | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) |
| `Install-WindowsFeature` | Install roles/features | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/servermanager/install-windowsfeature) |

### CyberArk Documentation

| Topic | Link |
|-------|------|
| AD Connector Setup | [CyberArk Identity - AD Integration](https://docs.cyberark.com/identity/latest/en/content/integrations/ad/ad-integration.htm) |
| PSM Installation | [Install Connector](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/install-connector.htm) |
| Connector Requirements | [System Requirements](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/connector-requirements.htm) |
| Service Account Best Practices | [PSM Service Account](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/psm-service-account.htm) |

### Windows Server Documentation

| Topic | Link |
|-------|------|
| Remote Desktop Services | [RDS Overview](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/welcome-to-rds) |
| Windows Defender Exclusions | [Configure Exclusions](https://learn.microsoft.com/en-us/microsoft-365/security/defender-endpoint/configure-exclusions-microsoft-defender-antivirus) |
| DNS Server | [DNS Documentation](https://learn.microsoft.com/en-us/windows-server/networking/dns/dns-top) |

---

## Table of Contents

1. [AD Connector Service Account](#1-ad-connector-service-account)
2. [TIER Model Structure](#2-tier-model-structure)
3. [User Management](#3-user-management)
4. [PSM Server Preparation](#4-psm-server-preparation)
5. [DNS Configuration](#5-dns-configuration)
6. [Verification Scripts](#6-verification-scripts)
7. [Quick Reference](#7-quick-reference)

---

## 1. AD Connector Service Account

> **📚 Reference:** [CyberArk Identity AD Integration](https://docs.cyberark.com/identity/latest/en/content/integrations/ad/ad-integration.htm)

Create the service account for CyberArk Identity AD connector.

| Property | Value |
|----------|-------|
| **Run on** | DC01 (Domain Controller) |
| **Permissions Required** | Domain Admin |
| **Account Created** | svc-CyberArkIdentity |

### 1.1 Create Service Account

```powershell
#Requires -Modules ActiveDirectory
#Requires -RunAsAdministrator

<#
.SYNOPSIS
    Creates the CyberArk Identity AD Connector service account.
.DESCRIPTION
    Creates a service account with read-only permissions for AD synchronization.
.EXAMPLE
    .\Create-IdentityServiceAccount.ps1
#>

# Configuration
$DomainDN = "DC=indramind,DC=cyblab"
$ServiceAccountName = "svc-CyberArkIdentity"
$ServiceAccountPath = "OU=ServiceAccounts,OU=PAM,$DomainDN"

# Generate secure password
$Password = -join ((65..90) + (97..122) + (48..57) + (33,35,36,37,38,42) |
    Get-Random -Count 32 | ForEach-Object {[char]$_})
$SecurePassword = ConvertTo-SecureString -String $Password -AsPlainText -Force

Write-Host "Creating AD Connector Service Account..." -ForegroundColor Cyan
Write-Host "=========================================" -ForegroundColor Cyan

try {
    # Check if account exists
    $Existing = Get-ADUser -Filter "SamAccountName -eq '$ServiceAccountName'" -ErrorAction SilentlyContinue

    if ($Existing) {
        Write-Host "[EXISTS] $ServiceAccountName already exists" -ForegroundColor Yellow
    } else {
        # Create service account
        New-ADUser -Name $ServiceAccountName `
            -SamAccountName $ServiceAccountName `
            -UserPrincipalName "$ServiceAccountName@indramind.cyblab" `
            -Path $ServiceAccountPath `
            -AccountPassword $SecurePassword `
            -Enabled $true `
            -PasswordNeverExpires $true `
            -CannotChangePassword $true `
            -Description "CyberArk Identity AD Connector - Read Only"

        Write-Host "[CREATED] $ServiceAccountName" -ForegroundColor Green
        Write-Host ""
        Write-Host "╔══════════════════════════════════════════════════════════════╗" -ForegroundColor Yellow
        Write-Host "║  SAVE THIS PASSWORD SECURELY - SHOWN ONLY ONCE               ║" -ForegroundColor Yellow
        Write-Host "╠══════════════════════════════════════════════════════════════╣" -ForegroundColor Yellow
        Write-Host "║  Account:  $ServiceAccountName@indramind.cyblab" -ForegroundColor White
        Write-Host "║  Password: $Password" -ForegroundColor White
        Write-Host "╚══════════════════════════════════════════════════════════════╝" -ForegroundColor Yellow
    }
} catch {
    Write-Host "[ERROR] $($_.Exception.Message)" -ForegroundColor Red
}
```

### 1.2 Grant Read Permissions

```powershell
# Grant read permissions to PAM OU
$ServiceAccount = Get-ADUser $ServiceAccountName
$PAM_OU = "OU=PAM,$DomainDN"

# Get the OU
$OU = Get-ADOrganizationalUnit -Identity $PAM_OU

# Get current ACL
$ACL = Get-Acl -Path "AD:\$($OU.DistinguishedName)"

# Create access rule
$UserSID = New-Object System.Security.Principal.SecurityIdentifier($ServiceAccount.SID)
$AccessRule = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $UserSID,
    "GenericRead",
    "Allow",
    "Descendents"
)

# Apply
$ACL.AddAccessRule($AccessRule)
Set-Acl -Path "AD:\$($OU.DistinguishedName)" -AclObject $ACL

Write-Host "[OK] Read permissions granted to $ServiceAccountName on $PAM_OU" -ForegroundColor Green
```

---

## 2. TIER Model Structure

> **📚 Reference:** [Microsoft Privileged Access Model](https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-access-model)

Complete script to create PAM OU structure, security groups, and service accounts.

| Property | Value |
|----------|-------|
| **Run on** | DC01 (Domain Controller) |
| **Permissions Required** | Domain Admin |
| **Save as** | `Create-TierStructure.ps1` |

### 2.1 Create-TierStructure.ps1

```powershell
#Requires -Modules ActiveDirectory
#Requires -RunAsAdministrator

<#
.SYNOPSIS
    Creates the complete TIER model OU structure for CyberArk PAM.

.DESCRIPTION
    This script creates:
    - PAM root OU with Tier0, Tier1, Tier2 sub-OUs
    - Security groups for each tier
    - Service account OUs
    - Admin groups for CyberArk roles
    - User OUs for active and disabled accounts

.PARAMETER DomainDN
    The domain distinguished name (e.g., "DC=indramind,DC=cyblab")

.EXAMPLE
    .\Create-TierStructure.ps1 -DomainDN "DC=indramind,DC=cyblab"

.NOTES
    Author: Lab Team
    Version: 3.0
    Requires: ActiveDirectory PowerShell module
#>

param(
    [Parameter(Mandatory = $true, HelpMessage = "Domain DN (e.g., DC=indramind,DC=cyblab)")]
    [ValidatePattern("^DC=.+")]
    [string]$DomainDN
)

$ErrorActionPreference = "Stop"

# ═══════════════════════════════════════════════════════════════════════════════
# BANNER
# ═══════════════════════════════════════════════════════════════════════════════

Write-Host ""
Write-Host "╔══════════════════════════════════════════════════════════════════════╗" -ForegroundColor Cyan
Write-Host "║           CYBERARK PAM - TIER MODEL STRUCTURE CREATION               ║" -ForegroundColor Cyan
Write-Host "╠══════════════════════════════════════════════════════════════════════╣" -ForegroundColor Cyan
Write-Host "║  Domain: $DomainDN" -ForegroundColor White
Write-Host "╚══════════════════════════════════════════════════════════════════════╝" -ForegroundColor Cyan
Write-Host ""

# ═══════════════════════════════════════════════════════════════════════════════
# ORGANIZATIONAL UNITS
# ═══════════════════════════════════════════════════════════════════════════════

Write-Host "[SECTION 1] Creating Organizational Units" -ForegroundColor Cyan
Write-Host "─────────────────────────────────────────" -ForegroundColor Cyan

$OUs = @(
    # Root PAM OU
    @{ Name = "PAM"; Path = $DomainDN; Description = "CyberArk Privileged Access Management" }

    # Tier 0 - Domain Controllers
    @{ Name = "Tier0"; Path = "OU=PAM,$DomainDN"; Description = "Tier 0 - Domain Controllers and Forest Admins" }
    @{ Name = "Groups"; Path = "OU=Tier0,OU=PAM,$DomainDN"; Description = "Tier 0 Security Groups" }
    @{ Name = "Accounts"; Path = "OU=Tier0,OU=PAM,$DomainDN"; Description = "Tier 0 Privileged Accounts" }

    # Tier 1 - Enterprise Servers
    @{ Name = "Tier1"; Path = "OU=PAM,$DomainDN"; Description = "Tier 1 - Enterprise Servers" }
    @{ Name = "Groups"; Path = "OU=Tier1,OU=PAM,$DomainDN"; Description = "Tier 1 Security Groups" }
    @{ Name = "Accounts"; Path = "OU=Tier1,OU=PAM,$DomainDN"; Description = "Tier 1 Privileged Accounts" }

    # Tier 2 - Workstations
    @{ Name = "Tier2"; Path = "OU=PAM,$DomainDN"; Description = "Tier 2 - Workstations" }
    @{ Name = "Groups"; Path = "OU=Tier2,OU=PAM,$DomainDN"; Description = "Tier 2 Security Groups" }
    @{ Name = "Accounts"; Path = "OU=Tier2,OU=PAM,$DomainDN"; Description = "Tier 2 Privileged Accounts" }

    # Service Accounts
    @{ Name = "ServiceAccounts"; Path = "OU=PAM,$DomainDN"; Description = "CyberArk Service Accounts" }

    # Admin Groups
    @{ Name = "AdminGroups"; Path = "OU=PAM,$DomainDN"; Description = "CyberArk Administrative Groups" }

    # Users
    @{ Name = "Users"; Path = "OU=PAM,$DomainDN"; Description = "Lab Team User Accounts" }
    @{ Name = "Active"; Path = "OU=Users,OU=PAM,$DomainDN"; Description = "Active Team Members" }
    @{ Name = "Disabled"; Path = "OU=Users,OU=PAM,$DomainDN"; Description = "Disabled Accounts" }
)

$OUCreated = 0; $OUExisted = 0; $OUFailed = 0

foreach ($OU in $OUs) {
    $FullPath = "OU=$($OU.Name),$($OU.Path)"
    try {
        $Existing = Get-ADOrganizationalUnit -Filter "DistinguishedName -eq '$FullPath'" -ErrorAction SilentlyContinue
        if ($Existing) {
            Write-Host "  [EXISTS]  $($OU.Name)" -ForegroundColor Yellow
            $OUExisted++
        } else {
            New-ADOrganizationalUnit -Name $OU.Name -Path $OU.Path -Description $OU.Description -ProtectedFromAccidentalDeletion $true
            Write-Host "  [CREATED] $($OU.Name)" -ForegroundColor Green
            $OUCreated++
        }
    } catch {
        Write-Host "  [ERROR]   $($OU.Name) - $($_.Exception.Message)" -ForegroundColor Red
        $OUFailed++
    }
}

Write-Host ""
Write-Host "  Summary: $OUCreated created, $OUExisted existed, $OUFailed failed" -ForegroundColor Gray
Write-Host ""

# ═══════════════════════════════════════════════════════════════════════════════
# SECURITY GROUPS
# ═══════════════════════════════════════════════════════════════════════════════

Write-Host "[SECTION 2] Creating Security Groups" -ForegroundColor Cyan
Write-Host "────────────────────────────────────" -ForegroundColor Cyan

$Groups = @(
    # Tier 0 Groups
    @{ Name = "PAM-T0-Admins"; Path = "OU=Groups,OU=Tier0,OU=PAM,$DomainDN"; Description = "Tier 0 Administrators - Full DC Access"; Scope = "Global" }
    @{ Name = "PAM-T0-Operators"; Path = "OU=Groups,OU=Tier0,OU=PAM,$DomainDN"; Description = "Tier 0 Operators - Limited DC Access"; Scope = "Global" }
    @{ Name = "PAM-T0-Auditors"; Path = "OU=Groups,OU=Tier0,OU=PAM,$DomainDN"; Description = "Tier 0 Auditors - Read-Only Access"; Scope = "Global" }

    # Tier 1 Groups
    @{ Name = "PAM-T1-Admins"; Path = "OU=Groups,OU=Tier1,OU=PAM,$DomainDN"; Description = "Tier 1 Administrators - Server Access"; Scope = "Global" }
    @{ Name = "PAM-T1-WindowsAdmins"; Path = "OU=Groups,OU=Tier1,OU=PAM,$DomainDN"; Description = "Tier 1 Windows Server Administrators"; Scope = "Global" }
    @{ Name = "PAM-T1-LinuxAdmins"; Path = "OU=Groups,OU=Tier1,OU=PAM,$DomainDN"; Description = "Tier 1 Linux Server Administrators"; Scope = "Global" }
    @{ Name = "PAM-T1-DatabaseAdmins"; Path = "OU=Groups,OU=Tier1,OU=PAM,$DomainDN"; Description = "Tier 1 Database Administrators"; Scope = "Global" }
    @{ Name = "PAM-T1-Operators"; Path = "OU=Groups,OU=Tier1,OU=PAM,$DomainDN"; Description = "Tier 1 Operators - Limited Server Access"; Scope = "Global" }

    # Tier 2 Groups
    @{ Name = "PAM-T2-Admins"; Path = "OU=Groups,OU=Tier2,OU=PAM,$DomainDN"; Description = "Tier 2 Administrators - Workstation Access"; Scope = "Global" }
    @{ Name = "PAM-T2-HelpDesk"; Path = "OU=Groups,OU=Tier2,OU=PAM,$DomainDN"; Description = "Tier 2 Help Desk - Limited Workstation Access"; Scope = "Global" }

    # Admin Groups
    @{ Name = "PAM-Vault-Admins"; Path = "OU=AdminGroups,OU=PAM,$DomainDN"; Description = "CyberArk Vault Administrators"; Scope = "Global" }
    @{ Name = "PAM-Safe-Managers"; Path = "OU=AdminGroups,OU=PAM,$DomainDN"; Description = "CyberArk Safe Managers"; Scope = "Global" }
    @{ Name = "PAM-Auditors"; Path = "OU=AdminGroups,OU=PAM,$DomainDN"; Description = "CyberArk Audit and Compliance Team"; Scope = "Global" }
    @{ Name = "PAM-Users"; Path = "OU=AdminGroups,OU=PAM,$DomainDN"; Description = "CyberArk Standard Users"; Scope = "Global" }
    @{ Name = "PAM-Approvers"; Path = "OU=AdminGroups,OU=PAM,$DomainDN"; Description = "CyberArk Access Request Approvers"; Scope = "Global" }
)

$GroupCreated = 0; $GroupExisted = 0; $GroupFailed = 0

foreach ($Group in $Groups) {
    try {
        $Existing = Get-ADGroup -Filter "Name -eq '$($Group.Name)'" -ErrorAction SilentlyContinue
        if ($Existing) {
            Write-Host "  [EXISTS]  $($Group.Name)" -ForegroundColor Yellow
            $GroupExisted++
        } else {
            New-ADGroup -Name $Group.Name -Path $Group.Path -GroupScope $Group.Scope -GroupCategory Security -Description $Group.Description
            Write-Host "  [CREATED] $($Group.Name)" -ForegroundColor Green
            $GroupCreated++
        }
    } catch {
        Write-Host "  [ERROR]   $($Group.Name) - $($_.Exception.Message)" -ForegroundColor Red
        $GroupFailed++
    }
}

Write-Host ""
Write-Host "  Summary: $GroupCreated created, $GroupExisted existed, $GroupFailed failed" -ForegroundColor Gray
Write-Host ""

# ═══════════════════════════════════════════════════════════════════════════════
# SERVICE ACCOUNTS
# ═══════════════════════════════════════════════════════════════════════════════

Write-Host "[SECTION 3] Creating Service Accounts" -ForegroundColor Cyan
Write-Host "─────────────────────────────────────" -ForegroundColor Cyan

$ServiceAccounts = @(
    @{ Name = "svc-CyberArkPSM"; Description = "CyberArk PSM Service Account" }
    @{ Name = "svc-CyberArkScan"; Description = "CyberArk Scanner Service Account" }
    @{ Name = "svc-CyberArkIdentity"; Description = "CyberArk Identity AD Connector" }
    @{ Name = "svc-CyberArkReconcile"; Description = "CyberArk Reconciliation Account" }
)

$SvcCreated = 0; $SvcExisted = 0; $SvcFailed = 0
$DomainName = ($DomainDN -replace "DC=", "" -replace ",", ".").ToLower()
$GeneratedPasswords = @()

foreach ($SvcAcct in $ServiceAccounts) {
    try {
        $Existing = Get-ADUser -Filter "SamAccountName -eq '$($SvcAcct.Name)'" -ErrorAction SilentlyContinue
        if ($Existing) {
            Write-Host "  [EXISTS]  $($SvcAcct.Name)" -ForegroundColor Yellow
            $SvcExisted++
        } else {
            # Generate secure password
            $Password = -join ((65..90) + (97..122) + (48..57) + (33,35,36,37,38,42) |
                Get-Random -Count 32 | ForEach-Object {[char]$_})
            $SecurePassword = ConvertTo-SecureString -String $Password -AsPlainText -Force

            New-ADUser -Name $SvcAcct.Name `
                -SamAccountName $SvcAcct.Name `
                -UserPrincipalName "$($SvcAcct.Name)@$DomainName" `
                -Path "OU=ServiceAccounts,OU=PAM,$DomainDN" `
                -AccountPassword $SecurePassword `
                -Enabled $true `
                -PasswordNeverExpires $true `
                -CannotChangePassword $true `
                -Description $SvcAcct.Description

            Write-Host "  [CREATED] $($SvcAcct.Name)" -ForegroundColor Green
            $GeneratedPasswords += @{ Account = "$($SvcAcct.Name)@$DomainName"; Password = $Password }
            $SvcCreated++
        }
    } catch {
        Write-Host "  [ERROR]   $($SvcAcct.Name) - $($_.Exception.Message)" -ForegroundColor Red
        $SvcFailed++
    }
}

Write-Host ""
Write-Host "  Summary: $SvcCreated created, $SvcExisted existed, $SvcFailed failed" -ForegroundColor Gray

# Output generated passwords
if ($GeneratedPasswords.Count -gt 0) {
    Write-Host ""
    Write-Host "╔══════════════════════════════════════════════════════════════════════╗" -ForegroundColor Yellow
    Write-Host "║  SAVE THESE PASSWORDS SECURELY - SHOWN ONLY ONCE                     ║" -ForegroundColor Yellow
    Write-Host "╠══════════════════════════════════════════════════════════════════════╣" -ForegroundColor Yellow
    foreach ($Cred in $GeneratedPasswords) {
        Write-Host "║  Account:  $($Cred.Account)" -ForegroundColor White
        Write-Host "║  Password: $($Cred.Password)" -ForegroundColor White
        Write-Host "║                                                                      ║" -ForegroundColor Yellow
    }
    Write-Host "╚══════════════════════════════════════════════════════════════════════╝" -ForegroundColor Yellow
}

# ═══════════════════════════════════════════════════════════════════════════════
# COMPLETION SUMMARY
# ═══════════════════════════════════════════════════════════════════════════════

Write-Host ""
Write-Host "╔══════════════════════════════════════════════════════════════════════╗" -ForegroundColor Green
Write-Host "║              TIER MODEL STRUCTURE CREATION COMPLETE                  ║" -ForegroundColor Green
Write-Host "╠══════════════════════════════════════════════════════════════════════╣" -ForegroundColor Green
Write-Host "║  OUs:              $OUCreated created, $OUExisted existed" -ForegroundColor White
Write-Host "║  Security Groups:  $GroupCreated created, $GroupExisted existed" -ForegroundColor White
Write-Host "║  Service Accounts: $SvcCreated created, $SvcExisted existed" -ForegroundColor White
Write-Host "╚══════════════════════════════════════════════════════════════════════╝" -ForegroundColor Green
Write-Host ""
```

### 2.2 Execute Script

```powershell
# Run the script on DC01
.\Create-TierStructure.ps1 -DomainDN "DC=indramind,DC=cyblab"
```

---

## 3. User Management

> **📚 Reference:** [CyberArk Identity User Management](https://docs.cyberark.com/identity/latest/en/content/admin/user-management.htm)

### 3.1 Create Domain Accounts

| Property | Value |
|----------|-------|
| **Run on** | DC01 (Domain Controller) |
| **Permissions Required** | Domain Admin or Account Operator |

```powershell
<#
.SYNOPSIS
    Creates domain accounts for lab team members.
.PARAMETER Users
    Array of user objects with FirstName, LastName, Email, and Groups properties.
.EXAMPLE
    .\Create-TeamMembers.ps1
#>

$DomainDN = "DC=indramind,DC=cyblab"
$UserPath = "OU=Active,OU=Users,OU=PAM,$DomainDN"

# Define team members
$Users = @(
    @{
        FirstName = "Juan"
        LastName = "Garcia"
        Email = "juan.garcia@company.com"
        Groups = @("PAM-Users", "PAM-T1-WindowsAdmins")
    }
    @{
        FirstName = "Maria"
        LastName = "Lopez"
        Email = "maria.lopez@company.com"
        Groups = @("PAM-Vault-Admins", "PAM-Safe-Managers")
    }
    @{
        FirstName = "Carlos"
        LastName = "Rodriguez"
        Email = "carlos.rodriguez@company.com"
        Groups = @("PAM-Users", "PAM-Auditors")
    }
)

Write-Host "Creating Team Member Accounts" -ForegroundColor Cyan
Write-Host "=============================" -ForegroundColor Cyan

foreach ($User in $Users) {
    $Username = "$($User.FirstName.ToLower()).$($User.LastName.ToLower())"
    $DisplayName = "$($User.FirstName) $($User.LastName)"

    try {
        # Check if exists
        $Existing = Get-ADUser -Filter "SamAccountName -eq '$Username'" -ErrorAction SilentlyContinue
        if ($Existing) {
            Write-Host "[EXISTS]  $Username" -ForegroundColor Yellow
            continue
        }

        # Generate temp password
        $TempPassword = "Welcome123!" + (Get-Random -Minimum 100 -Maximum 999)
        $SecurePassword = ConvertTo-SecureString $TempPassword -AsPlainText -Force

        # Create user
        New-ADUser -Name $DisplayName `
            -GivenName $User.FirstName `
            -Surname $User.LastName `
            -SamAccountName $Username `
            -UserPrincipalName "$Username@indramind.cyblab" `
            -EmailAddress $User.Email `
            -Path $UserPath `
            -AccountPassword $SecurePassword `
            -Enabled $true `
            -ChangePasswordAtLogon $true

        # Add to groups
        foreach ($Group in $User.Groups) {
            Add-ADGroupMember -Identity $Group -Members $Username -ErrorAction SilentlyContinue
        }

        Write-Host "[CREATED] $Username (Password: $TempPassword)" -ForegroundColor Green
        Write-Host "          Groups: $($User.Groups -join ', ')" -ForegroundColor Gray

    } catch {
        Write-Host "[ERROR]   $Username - $($_.Exception.Message)" -ForegroundColor Red
    }
}

Write-Host ""
Write-Host "[COMPLETE] Trigger AD sync in CyberArk Identity" -ForegroundColor Cyan
```

### 3.2 Disable Departed Users

```powershell
<#
.SYNOPSIS
    Disables departed user accounts and moves to Disabled OU.
#>

$DomainDN = "DC=indramind,DC=cyblab"
$DisabledOU = "OU=Disabled,OU=Users,OU=PAM,$DomainDN"

# List of departed users
$DepartedUsers = @(
    "yago",
    "olmedo",
    "ana"
)

Write-Host "Disabling Departed Users" -ForegroundColor Cyan
Write-Host "========================" -ForegroundColor Cyan

foreach ($Username in $DepartedUsers) {
    try {
        $User = Get-ADUser -Filter "SamAccountName -eq '$Username'" -ErrorAction SilentlyContinue

        if (-not $User) {
            Write-Host "[NOT FOUND] $Username" -ForegroundColor Yellow
            continue
        }

        # Disable account
        Disable-ADAccount -Identity $User

        # Add description
        Set-ADUser -Identity $User -Description "Disabled $(Get-Date -Format 'yyyy-MM-dd') - Departed"

        # Move to Disabled OU
        Move-ADObject -Identity $User.DistinguishedName -TargetPath $DisabledOU

        Write-Host "[DISABLED] $Username - Moved to Disabled OU" -ForegroundColor Green

    } catch {
        Write-Host "[ERROR]    $Username - $($_.Exception.Message)" -ForegroundColor Red
    }
}
```

---

## 4. PSM Server Preparation

> **📚 Reference:** [CyberArk PSM Installation](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/install-connector.htm)

### 4.1 Pre-Installation Check

| Property | Value |
|----------|-------|
| **Run on** | Connector01, Connector02 |
| **Permissions Required** | Local Administrator |

```powershell
<#
.SYNOPSIS
    Pre-installation verification for PSM connector servers.
#>

Write-Host ""
Write-Host "╔══════════════════════════════════════════════════════════════════════╗" -ForegroundColor Cyan
Write-Host "║              PSM SERVER PRE-INSTALLATION CHECK                       ║" -ForegroundColor Cyan
Write-Host "╚══════════════════════════════════════════════════════════════════════╝" -ForegroundColor Cyan
Write-Host ""

$AllPassed = $true

# Check 1: Windows Features
Write-Host "[1] Windows Features" -ForegroundColor White
$RDS = Get-WindowsFeature -Name RDS-RD-Server
$NET = Get-WindowsFeature -Name NET-Framework-45-Core
Write-Host "    RDS Session Host:    $(if($RDS.Installed){'[PASS]'}else{'[MISSING]'})" -ForegroundColor $(if($RDS.Installed){'Green'}else{'Red'})
Write-Host "    .NET Framework:      $(if($NET.Installed){'[PASS]'}else{'[MISSING]'})" -ForegroundColor $(if($NET.Installed){'Green'}else{'Red'})
if (-not $RDS.Installed) { $AllPassed = $false }

# Check 2: Domain Membership
Write-Host ""
Write-Host "[2] Domain Membership" -ForegroundColor White
$CS = Get-WmiObject -Class Win32_ComputerSystem
Write-Host "    Domain:              $($CS.Domain)" -ForegroundColor $(if($CS.Domain -eq 'indramind.cyblab'){'Green'}else{'Yellow'})
Write-Host "    Part of Domain:      $(if($CS.PartOfDomain){'[PASS]'}else{'[FAIL]'})" -ForegroundColor $(if($CS.PartOfDomain){'Green'}else{'Red'})
if (-not $CS.PartOfDomain) { $AllPassed = $false }

# Check 3: Cloud Connectivity
Write-Host ""
Write-Host "[3] Cloud Connectivity" -ForegroundColor White
$CloudEndpoints = @(
    "connector.privilegecloud.cyberark.cloud",
    "console.privilegecloud.cyberark.cloud"
)
foreach ($Endpoint in $CloudEndpoints) {
    $Test = Test-NetConnection -ComputerName $Endpoint -Port 443 -WarningAction SilentlyContinue
    Write-Host "    $Endpoint`:443  $(if($Test.TcpTestSucceeded){'[PASS]'}else{'[FAIL]'})" -ForegroundColor $(if($Test.TcpTestSucceeded){'Green'}else{'Red'})
    if (-not $Test.TcpTestSucceeded) { $AllPassed = $false }
}

# Check 4: Domain Controller Connectivity
Write-Host ""
Write-Host "[4] Domain Controller Connectivity" -ForegroundColor White
$DCPorts = @(389, 636, 88, 445)
foreach ($Port in $DCPorts) {
    $Test = Test-NetConnection -ComputerName "DC01.indramind.cyblab" -Port $Port -WarningAction SilentlyContinue
    Write-Host "    DC01:$Port  $(if($Test.TcpTestSucceeded){'[PASS]'}else{'[FAIL]'})" -ForegroundColor $(if($Test.TcpTestSucceeded){'Green'}else{'Red'})
}

# Check 5: Disk Space
Write-Host ""
Write-Host "[5] Disk Space" -ForegroundColor White
$CDisk = Get-WmiObject Win32_LogicalDisk -Filter "DeviceID='C:'"
$FreeGB = [math]::Round($CDisk.FreeSpace / 1GB, 2)
Write-Host "    C: Drive Free:       $FreeGB GB $(if($FreeGB -gt 50){'[PASS]'}else{'[LOW]'})" -ForegroundColor $(if($FreeGB -gt 50){'Green'}else{'Yellow'})

# Summary
Write-Host ""
Write-Host "════════════════════════════════════════════════════════════════════════" -ForegroundColor Cyan
if ($AllPassed) {
    Write-Host "RESULT: All critical checks PASSED - Ready for installation" -ForegroundColor Green
} else {
    Write-Host "RESULT: Some checks FAILED - Resolve issues before installation" -ForegroundColor Red
}
Write-Host "════════════════════════════════════════════════════════════════════════" -ForegroundColor Cyan
```

### 4.2 Install Windows Features

```powershell
# Install required Windows features
Write-Host "Installing Windows Features..." -ForegroundColor Cyan

$Features = @(
    "RDS-RD-Server",
    "NET-Framework-45-Core",
    "RSAT-AD-PowerShell"
)

foreach ($Feature in $Features) {
    $Installed = Get-WindowsFeature -Name $Feature
    if ($Installed.Installed) {
        Write-Host "[EXISTS]  $Feature" -ForegroundColor Yellow
    } else {
        Write-Host "[INSTALLING] $Feature..." -ForegroundColor Cyan
        Install-WindowsFeature -Name $Feature -IncludeManagementTools
        Write-Host "[INSTALLED] $Feature" -ForegroundColor Green
    }
}

# Check if reboot required
$RebootPending = Test-Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending"
if ($RebootPending) {
    Write-Host ""
    Write-Host "[ACTION REQUIRED] Server reboot required before continuing" -ForegroundColor Yellow
}
```

### 4.3 Verify Installation

```powershell
<#
.SYNOPSIS
    Verifies PSM and SIA installation status.
#>

Write-Host ""
Write-Host "PSM Installation Verification" -ForegroundColor Cyan
Write-Host "==============================" -ForegroundColor Cyan

# Check PSM Service
$PSM = Get-Service -Name "CyberArk Privileged Session Manager" -ErrorAction SilentlyContinue
if ($PSM) {
    Write-Host "[PSM Service]" -ForegroundColor White
    Write-Host "  Status: $($PSM.Status)" -ForegroundColor $(if($PSM.Status -eq 'Running'){'Green'}else{'Red'})
} else {
    Write-Host "[PSM Service] NOT FOUND" -ForegroundColor Red
}

# Check SIA Service
$SIA = Get-Service -Name "CyberArk*SIA*" -ErrorAction SilentlyContinue
if ($SIA) {
    Write-Host "[SIA Service]" -ForegroundColor White
    Write-Host "  Status: $($SIA.Status)" -ForegroundColor $(if($SIA.Status -eq 'Running'){'Green'}else{'Red'})
} else {
    Write-Host "[SIA Service] NOT FOUND" -ForegroundColor Red
}

# Check Installation Path
$InstallPath = "C:\Program Files (x86)\CyberArk\PSM"
Write-Host "[Installation Directory]" -ForegroundColor White
Write-Host "  Path: $InstallPath" -ForegroundColor $(if(Test-Path $InstallPath){'Green'}else{'Red'})
```

---

## 5. DNS Configuration

> **📚 Reference:** [Microsoft DNS Documentation](https://learn.microsoft.com/en-us/windows-server/networking/dns/dns-top)

### 5.1 Create PSM HA DNS Records

| Property | Value |
|----------|-------|
| **Run on** | DC01 (DNS Server) |
| **Permissions Required** | DNS Admin |

```powershell
<#
.SYNOPSIS
    Creates DNS A records for PSM High Availability (Round Robin).
#>

$ZoneName = "indramind.cyblab"

Write-Host "Creating PSM HA DNS Records" -ForegroundColor Cyan
Write-Host "============================" -ForegroundColor Cyan

# PSM HA Records (Round Robin)
$Records = @(
    @{ Name = "psm"; IP = "10.10.3.8"; Comment = "Connector01" }
    @{ Name = "psm"; IP = "10.10.3.70"; Comment = "Connector02" }
    @{ Name = "connector01"; IP = "10.10.3.8"; Comment = "Direct" }
    @{ Name = "connector02"; IP = "10.10.3.70"; Comment = "Direct" }
)

foreach ($Record in $Records) {
    try {
        $Existing = Get-DnsServerResourceRecord -ZoneName $ZoneName -Name $Record.Name -RRType A -ErrorAction SilentlyContinue |
            Where-Object { $_.RecordData.IPv4Address.ToString() -eq $Record.IP }

        if ($Existing) {
            Write-Host "[EXISTS]  $($Record.Name).$ZoneName -> $($Record.IP)" -ForegroundColor Yellow
        } else {
            Add-DnsServerResourceRecordA -ZoneName $ZoneName -Name $Record.Name -IPv4Address $Record.IP -TimeToLive 00:05:00
            Write-Host "[CREATED] $($Record.Name).$ZoneName -> $($Record.IP) ($($Record.Comment))" -ForegroundColor Green
        }
    } catch {
        Write-Host "[ERROR]   $($Record.Name) -> $($Record.IP): $($_.Exception.Message)" -ForegroundColor Red
    }
}

# Verify
Write-Host ""
Write-Host "Verifying DNS Resolution:" -ForegroundColor Cyan
Resolve-DnsName -Name "psm.indramind.cyblab" -Type A | Format-Table Name, IPAddress -AutoSize
```

---

## 6. Verification Scripts

### 6.1 Complete AD Verification

```powershell
<#
.SYNOPSIS
    Comprehensive verification of AD structure.
#>

$DomainDN = "DC=indramind,DC=cyblab"

Write-Host ""
Write-Host "╔══════════════════════════════════════════════════════════════════════╗" -ForegroundColor Cyan
Write-Host "║              ACTIVE DIRECTORY STRUCTURE VERIFICATION                 ║" -ForegroundColor Cyan
Write-Host "╚══════════════════════════════════════════════════════════════════════╝" -ForegroundColor Cyan
Write-Host ""

# OUs
Write-Host "[1] Organizational Units" -ForegroundColor White
$OUs = Get-ADOrganizationalUnit -Filter * -SearchBase "OU=PAM,$DomainDN" -Properties Description
Write-Host "    Count: $($OUs.Count) OUs" -ForegroundColor $(if($OUs.Count -ge 12){'Green'}else{'Yellow'})
$OUs | Select-Object Name | Format-Table -HideTableHeaders

# Groups
Write-Host "[2] Security Groups" -ForegroundColor White
$Groups = Get-ADGroup -Filter * -SearchBase "OU=PAM,$DomainDN"
Write-Host "    Count: $($Groups.Count) groups" -ForegroundColor $(if($Groups.Count -ge 15){'Green'}else{'Yellow'})
$Groups | Select-Object Name, GroupScope | Format-Table -AutoSize

# Service Accounts
Write-Host "[3] Service Accounts" -ForegroundColor White
$SvcAccounts = Get-ADUser -Filter * -SearchBase "OU=ServiceAccounts,OU=PAM,$DomainDN" -Properties Enabled, Description
$SvcAccounts | Select-Object Name, Enabled, Description | Format-Table -AutoSize

# Users
Write-Host "[4] User Accounts" -ForegroundColor White
$ActiveUsers = (Get-ADUser -Filter * -SearchBase "OU=Active,OU=Users,OU=PAM,$DomainDN").Count
$DisabledUsers = (Get-ADUser -Filter * -SearchBase "OU=Disabled,OU=Users,OU=PAM,$DomainDN").Count
Write-Host "    Active Users:   $ActiveUsers" -ForegroundColor Green
Write-Host "    Disabled Users: $DisabledUsers" -ForegroundColor Gray
```

### 6.2 PSM HA Verification

```powershell
<#
.SYNOPSIS
    Verifies PSM High Availability configuration.
#>

Write-Host ""
Write-Host "PSM High Availability Verification" -ForegroundColor Cyan
Write-Host "====================================" -ForegroundColor Cyan

# DNS Round Robin Test
Write-Host ""
Write-Host "[1] DNS Round Robin Test" -ForegroundColor White
1..4 | ForEach-Object {
    $Result = Resolve-DnsName -Name "psm.indramind.cyblab" -Type A -DnsOnly
    Write-Host "    Query $_: $($Result.IPAddress -join ', ')"
}

# Service Status (run on each connector)
Write-Host ""
Write-Host "[2] Service Status" -ForegroundColor White
$PSMService = Get-Service -Name "CyberArk Privileged Session Manager" -ErrorAction SilentlyContinue
$SIAService = Get-Service -Name "CyberArk*SIA*" -ErrorAction SilentlyContinue

Write-Host "    PSM: $(if($PSMService){$PSMService.Status}else{'NOT FOUND'})" -ForegroundColor $(if($PSMService.Status -eq 'Running'){'Green'}else{'Red'})
Write-Host "    SIA: $(if($SIAService){$SIAService.Status}else{'NOT FOUND'})" -ForegroundColor $(if($SIAService.Status -eq 'Running'){'Green'}else{'Red'})
```

---

## 7. Quick Reference

### Common Commands

```powershell
# ═══════════════════════════════════════════════════════════════════════════════
# ACTIVE DIRECTORY
# ═══════════════════════════════════════════════════════════════════════════════

# List PAM OUs
Get-ADOrganizationalUnit -Filter * -SearchBase "OU=PAM,DC=indramind,DC=cyblab" |
    Select-Object Name, DistinguishedName

# List PAM Groups
Get-ADGroup -Filter * -SearchBase "OU=PAM,DC=indramind,DC=cyblab" |
    Select-Object Name, GroupScope

# List Service Accounts
Get-ADUser -Filter * -SearchBase "OU=ServiceAccounts,OU=PAM,DC=indramind,DC=cyblab" |
    Select-Object Name, Enabled

# Check Group Membership
Get-ADGroupMember -Identity "PAM-Users" | Select-Object Name

# ═══════════════════════════════════════════════════════════════════════════════
# CYBERARK SERVICES
# ═══════════════════════════════════════════════════════════════════════════════

# Check CyberArk Services
Get-Service -Name "CyberArk*"

# Start/Stop PSM
Start-Service -Name "CyberArk Privileged Session Manager"
Stop-Service -Name "CyberArk Privileged Session Manager"

# ═══════════════════════════════════════════════════════════════════════════════
# CONNECTIVITY
# ═══════════════════════════════════════════════════════════════════════════════

# Test Cloud Connectivity
Test-NetConnection -ComputerName "connector.privilegecloud.cyberark.cloud" -Port 443

# Test DC Connectivity
Test-NetConnection -ComputerName "DC01.indramind.cyblab" -Port 636

# Test PSM DNS
Resolve-DnsName -Name "psm.indramind.cyblab" -Type A

# ═══════════════════════════════════════════════════════════════════════════════
# DNS
# ═══════════════════════════════════════════════════════════════════════════════

# List DNS A Records
Get-DnsServerResourceRecord -ZoneName "indramind.cyblab" -RRType A |
    Where-Object { $_.HostName -like "psm*" -or $_.HostName -like "connector*" }
```

---

## Document Footer

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║  Document: LAB-SCRIPTS-001 | Version: 3.0 | Status: Production Ready            ║
║  Classification: Internal Use | Owner: Lab Team | Updated: January 2026         ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

**Related Documents:**
- [lab_pcloud_architecture.md](lab_pcloud_architecture.md) - Technical architecture
- [INSTALLATION.md](INSTALLATION.md) - Complete installation guide
- [README.md](README.md) - Repository overview
