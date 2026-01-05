# Lab Privilege Cloud - PowerShell Scripts

Scripts for CyberArk Privilege Cloud lab environment implementation.

> **Reference:** See [lab_pcloud_architecture.md](lab_pcloud_architecture.md) for full architecture documentation.

---

## 📚 External References

### PowerShell & Active Directory

| Cmdlet/Topic | Documentation |
|--------------|---------------|
| **New-ADUser** | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/new-aduser) |
| **New-ADGroup** | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/new-adgroup) |
| **New-ADOrganizationalUnit** | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/new-adorganizationalunit) |
| **Get-ADUser** | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-aduser) |
| **Add-ADGroupMember** | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/add-adgroupmember) |
| **Disable-ADAccount** | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/disable-adaccount) |
| **Move-ADObject** | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/move-adobject) |
| **Get-Service** | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-service) |
| **Test-NetConnection** | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) |
| **Install-WindowsFeature** | [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/servermanager/install-windowsfeature) |

### CyberArk Documentation

| Topic | Documentation |
|-------|---------------|
| **AD Connector Setup** | [CyberArk Identity - AD Integration](https://docs.cyberark.com/identity/latest/en/content/integrations/ad/ad-integration.htm) |
| **PSM Installation** | [CyberArk - Install Connector](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/install-connector.htm) |
| **PSM Requirements** | [CyberArk - Connector Requirements](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/connector-requirements.htm) |
| **Service Accounts** | [CyberArk - Service Account Best Practices](https://docs.cyberark.com/privilege-cloud/latest/en/content/pasimp/psm-service-account.htm) |

### Windows Server

| Topic | Documentation |
|-------|---------------|
| **RDS Session Host** | [Microsoft - Remote Desktop Services](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/welcome-to-rds) |
| **Windows Defender Exclusions** | [Microsoft - Configure Exclusions](https://learn.microsoft.com/en-us/microsoft-365/security/defender-endpoint/configure-exclusions-microsoft-defender-antivirus) |
| **DNS Server** | [Microsoft - DNS Overview](https://learn.microsoft.com/en-us/windows-server/networking/dns/dns-top) |

---

## Table of Contents

1. [AD Connector Service Account](#1-ad-connector-service-account)
2. [TIER Model Structure](#2-tier-model-structure)
3. [User Management](#3-user-management)
4. [PSM Server Preparation](#4-psm-server-preparation)
5. [Quick Reference Commands](#5-quick-reference-commands)

---

## 1. AD Connector Service Account

Create the service account for CyberArk Identity AD connector.

**Run on:** DC01 (Domain Controller)

```powershell
# Create the service account
$Password = ConvertTo-SecureString -String (New-Guid).Guid -AsPlainText -Force
New-ADUser -Name "svc-CyberArkIdentity" `
    -SamAccountName "svc-CyberArkIdentity" `
    -UserPrincipalName "svc-CyberArkIdentity@indramind.cyblab" `
    -Path "OU=ServiceAccounts,OU=PAM,DC=indramind,DC=cyblab" `
    -AccountPassword $Password `
    -Enabled $true `
    -PasswordNeverExpires $true `
    -CannotChangePassword $true `
    -Description "CyberArk Identity AD Connector - Read Only"

# Grant read permissions to the PAM OU
$ServiceAccount = Get-ADUser "svc-CyberArkIdentity"
$PAM_OU = "OU=PAM,DC=indramind,DC=cyblab"

# Add to Domain Users (minimum required)
Add-ADGroupMember -Identity "Domain Users" -Members $ServiceAccount

# Display the generated password (save securely)
Write-Host "Service Account Password: $((New-Guid).Guid)"
Write-Host "IMPORTANT: Save this password securely for CyberArk Identity configuration"
```

---

## 2. TIER Model Structure

### 2.1 Create-TierStructure.ps1

Complete script to create PAM OU structure, security groups, and service accounts.

**Run on:** DC01 (Domain Controller)
**Save as:** `Create-TierStructure.ps1`

```powershell
#Requires -Modules ActiveDirectory

<#
.SYNOPSIS
    Creates the complete TIER model OU structure and security groups for CyberArk PAM.
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
#>

param(
    [Parameter(Mandatory=$true)]
    [string]$DomainDN = "DC=indramind,DC=cyblab"
)

$ErrorActionPreference = "Stop"

Write-Host "========================================" -ForegroundColor Cyan
Write-Host "Creating PAM TIER Model Structure" -ForegroundColor Cyan
Write-Host "Domain: $DomainDN" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan

# Define OU structure
$OUs = @(
    @{ Name = "PAM"; Path = $DomainDN; Description = "CyberArk Privileged Access Management" }

    # Tier 0
    @{ Name = "Tier0"; Path = "OU=PAM,$DomainDN"; Description = "Tier 0 - Domain Controllers and Forest Admins" }
    @{ Name = "Groups"; Path = "OU=Tier0,OU=PAM,$DomainDN"; Description = "Tier 0 Security Groups" }
    @{ Name = "Accounts"; Path = "OU=Tier0,OU=PAM,$DomainDN"; Description = "Tier 0 Privileged Accounts" }

    # Tier 1
    @{ Name = "Tier1"; Path = "OU=PAM,$DomainDN"; Description = "Tier 1 - Enterprise Servers" }
    @{ Name = "Groups"; Path = "OU=Tier1,OU=PAM,$DomainDN"; Description = "Tier 1 Security Groups" }
    @{ Name = "Accounts"; Path = "OU=Tier1,OU=PAM,$DomainDN"; Description = "Tier 1 Privileged Accounts" }

    # Tier 2
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

# Create OUs
foreach ($OU in $OUs) {
    $FullPath = "OU=$($OU.Name),$($OU.Path)"
    try {
        if (-not (Get-ADOrganizationalUnit -Filter "DistinguishedName -eq '$FullPath'" -ErrorAction SilentlyContinue)) {
            New-ADOrganizationalUnit -Name $OU.Name -Path $OU.Path -Description $OU.Description -ProtectedFromAccidentalDeletion $true
            Write-Host "[CREATED] $FullPath" -ForegroundColor Green
        } else {
            Write-Host "[EXISTS]  $FullPath" -ForegroundColor Yellow
        }
    } catch {
        Write-Host "[ERROR]   $FullPath - $($_.Exception.Message)" -ForegroundColor Red
    }
}

# Define Security Groups
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

Write-Host "`n----------------------------------------" -ForegroundColor Cyan
Write-Host "Creating Security Groups" -ForegroundColor Cyan
Write-Host "----------------------------------------" -ForegroundColor Cyan

# Create Groups
foreach ($Group in $Groups) {
    try {
        if (-not (Get-ADGroup -Filter "Name -eq '$($Group.Name)'" -ErrorAction SilentlyContinue)) {
            New-ADGroup -Name $Group.Name `
                -Path $Group.Path `
                -GroupScope $Group.Scope `
                -GroupCategory Security `
                -Description $Group.Description
            Write-Host "[CREATED] $($Group.Name)" -ForegroundColor Green
        } else {
            Write-Host "[EXISTS]  $($Group.Name)" -ForegroundColor Yellow
        }
    } catch {
        Write-Host "[ERROR]   $($Group.Name) - $($_.Exception.Message)" -ForegroundColor Red
    }
}

# Define Service Accounts
$ServiceAccounts = @(
    @{ Name = "svc-CyberArkCPM"; Description = "CyberArk CPM Service Account" }
    @{ Name = "svc-CyberArkPSM"; Description = "CyberArk PSM Service Account" }
    @{ Name = "svc-CyberArkScan"; Description = "CyberArk Scanner Service Account" }
    @{ Name = "svc-CyberArkIdentity"; Description = "CyberArk Identity AD Connector" }
    @{ Name = "svc-CyberArkReconcile"; Description = "CyberArk Reconciliation Account" }
)

Write-Host "`n----------------------------------------" -ForegroundColor Cyan
Write-Host "Creating Service Accounts" -ForegroundColor Cyan
Write-Host "----------------------------------------" -ForegroundColor Cyan

foreach ($SvcAcct in $ServiceAccounts) {
    try {
        if (-not (Get-ADUser -Filter "SamAccountName -eq '$($SvcAcct.Name)'" -ErrorAction SilentlyContinue)) {
            $Password = ConvertTo-SecureString -String ([System.Guid]::NewGuid().ToString() + "!Aa1") -AsPlainText -Force
            New-ADUser -Name $SvcAcct.Name `
                -SamAccountName $SvcAcct.Name `
                -UserPrincipalName "$($SvcAcct.Name)@indramind.cyblab" `
                -Path "OU=ServiceAccounts,OU=PAM,$DomainDN" `
                -AccountPassword $Password `
                -Enabled $true `
                -PasswordNeverExpires $true `
                -CannotChangePassword $true `
                -Description $SvcAcct.Description
            Write-Host "[CREATED] $($SvcAcct.Name)" -ForegroundColor Green
        } else {
            Write-Host "[EXISTS]  $($SvcAcct.Name)" -ForegroundColor Yellow
        }
    } catch {
        Write-Host "[ERROR]   $($SvcAcct.Name) - $($_.Exception.Message)" -ForegroundColor Red
    }
}

Write-Host "`n========================================" -ForegroundColor Cyan
Write-Host "TIER Model Structure Creation Complete" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
```

### 2.2 Run and Verify

```powershell
# Run the script
.\Create-TierStructure.ps1 -DomainDN "DC=indramind,DC=cyblab"
```

```powershell
# Verify OUs
Get-ADOrganizationalUnit -Filter * -SearchBase "OU=PAM,DC=indramind,DC=cyblab" |
    Select-Object Name, DistinguishedName | Format-Table -AutoSize

# Verify Groups
Get-ADGroup -Filter * -SearchBase "OU=PAM,DC=indramind,DC=cyblab" |
    Select-Object Name, GroupScope | Format-Table -AutoSize

# Verify Service Accounts
Get-ADUser -Filter * -SearchBase "OU=ServiceAccounts,OU=PAM,DC=indramind,DC=cyblab" |
    Select-Object Name, Enabled | Format-Table -AutoSize
```

---

## 3. User Management

### 3.1 Bulk Update User Emails via API

```powershell
# Example: Update user email via CyberArk Identity API
$TenantURL = "https://[tenant].id.cyberark.cloud"
$Token = "your-api-token"

$Users = @(
    @{ username = "user1"; newEmail = "user1@newdomain.com" }
    @{ username = "user2"; newEmail = "user2@newdomain.com" }
)

foreach ($User in $Users) {
    $Body = @{
        email = $User.newEmail
    } | ConvertTo-Json

    Invoke-RestMethod -Uri "$TenantURL/api/users/$($User.username)" `
        -Method PATCH `
        -Headers @{ Authorization = "Bearer $Token" } `
        -Body $Body `
        -ContentType "application/json"

    Write-Host "Updated: $($User.username)"
}
```

### 3.2 Create Domain Accounts

```powershell
# Create domain user account
$Users = @(
    @{ FirstName = "John"; LastName = "Doe"; Email = "john.doe@newdomain.com" }
    @{ FirstName = "Jane"; LastName = "Smith"; Email = "jane.smith@newdomain.com" }
)

foreach ($User in $Users) {
    $Username = "$($User.FirstName.ToLower()).$($User.LastName.ToLower())"
    $Password = ConvertTo-SecureString -String "TempPassword123!" -AsPlainText -Force

    New-ADUser -Name "$($User.FirstName) $($User.LastName)" `
        -GivenName $User.FirstName `
        -Surname $User.LastName `
        -SamAccountName $Username `
        -UserPrincipalName "$Username@indramind.cyblab" `
        -EmailAddress $User.Email `
        -Path "OU=Active,OU=Users,OU=PAM,DC=indramind,DC=cyblab" `
        -AccountPassword $Password `
        -Enabled $true `
        -ChangePasswordAtLogon $true

    # Add to PAM-Users group
    Add-ADGroupMember -Identity "PAM-Users" -Members $Username

    Write-Host "[CREATED] $Username"
}
```

### 3.3 Disable Departed Users

```powershell
# Users to disable (departed team members)
$DepartedUsers = @("yago", "olmedo", "ana")

foreach ($Username in $DepartedUsers) {
    # In CyberArk Identity - disable via API or UI
    # In AD - disable and move to Disabled OU

    $User = Get-ADUser -Filter "SamAccountName -eq '$Username'" -ErrorAction SilentlyContinue
    if ($User) {
        # Disable account
        Disable-ADAccount -Identity $User

        # Move to Disabled OU
        Move-ADObject -Identity $User.DistinguishedName `
            -TargetPath "OU=Disabled,OU=Users,OU=PAM,DC=indramind,DC=cyblab"

        Write-Host "[DISABLED] $Username - Moved to Disabled OU"
    }
}
```

---

## 4. PSM Server Preparation

### 4.1 Prepare Connector Servers

**Run on:** Connector01 and Connector02

```powershell
# 1. Install Remote Desktop Session Host role
Install-WindowsFeature -Name RDS-RD-Server -IncludeManagementTools

# 2. Configure RDS licensing (if required)
# This may require RDS CALs depending on your licensing

# 3. Verify network connectivity to CyberArk Cloud
Test-NetConnection -ComputerName "connector.privilegecloud.cyberark.cloud" -Port 443

# 4. Verify domain membership
Get-WmiObject -Class Win32_ComputerSystem | Select-Object Domain, PartOfDomain

# 5. Create local admin for PSM (optional)
$Password = ConvertTo-SecureString "ComplexPassword123!" -AsPlainText -Force
New-LocalUser -Name "PSMAdmin" -Password $Password -PasswordNeverExpires
Add-LocalGroupMember -Group "Administrators" -Member "PSMAdmin"
```

### 4.2 Run Connector Installer

```powershell
# Run as Administrator
.\PrivilegeCloudConnector.exe
```

### 4.3 Verify Installation

```powershell
# Check PSM service
Get-Service -Name "CyberArk Privileged Session Manager"

# Check SIA service
Get-Service -Name "CyberArk SIA Connector"

# Both should show Status: Running
```

### 4.4 Test HA Failover

```powershell
# On Connector01 - Stop PSM service
Stop-Service -Name "CyberArk Privileged Session Manager"

# Attempt new connection
# Connection should route to Connector02

# Verify in Portal:
# Administration → Session Monitoring → Check PSM Server

# Restart PSM01
Start-Service -Name "CyberArk Privileged Session Manager"
```

---

## 5. Quick Reference Commands

### AD Verification

```powershell
# Verify AD Structure
Get-ADOrganizationalUnit -Filter * -SearchBase "OU=PAM,DC=indramind,DC=cyblab" |
    Select-Object Name, DistinguishedName

# List PAM Groups
Get-ADGroup -Filter * -SearchBase "OU=PAM,DC=indramind,DC=cyblab" |
    Select-Object Name, GroupScope

# List Service Accounts
Get-ADUser -Filter * -SearchBase "OU=ServiceAccounts,OU=PAM,DC=indramind,DC=cyblab" |
    Select-Object Name, Enabled

# Check PSM Service
Get-Service -Name "CyberArk*"

# Test Cloud Connectivity
Test-NetConnection -ComputerName "connector.privilegecloud.cyberark.cloud" -Port 443
```

---

*See [lab_pcloud_architecture.md](lab_pcloud_architecture.md) for complete implementation guide.*
